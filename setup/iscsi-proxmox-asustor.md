# iSCSI Setup — Asustor AS6806T + Proxmox VE

## Vue d'ensemble

Ce document décrit la procédure complète pour configurer un stockage iSCSI partagé entre un NAS Asustor AS6806T (ADM 5.1.2) et un ou plusieurs nœuds Proxmox VE, en isolant le trafic storage sur un réseau dédié 10GbE (`10.0.20.x`).

### Infrastructure

| Équipement | IP Management | IP Storage (10G) |
|---|---|---|
| NAS Asustor AS6806T | 192.168.1.10 | 10.0.20.10 |
| Proxmox pve1 | 192.168.1.20 | 10.0.20.20 |
| Proxmox pve2 | 192.168.1.21 | 10.0.20.21 |

### Pourquoi cette approche

L'Asustor ADM ne permet pas de restreindre les portals iSCSI à une seule interface réseau — le NAS annonce le target sur **toutes** ses interfaces. La solution est de gérer la connexion iSCSI manuellement côté Proxmox pour forcer l'utilisation exclusive du réseau storage `10.0.20.x`, puis de créer le volume group LVM manuellement avant de l'enregistrer dans Proxmox.

> ⚠️ Ne jamais ajouter un storage de type **iSCSI** dans Proxmox GUI — cela provoquerait une re-découverte automatique de tous les portals du NAS, y compris les interfaces management.

---

## Étape 1 — Création du LUN iSCSI sur le NAS (ADM)

1. Ouvrir **Gestionnaire de Stockage** → **iSCSI LUN**
2. Cliquer **Créer**
3. Remplir les paramètres du LUN :
   - **Nom LUN** : `VMStore`
   - **Emplacement** : `Volume 1` (volume SSD RAID10)
   - **Thin provisioning** : `Oui`
   - **Soutien cliché (snapshot)** : `Oui`
   - **Taille** : `2048 Go`
4. Cliquer **Suivant**

### Paramètres du Target

- **Nom target** : `vmstore`
- **IQN généré** : `iqn.2011-08.com.asustor:as6806t-435b57.vmstore`
- **CRC / Checksum** : décoché (inutile sur réseau local dédié)
- **Authentification CHAP** : désactivée (à activer ultérieurement si nécessaire)

### Mapping LUN

- Sélectionner **Mapper iSCSI LUN existant**
- Cocher le LUN `VMStore` (2.00 To)

### Résultat attendu dans ADM

Vérifier dans la vue **iSCSI** que :
- Le target `vmstore` est en état **Prêt**
- Le LUN `VMStore` est **En ligne**

> **Note** : Le port iSCSI par défaut est `3260`. Ne pas activer iSNS.

---

## Étape 2 — Configuration de l'initiateur iSCSI sur Proxmox

> ⚠️ Cette étape est à répéter sur **chaque nœud Proxmox** (`pve1`, `pve2`, etc.) en adaptant l'IP storage du nœud.

### 2.1 Vérifier l'IQN de l'initiateur

```bash
cat /etc/iscsi/initiatorname.iscsi
```

Noter l'IQN — il sera visible dans l'interface ADM lors de la connexion.

### 2.2 Découverte du target

```bash
iscsiadm -m discovery -t sendtargets -p 10.0.20.10:3260
```

Le NAS retourne le target sur **toutes ses interfaces** (comportement normal d'ADM) :

```
10.0.20.10:3260,1 iqn.2011-08.com.asustor:as6806t-435b57.vmstore
192.168.1.10:3260,1 iqn.2011-08.com.asustor:as6806t-435b57.vmstore
10.172.5.1:3260,1 iqn.2011-08.com.asustor:as6806t-435b57.vmstore
...
```

### 2.3 Supprimer les portals indésirables

Ne garder **que** le portal storage `10.0.20.10` :

```bash
iscsiadm -m node -T iqn.2011-08.com.asustor:as6806t-435b57.vmstore -p 192.168.1.10:3260 --op delete
iscsiadm -m node -T iqn.2011-08.com.asustor:as6806t-435b57.vmstore -p 10.172.5.1:3260 --op delete
iscsiadm -m node -T iqn.2011-08.com.asustor:as6806t-435b57.vmstore -p 100.94.231.61:3260 --op delete
```

Vérifier qu'il ne reste que le bon portal :

```bash
iscsiadm -m node
# Doit afficher uniquement :
# 10.0.20.10:3260,1 iqn.2011-08.com.asustor:as6806t-435b57.vmstore
```

### 2.4 Connexion et persistance au boot

```bash
# Se connecter
iscsiadm -m node -T iqn.2011-08.com.asustor:as6806t-435b57.vmstore -p 10.0.20.10:3260 --login

# Configurer le démarrage automatique au boot
iscsiadm -m node -T iqn.2011-08.com.asustor:as6806t-435b57.vmstore -p 10.0.20.10:3260 --op update -n node.startup -v automatic
```

### 2.5 Vérifier la présence du disque

```bash
lsblk | grep sda
# Le LUN doit apparaître comme /dev/sda (2To)
```

---

## Étape 3 — Création du volume group LVM

> ⚠️ Cette étape est à effectuer **uniquement sur pve1** (premier nœud). Le VG sera partagé entre les nœuds via iSCSI.

```bash
# Initialiser le disque iSCSI comme volume physique LVM
pvcreate /dev/sda

# Créer le volume group
vgcreate vg-vmstore /dev/sda

# Vérifier
vgs
```

Résultat attendu :
```
VG         #PV #LV #SN Attr   VSize  VFree
vg-vmstore   1   0   0 wz--n- <2.00t <2.00t
```

---

## Étape 4 — Optimisation du scheduler I/O

Pour des performances optimales sur le LUN iSCSI, configurer le scheduler I/O sur `none` côté Proxmox :

```bash
# Appliquer immédiatement
echo none | tee /sys/block/sda/queue/scheduler

# Rendre persistant au boot
echo 'ACTION=="add|change", KERNEL=="sda", ATTR{queue/scheduler}="none"' | tee /etc/udev/rules.d/60-iscsi-scheduler.rules

# Recharger udev
udevadm control --reload-rules
udevadm trigger

# Vérifier
cat /sys/block/sda/queue/scheduler
# Doit afficher : [none] mq-deadline
```

---

## Étape 5 — Ajout du storage dans Proxmox

> ⚠️ Utiliser le type **LVM** et non iSCSI dans Proxmox GUI.

### Dans Proxmox GUI : Datacenter → Storage → Add → LVM

| Paramètre | Valeur |
|---|---|
| ID | `VMStore` |
| Base storage | `Existing volume groups` |
| Volume group | `vg-vmstore` |
| Content | `Disk image, Container` |
| Shared | ✅ Coché |

---

## Étape 6 — Création d'une VM sur le storage iSCSI

Lors de la création d'une VM, pour l'onglet **Disks** :

| Paramètre | Valeur recommandée |
|---|---|
| Bus/Device | `SCSI` |
| SCSI Controller | `VirtIO SCSI single` |
| Storage | `VMStore` |
| Cache | `Default (No cache)` |
| Discard | ✅ Coché (TRIM pour thin provisioning) |
| SSD emulation | ✅ Coché |
| IO thread | ✅ Coché |
| Async IO | `Default (io_uring)` |

---

## Étape 7 — Ajout d'un deuxième nœud (pve2)

Sur `pve2`, répéter les étapes **2.1 à 2.4** en adaptant l'IP du nœud (`10.0.20.21`), ainsi que l'**étape 4** pour le scheduler.

Le volume group `vg-vmstore` existe déjà sur le LUN — **ne pas relancer** `pvcreate` ni `vgcreate`.

Activer le VG existant :

```bash
vgscan
vgchange -ay vg-vmstore
```

Puis dans Proxmox GUI, ajouter le même storage LVM avec les mêmes paramètres qu'en étape 5.

---

## Dépannage

### Les portals indésirables reviennent après suppression
**Cause** : Un storage de type iSCSI dans Proxmox GUI re-découvre automatiquement tous les portals.  
**Solution** : S'assurer qu'aucun storage de type **iSCSI** n'est présent dans Datacenter → Storage. Utiliser uniquement le type **LVM**.

### Erreur `Can't initialize physical volume without -ff`
Des métadonnées LVM résiduelles sont présentes sur le disque.  
**Solution** :
```bash
pvcreate -ff /dev/sda
```

### Vérifier la session iSCSI active
```bash
iscsiadm -m session
# Doit afficher uniquement la session via 10.0.20.10
```

### Vérifier le scheduler I/O
```bash
cat /sys/block/sda/queue/scheduler
# Doit afficher : [none] mq-deadline
```
