# Cluster Proxmox + Qdevice — Homelab

> **Statut :** ✅ En production  
> **Date :** Mars 2026  
> **OS Proxmox :** VE 9.1.x  
> **OS nasctl :** Debian 13 (trixie)

---

## 1. Vue d'ensemble

Ce document décrit la mise en place du cluster Proxmox à deux nœuds (pve1 + pve2) avec un qdevice
hébergé sur le container `nasctl` du NAS. Le trafic Corosync est isolé sur un réseau physique dédié
(VLAN 40, `10.0.40.0/24`) via le CRS310.

### 1.1 Pourquoi un qdevice

Un cluster à deux nœuds ne peut pas atteindre le quorum par lui-même en cas de panne d'un nœud —
les deux ont un vote chacun, et aucun ne peut obtenir la majorité seul (1/2). Le qdevice ajoute un
troisième votant externe (nasctl), ce qui permet à un nœud survivant d'obtenir 2/3 des votes et de
continuer à opérer normalement.

### 1.2 Infrastructure

| Équipement | Rôle | IP Management | IP Cluster (VLAN 40) |
|---|---|---|---|
| pve1 (MS-01) | Nœud Proxmox 1 | 192.168.1.20 | 10.0.40.20/24 |
| pve2 (MS-01) | Nœud Proxmox 2 | 192.168.1.21 | 10.0.40.21/24 |
| nasctl (LXC) | Qdevice (corosync-qnetd) | 192.168.1.11 | 10.0.40.11/24 |
| NAS (AS6806T) | Hôte nasctl | 192.168.1.10 | — |
| CRS310 | Switch accès cluster | 192.168.1.2 | — |

### 1.3 Topologie réseau VLAN 40

```
pve1 (nic1) ──── ether3 ──┐
                           ├── br-cluster (CRS310, isolé)
pve2 (nic1) ──── ether5 ──┤
                           │
NAS (LAN4)  ──── ether7 ──┘
                  │
           eth2-br (bridge NAS)
                  │
          nasctl (eth3) → 10.0.40.11
```

> Le VLAN 40 est local au CRS310 — il ne traverse pas le trunk vers le CRS317.  
> Pour l'instant, le bridge `br-cluster` n'a pas de VLAN filtering — c'est une isolation physique
> via bridge dédié. Le VLAN tagging sera ajouté lors de l'implémentation du VLAN 10 / OPNsense.

---

## 2. Câblage physique

Brancher les câbles suivants avant de commencer :

| Câble | De | Vers |
|---|---|---|
| RJ45 | pve1 nic1 | CRS310 ether3 |
| RJ45 | pve2 nic1 | CRS310 ether5 |
| RJ45 | NAS LAN4 (eth2) | CRS310 ether7 |

> pve1/pve2 : utiliser le port RJ45 libre (nic1 sur les MS-01).  
> NAS : LAN4 est le port 2.5G `eth2`, jusqu'ici non utilisé.

---

## 3. Configuration CRS310 — bridge br-cluster

Créer un bridge dédié sur le CRS310 pour isoler le trafic cluster des autres flux.

### 3.1 Se connecter au CRS310

```bash
ssh admin@192.168.1.2
```

### 3.2 Créer le bridge br-cluster

```routeros
/interface bridge add name=br-cluster mtu=1500 vlan-filtering=no protocol-mode=rstp
```

### 3.3 Ajouter les ports cluster dans br-cluster

```routeros
/interface bridge port add bridge=br-cluster interface=ether3 hw=yes
/interface bridge port add bridge=br-cluster interface=ether5 hw=yes
/interface bridge port add bridge=br-cluster interface=ether7 hw=yes
```

> Ces ports sont retirés du bridge de management flat. Vérifier qu'ils n'y étaient pas déjà :
> ```routeros
> /interface bridge port print where bridge=bridge1
> # Si ether3, ether5 ou ether7 apparaissent, les retirer :
> /interface bridge port remove [find interface=ether3]
> /interface bridge port remove [find interface=ether5]
> /interface bridge port remove [find interface=ether7]
> ```

### 3.4 Vérification

```routeros
/interface bridge port print where bridge=br-cluster
# Attendu : ether3, ether5, ether7 membres de br-cluster
```

### 3.5 Sauvegarder

```routeros
/system backup save name=crs310-config-br-cluster
```

---

## 4. Configuration pve1 — Interface VLAN 40

### 4.1 Via Proxmox GUI — pve1 → System → Network

Créer l'interface cluster sur nic1 :

- Double-cliquer sur **nic1** → MTU : `1500` → OK
- **Create → Linux Network Device** :

| Champ | Valeur |
|---|---|
| Name | `nic1` |
| IP address | `10.0.40.20/24` |
| MTU | `1500` |
| Autostart | ✅ |
| Comment | `CLUSTER` |

- Cliquer **Apply Configuration**

### 4.2 Vérification

```bash
ip addr show nic1
# Attendu : inet 10.0.40.20/24, state UP

ping -c 3 10.0.40.21   # pve2 (une fois configuré)
```

---

## 5. Configuration pve2 — Interface VLAN 40

### 5.1 Via Proxmox GUI — pve2 → System → Network

Même procédure que pve1 :

| Champ | Valeur |
|---|---|
| Name | `nic1` |
| IP address | `10.0.40.21/24` |
| MTU | `1500` |
| Autostart | ✅ |
| Comment | `CLUSTER` |

- Cliquer **Apply Configuration**

### 5.2 Vérification

```bash
ip addr show nic1
# Attendu : inet 10.0.40.21/24, state UP

ping -c 3 10.0.40.20   # pve1
```

---

## 6. Configuration NAS — Interface eth2 pour le cluster

Le NAS LAN4 (`eth2`) doit être ponté pour y connecter le container nasctl, en suivant le même
modèle que `eth1-br` pour le backup.

### 6.1 Configurer l'IP de LAN4 via ADM

Dans **ADM → Réglages → Réseau → LAN4** (`eth2`) :

- Adresse IP : `10.0.40.10`
- Masque : `255.255.255.0`
- Passerelle : vide (réseau isolé)
- MTU : 1500

> L'IP `10.0.40.10` est attribuée au NAS lui-même sur ce réseau — elle ne sera pas utilisée
> activement mais sert de référence et évite tout conflit avec le bridge.

### 6.2 Créer le script de bridge eth2-br

Sur le NAS (SSH ou terminal ADM) :

```bash
sudo tee /usr/local/etc/init.d/S46eth2-bridge << 'EOF'
#!/bin/sh
case $1 in
    start)
        ip link add eth2-br type bridge 2>/dev/null || true
        ip link set eth2 master eth2-br
        ip link set eth2-br up
        ip link set eth2 up
        ip addr del 10.0.40.10/24 dev eth2 2>/dev/null || true
        ip addr add 10.0.40.10/24 dev eth2-br 2>/dev/null || true
        ;;
    stop)
        ip link set eth2 nomaster 2>/dev/null || true
        ip link delete eth2-br 2>/dev/null || true
        ;;
    *)
        echo "usage: $0 {start|stop}"
        exit 1
        ;;
esac
exit 0
EOF

sudo chmod +x /usr/local/etc/init.d/S46eth2-bridge
sudo /usr/local/etc/init.d/S46eth2-bridge start
```

> Nommé `S46eth2-bridge` pour s'exécuter après `S45eth1-bridge` et avant `S50linux-center`.

### 6.3 Vérification

```bash
ip addr show eth2-br
# Attendu : inet 10.0.40.10/24, state UP
```

---

## 7. Configuration nasctl — Interface eth3 (VLAN 40)

### 7.1 Arrêter le container nasctl

Dans **ADM → Linux Center**, arrêter le container `nasctl`.

### 7.2 Ajouter l'interface eth3 dans la config LXC

Sur le NAS :

```bash
sudo tee -a /usr/local/AppCentral/linux-center/containers/debian13-server/net << 'EOF'
lxc.net.3.type=veth
lxc.net.3.flags=up
lxc.net.3.link=eth2-br
lxc.net.3.name=eth3
EOF
```

### 7.3 Redémarrer le container

Dans **ADM → Linux Center**, démarrer nasctl.

### 7.4 Configurer l'IP statique sur eth3

Dans nasctl, éditer `/etc/systemd/system/network-static.service` pour ajouter eth3 :

```ini
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c '/sbin/ip addr add 192.168.1.11/24 dev eth0 2>/dev/null || true'
ExecStart=/bin/sh -c '/sbin/ip route add default via 192.168.1.1 2>/dev/null || true'
ExecStart=/bin/sh -c '/sbin/ip link set eth2 mtu 9000 2>/dev/null || true'
ExecStart=/bin/sh -c '/sbin/ip addr add 10.0.50.11/24 dev eth2 2>/dev/null || true'
ExecStart=/bin/sh -c '/sbin/ip addr add 10.0.40.11/24 dev eth3 2>/dev/null || true'
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart network-static
```

### 7.5 Vérification

```bash
ip addr show eth3
# Attendu : inet 10.0.40.11/24

ping -c 3 10.0.40.20   # pve1
ping -c 3 10.0.40.21   # pve2
```

---

## 8. Installation de corosync-qnetd sur nasctl

```bash
sudo apt update
sudo apt install corosync-qnetd -y
```

Vérifier que le service tourne :

```bash
systemctl status corosync-qnetd
# Attendu : active (running)
```

> `corosync-qnetd` écoute sur le port `5403` par défaut.  
> Il n'y a pas de configuration manuelle nécessaire — Proxmox gère l'initialisation via `pvecm`.

---

## 9. Création du cluster Proxmox sur pve1

### 9.1 Créer le cluster

Sur **pve1** :

```bash
pvecm create homelab-cluster --link0 10.0.40.20
```

> `--link0` spécifie l'interface Corosync dédiée (VLAN 40).  
> Le nom du cluster `homelab-cluster` est permanent et ne peut pas être changé sans recréer le cluster.

### 9.2 Vérifier l'état du cluster

```bash
pvecm status
```

Résultat attendu :
```
Cluster information
-------------------
Name:             homelab-cluster
Config Version:   1
Transport:        knet
Secure auth:      on

Quorum information
------------------
Date:             ...
Quorum provider:  corosync_votequorum
Nodes:            1
Node state:       Established

Membership information
----------------------
    Nodeid      Votes Name
         1          1 pve1 (local)
```

---

## 10. Ajout de pve2 au cluster

### 10.1 Récupérer l'empreinte du cluster

Sur **pve1** :

```bash
pvecm status | grep -i fingerprint
# ou
cat /etc/corosync/authkey | sha256sum
```

### 10.2 Rejoindre le cluster depuis pve2

Sur **pve2** :

```bash
pvecm add 192.168.1.20 --link0 10.0.40.21
```

Il sera demandé le mot de passe root de pve1, puis la connexion est établie.

> `192.168.1.20` est l'IP management de pve1 — utilisée uniquement pour l'échange initial de clés.  
> Une fois la confiance établie, le trafic Corosync passe exclusivement par link0 (10.0.40.x).

### 10.3 Vérifier depuis pve1

```bash
pvecm status
```

Résultat attendu :
```
Membership information
----------------------
    Nodeid      Votes Name
         1          1 pve1 (local)
         2          1 pve2
```

```bash
pvecm nodes
# Doit afficher les deux nœuds
```

> À ce stade, le cluster est à deux nœuds avec 2 votes. Le quorum nécessite 2/2 votes — si un
> nœud tombe, le cluster est bloqué. C'est pourquoi le qdevice est indispensable.

---

## 11. Configuration du qdevice

### 11.1 Installer corosync-qdevice sur pve1 et pve2

Le package `corosync-qdevice` doit être installé sur **chaque nœud Proxmox** avant de lancer le setup :

```bash
# Sur pve1 ET pve2
apt install corosync-qdevice -y
```

### 11.2 Autoriser le login root par SSH sur nasctl (temporaire)

`pvecm qdevice setup` se connecte en SSH avec le compte root pour initialiser les certificats.
Il faut activer temporairement le login root par mot de passe sur nasctl :

```bash
# Sur nasctl
sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd
sudo passwd root   # définir un mot de passe root
```

### 11.3 Configurer le qdevice depuis pve1

Sur **pve1** :

```bash
pvecm qdevice setup 10.0.40.11
```

Proxmox va :
1. Copier sa clé SSH publique sur nasctl
2. Initialiser les certificats QNet
3. Configurer Corosync sur les deux nœuds
4. Démarrer et activer `corosync-qdevice` sur pve1 et pve2

### 11.4 Révoquer le login root par mot de passe sur nasctl

Une fois le setup terminé, remettre la config SSH à sa valeur par défaut :

```bash
# Sur nasctl
sudo sed -i 's/PermitRootLogin yes/#PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

> La clé SSH de pve1 est maintenant dans `/root/.ssh/authorized_keys` sur nasctl — le qdevice
> fonctionnera sans mot de passe lors des prochains redémarrages.

### 11.5 Vérifier l'état du quorum

```bash
pvecm status
```

Résultat attendu :
```
Quorum information
------------------
Quorum provider:  corosync_votequorum
Nodes:            2
Node state:       Established

Membership and quorum information
----------------------------------
    Nodeid      Votes    Qdevice Name
         1          1    A,V,NMW pve1 (local)
         2          1    A,V,NMW pve2
         0          1            Qdevice
```

> `Qdevice` doit apparaître avec `1` vote et les flags `A` (alive), `V` (vote), `NMW` (non-master wait).  
> Total des votes : 3. Quorum à 2 votes → un nœud seul peut continuer à opérer.

### 11.6 Vérification sur nasctl

```bash
systemctl status corosync-qnetd
# Doit afficher les connexions actives des deux nœuds
```

---

## 12. Vérifications finales

### 12.1 État général du cluster

```bash
# Depuis pve1 ou pve2
pvecm status
pvecm nodes

# Logs Corosync
journalctl -u corosync --no-pager | tail -20
```

### 12.2 Test de résilience quorum

Simuler la perte d'un nœud (optionnel) :

```bash
# Depuis pve2 — arrêter Corosync temporairement
systemctl stop corosync

# Depuis pve1 — vérifier que le cluster maintient le quorum
pvecm status
# pve1 doit obtenir 2/3 votes (lui-même + qdevice) et rester opérationnel
```

Redémarrer Corosync sur pve2 :
```bash
systemctl start corosync
```

### 12.3 Checklist finale

| Composant | Vérification | Attendu |
|---|---|---|
| CRS310 br-cluster | `/interface bridge port print where bridge=br-cluster` | ether3, ether5, ether7 |
| pve1 nic1 | `ip addr show nic1` | 10.0.40.20/24, UP |
| pve2 nic1 | `ip addr show nic1` | 10.0.40.21/24, UP |
| nasctl eth3 | `ip addr show eth3` | 10.0.40.11/24 |
| Ping cluster pve1→pve2 | `ping -c 3 10.0.40.21` depuis pve1 | 0% loss |
| Ping cluster pve1→nasctl | `ping -c 3 10.0.40.11` depuis pve1 | 0% loss |
| Cluster 2 nœuds | `pvecm nodes` | pve1 + pve2 |
| Qdevice actif | `pvecm status` | Qdevice avec 1 vote |
| corosync-qnetd | `systemctl status corosync-qnetd` sur nasctl | active (running) |

---

## 13. Dépannage

### pvecm add échoue avec "already member of a cluster"

pve2 a peut-être des résidus d'une tentative précédente :

```bash
# Sur pve2
systemctl stop pve-cluster corosync
pmxcfs -l   # monter en mode local
rm /etc/pve/corosync.conf
rm -f /etc/corosync/authkey
systemctl start pve-cluster
```

### Le qdevice ne rejoint pas — erreur de certificat

```bash
# Sur nasctl
corosync-qnetd-certutil -i
# Réinitialiser les certificats

# Puis relancer depuis pve1
pvecm qdevice remove
pvecm qdevice setup 10.0.40.11
```

### Corosync utilise la mauvaise interface

Vérifier la config dans `/etc/corosync/corosync.conf` sur pve1 :

```bash
grep -A5 "interface" /etc/corosync/corosync.conf
# linknumber: 0 doit pointer vers 10.0.40.x
```

### pve2 ne peut pas pinguer 10.0.40.20

Vérifier que nic1 est bien dans br-cluster sur le CRS310 et que le câble est branché sur ether5 :

```routeros
/interface bridge port print where bridge=br-cluster
/interface print where name=ether5
# Vérifier le flag R (running)
```

### Quorum perdu après reboot du NAS

`corosync-qnetd` démarre automatiquement avec systemd. Si le qdevice n'est pas disponible,
le cluster fonctionne encore avec 2/2 nœuds actifs. Si un nœud est aussi down, attendre que
nasctl redémarre ou démarrer manuellement :

```bash
# Sur nasctl
systemctl start corosync-qnetd
```

---

## 14. Résumé des IPs VLAN 40

| Équipement | Interface | IP | MTU |
|---|---|---|---|
| pve1 | nic1 | `10.0.40.20/24` | 1500 |
| pve2 | nic1 | `10.0.40.21/24` | 1500 |
| nasctl (eth3) | veth → eth2-br | `10.0.40.11/24` | 1500 |
| NAS (eth2-br) | bridge sur LAN4 | `10.0.40.10/24` | 1500 |
