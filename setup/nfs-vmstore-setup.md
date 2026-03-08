# VMStore NFS — Stockage Partagé Proxmox

> **Statut :** ✅ En production  
> **Date :** Mars 2026  
> **OS Proxmox :** VE 9.1.x  
> **NAS :** Asustor AS6806T (ADM)

---

## 1. Vue d'ensemble

`VMStore` est le storage partagé entre pve1 et pve2, exposé via NFS depuis le NAS. Il permet la
live migration de VMs entre les deux nœuds et supporte les snapshots (format qcow2).

### 1.1 Pourquoi NFS plutôt qu'iSCSI/LVM

LVM-Thin partagé sur iSCSI nécessite `lvmlockd` + `sanlock` pour coordonner les accès concurrents
entre nœuds. Cette configuration est complexe et non officiellement supportée par Proxmox sans
Ceph. NFS est le storage partagé recommandé pour les clusters Proxmox à 2 nœuds.

| Critère | iSCSI + LVM-Thin | NFS |
|---|---|---|
| Live migration | ❌ (sans lvmlockd) | ✅ natif |
| Snapshots live | ❌ | ✅ (qcow2) |
| Configuration | Complexe | Simple |
| Support Proxmox | Non officiel | Officiel |
| Performance 10GbE | ~~Maximale~~ | Très bonne (~15% overhead) |

### 1.2 Infrastructure

| Composant | Détail |
|---|---|
| Serveur NFS | NAS Asustor AS6806T |
| IP serveur | `10.0.20.10` (réseau VMStore isolé) |
| Export | `/volume1/VMStore` |
| Réseau | `10.0.20.0/24`, via CRS317 br-vmstore (flat, MTU 9000) |
| Montage pve1 | `10.0.20.10:/volume1/VMStore` via nic3 (`10.0.20.20`) |
| Montage pve2 | `10.0.20.10:/volume1/VMStore` via nic3 (`10.0.20.21`) |
| Format disques VM | qcow2 |
| Storage Proxmox ID | `VMStore` |

---

## 2. Configuration NAS (ADM)

### 2.1 Créer le dossier partagé

Dans **ADM → Contrôle d'accès → Dossiers Partagés** :

- **Ajouter** → Nom : `VMStore`
- Volume : `Volume 1 (Btrfs)`
- Droits d'accès : `Lecture & Écriture pour tous les utilisateurs`
- Cliquer **Suivant** → **Terminer**

### 2.2 Activer le service NFS

Dans **ADM → Services → NFS** : activer le service NFS si pas déjà fait.

### 2.3 Configurer les droits NFS du share

Dans **ADM → Contrôle d'accès → Dossiers Partagés** :

- Sélectionner `VMStore` → **Droits d'Accès** → onglet **Privilèges NFS**
- Cliquer **Ajouter** :

| Paramètre | Valeur |
|---|---|
| Adresse client | `10.0.20.0/24` |
| Privilèges | Lecture & Écriture |
| Mappage root | `root (0)` |
| Asynchrone | **Non** (sync) |

- Cliquer **OK** → **OK**

> Le chemin d'export sera `/volume1/VMStore`.

---

## 3. Configuration Proxmox

### 3.1 Ajouter le storage NFS (Datacenter)

Dans **Proxmox GUI → Datacenter → Storage → Add → NFS** :

| Champ | Valeur |
|---|---|
| ID | `VMStore` |
| Server | `10.0.20.10` |
| Export | `/volume1/VMStore` |
| Content | Disk image, Container, ISO Image, Container template, Snippets |
| Nodes | All |
| Enable | ✅ |

Cliquer **Add**.

> Le storage apparaît immédiatement sur pve1 et pve2 — pas besoin de configuration séparée
> par nœud car c'est un storage Datacenter-level.

### 3.2 Vérification

```bash
# Sur pve1 et pve2
df -h | grep VMStore
mount | grep VMStore
```

Résultat attendu :
```
10.0.50.10:/volume1/VMStore  7.9T  xxx  xxx  x%  /mnt/pve/VMStore
```

---

## 4. Live Migration

La live migration fonctionne nativement sans configuration supplémentaire. Les disques VM sur
VMStore sont accessibles depuis les deux nœuds simultanément via NFS.

### 4.1 Tester la live migration

1. Créer une VM avec le disque sur `VMStore`
2. Démarrer la VM
3. VM → **Migrate** → sélectionner le nœud cible → **Migrate**

> Proxmox affiche un warning "Migration with local disk might take long" pour les disques sur NFS
> partagé — c'est informatif, pas une erreur. La migration copie l'état mémoire en fond pendant
> que la VM tourne, puis bascule en quelques secondes.

### 4.2 Prérequis pour la live migration

- VM doit utiliser un storage partagé (VMStore) pour ses disques
- Aucun disque sur LOCAL-STORAGE ou local-zfs (non partagés)
- Réseau cluster VLAN 40 opérationnel (Corosync)

---

## 5. Snapshots

Les VMs sur VMStore (format qcow2) supportent les snapshots live.

### 5.1 Créer un snapshot

VM → **Snapshots** → **Take Snapshot** :
- Inclure la RAM : optionnel (plus long mais restauration complète)
- Description : note sur l'état de la VM

### 5.2 Restaurer un snapshot

VM → **Snapshots** → sélectionner le snapshot → **Rollback**

> La VM doit être éteinte pour un rollback complet. Le rollback avec RAM incluse peut se faire
> à chaud sur certaines configurations.

---

## 6. Notes et Limites

- **Format qcow2** : légèrement moins performant que raw mais nécessaire pour les snapshots.
  Pour les VMs sans besoin de snapshot, format raw possible (`raw` dans les options de disque).
- **NFS synchrone** : le mode sync est plus lent qu'async mais évite la perte de données en cas
  de crash NAS. Acceptable pour un homelab.
- **Réseau dédié** : le trafic VMStore transite par `10.0.20.0/24` via br-vmstore (CRS317), complètement séparé du trafic PBS (VLAN 50). pve1 utilise nic3 (`10.0.20.20`), pve2 utilise nic3 (`10.0.20.21`), NAS sur LAN2/eth0 (`10.0.20.10`).
- **Pas de VLAN 30 dédié** : la migration utilise actuellement le réseau VMStore (`10.0.20.0/24`). Un VLAN 30 dédié à la migration sera configuré lors de la prochaine phase réseau.
