# PBS Backup Setup — Homelab

## Vue d'ensemble

Ce document décrit la mise en place complète de l'infrastructure de backup basée sur **Proxmox Backup Server (PBS)** dans un container LXC Debian sur le NAS Asustor, avec un réseau dédié VLAN 50.

### Infrastructure concernée

| Appareil | Rôle | IP Management | IP Backup (VLAN 50) |
|---|---|---|---|
| MikroTik CRS317 | Switch 10G | 192.168.1.3 | — |
| Asustor AS6806T (NAS) | Hôte PBS + stockage | 192.168.1.10 | 10.0.50.10 |
| `nasctl` (LXC sur NAS) | PBS Server + futur qdevice | 192.168.1.11 | 10.0.50.11 |
| pve1 (MS-01) | Nœud Proxmox 1 | 192.168.1.20 | 10.0.50.20 |
| pve2 (MS-01) | Nœud Proxmox 2 | 192.168.1.21 | 10.0.50.21 |

### Topologie réseau VLAN 50

```
pve1 (nic2) ──── sfp14 ──┐
                          ├── bridge1 (VLAN 50) ── CRS317
NAS (eth1)  ──── sfp12 ──┘
                          │
                    eth1-br (bridge NAS)
                          │
                   nasctl (eth2) → 10.0.50.11
```

---

## 1. Configuration MikroTik CRS317 — VLAN 50

Le port `sfp12` est connecté au NAS (LAN1, eth1) en mode **access VLAN 50** (untagged).
Le port `sfp14` est connecté à pve1 (nic2) en mode **trunk** (tagged VLAN 50).

### 1.1 Configurer les ports

```routeros
# sfp12 — access VLAN 50 (NAS backup, untagged)
/interface bridge port add bridge=bridge1 interface=sfp-sfpplus12 \
  frame-types=admit-only-untagged-and-priority-tagged pvid=50 hw=yes

# sfp14 — trunk VLAN 50 (pve1 nic2, tagged)
/interface bridge port add bridge=bridge1 interface=sfp-sfpplus14 \
  frame-types=admit-only-vlan-tagged pvid=1 hw=yes
```

### 1.2 Ajouter les VLANs au bridge

```routeros
# VLAN 50 — backup
/interface bridge vlan add bridge=bridge1 vlan-ids=50 \
  tagged=sfp-sfpplus1,sfp-sfpplus14,sfp-sfpplus16 \
  untagged=sfp-sfpplus12
```

> sfp16 (pve2 backup) est inclus dans les tagged dès maintenant — il sera inactif jusqu'au branchement de pve2.

### 1.3 Vérification

```routeros
/interface bridge vlan print
/interface bridge port print where bridge=bridge1
```

Résultat attendu :
- `sfp12` : PVID=50, `admit-only-untagged-and-priority-tagged`
- `sfp14` : PVID=1, `admit-only-vlan-tagged`
- VLAN 50 : tagged=sfp1,sfp14,sfp16 / untagged=sfp12

---

## 2. Configuration réseau NAS — Interface backup

L'interface `eth1` (LAN1 du NAS) est dédiée au backup. Elle reçoit l'IP `10.0.50.10` via un bridge interne nécessaire pour le container LXC.

### 2.1 Configurer l'IP via ADM

Dans **ADM → Réglages → Réseau → LAN1** :

- Adresse IP : `10.0.50.10`
- Masque : `255.255.255.0`
- Passerelle : vide (réseau isolé)
- MTU : 1500
- VLAN Tag : **désactivé** (le switch gère le tag, port access)

### 2.2 Créer le bridge eth1-br (persistant au reboot)

Linux Center nécessite un bridge sur `eth1` pour y attacher le container. Ce bridge est créé via un script de démarrage Asustor.

Créer le fichier `/usr/local/etc/init.d/S45eth1-bridge` :

```bash
sudo tee /usr/local/etc/init.d/S45eth1-bridge << 'EOF'
#!/bin/sh
case $1 in
    start)
        ip link add eth1-br type bridge 2>/dev/null || true
        ip link set eth1 master eth1-br
        ip link set eth1-br up
        ip link set eth1 up
        ip addr del 10.0.50.10/24 dev eth1 2>/dev/null || true
        ip addr add 10.0.50.10/24 dev eth1-br 2>/dev/null || true
        ;;
    stop)
        ip link set eth1 nomaster 2>/dev/null || true
        ip link delete eth1-br 2>/dev/null || true
        ;;
    *)
        echo "usage: $0 {start|stop}"
        exit 1
        ;;
esac
exit 0
EOF

sudo chmod +x /usr/local/etc/init.d/S45eth1-bridge
sudo /usr/local/etc/init.d/S45eth1-bridge start
```

> **Note** : Le script est nommé `S45eth1-bridge` pour s'exécuter **avant** `S50linux-center` (qui démarre les containers LXC). Cela garantit que `eth1-br` existe avant que le container essaie d'y connecter son interface.

### 2.3 Corriger la route parasite au boot

ADM assigne l'IP `10.0.50.10` directement sur `eth1` au démarrage, créant automatiquement une route vers `eth1`. Après l'exécution du script S45, l'IP est déplacée sur `eth1-br` mais la route vers `eth1` persiste. ADM recréant cette route après le script, un cron corrige le problème après le démarrage complet :

```bash
sudo crontab -e
```

Ajouter :
```
@reboot sleep 30 && ip route del 10.0.50.0/24 dev eth1 2>/dev/null
```

> Le `sleep 30` laisse le temps à ADM de finir son initialisation avant de nettoyer la route parasite.

### 2.4 Vérification

```bash
sudo ip addr show eth1-br
# Attendu : inet 10.0.50.10/24

ip route show | grep 10.0.50
# Attendu : une seule ligne via eth1-br

sudo ping -c 3 10.0.50.20   # ping pve1
```

---

## 3. Configuration Proxmox pve1 — Interface VLAN 50

Le nic2 de pve1 est branché sur `sfp14` du CRS317 (trunk VLAN 50). Il faut créer une interface VLAN 50 dessus.

### 3.1 Ajouter la configuration dans /etc/network/interfaces

```bash
nano /etc/network/interfaces
```

Ajouter :

```
auto nic2
iface nic2 inet manual

auto nic2.50
iface nic2.50 inet static
        address 10.0.50.20/24
        vlan-raw-device nic2
#BACKUP
```

### 3.2 Appliquer

```bash
ifup nic2
ifup nic2.50
```

### 3.3 Vérification

```bash
ip addr show nic2.50
# Attendu : inet 10.0.50.20/24

ping -c 3 10.0.50.10   # ping NAS
```

---

## 4. Configuration Proxmox pve2 — Interface VLAN 50 (à faire)

Même procédure que pve1. pve2 utilise son nic2 branché sur `sfp16` du CRS317 (trunk VLAN 50).

### 4.1 Ajouter dans /etc/network/interfaces sur pve2

```
auto nic2
iface nic2 inet manual

auto nic2.50
iface nic2.50 inet static
        address 10.0.50.21/24
        vlan-raw-device nic2
#BACKUP
```

### 4.2 Appliquer

```bash
ifup nic2
ifup nic2.50
ping -c 3 10.0.50.10   # ping NAS
ping -c 3 10.0.50.11   # ping PBS container
```

---

## 5. Création du container LXC Debian — Linux Center

### 5.1 Créer le container

Dans **ADM → Linux Center** :

- Image : **Debian 13 Server**
- Interface réseau : **LAN3** (port 2.5G management, `192.168.1.x`)
- Monter dossier partagé : ✅

Démarrer le container. Il obtient une IP DHCP sur le réseau management (`192.168.1.x`).

### 5.2 Renommer le container

`hostnamectl` ne fonctionne pas dans un container LXC. Utiliser à la place :

```bash
sudo sh -c 'echo nasctl > /etc/hostname'
sudo sed -i 's/LXCDEBIAN13S/nasctl/g' /etc/hosts
sudo hostname nasctl
```

### 5.3 Attacher l'interface backup au container

Sur le **NAS**, ajouter l'interface `eth2` dans la config LXC du container :

```bash
sudo tee -a /usr/local/AppCentral/linux-center/containers/debian13-server/net << 'EOF'
lxc.net.2.type=veth
lxc.net.2.flags=up
lxc.net.2.link=eth1-br
lxc.net.2.name=eth2
EOF
```

Redémarrer le container depuis Linux Center pour appliquer.

### 5.4 Mise à jour système

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 6. Configuration des utilisateurs dans le container

Les UIDs/GIDs sont synchronisés avec le NAS pour la cohérence des permissions sur les fichiers partagés.

| Utilisateur | UID | GID | Rôle |
|---|---|---|---|
| `luca` | 1000 | 1000 | Admin |
| `svcpbs` | 2001 | 2001 | Service PBS |
| `svcrsync` | 2002 | 2002 | Service rSync (futur) |

### 6.1 Créer les groupes

```bash
sudo -i
groupadd -g 1000 luca
groupadd -g 2001 svcpbs
groupadd -g 2002 svcrsync
```

### 6.2 Corriger le groupe admin par défaut

Le container Linux Center crée un user `admin` avec UID/GID 1000. Il faut libérer cet UID.

```bash
groupmod -g 1001 admin
sed -i 's/^admin:x:1000:/admin:x:1001:/' /etc/passwd
```

### 6.3 Créer les utilisateurs

```bash
# luca — admin principal
useradd -u 1000 -g 1000 -m -s /bin/bash luca
passwd luca
usermod -aG sudo luca

# svcpbs — service PBS
useradd -u 2001 -g 2001 -m -s /bin/bash -c "Service account for PBS" svcpbs

# svcrsync — service rSync (futur)
useradd -u 2002 -g 2002 -m -s /bin/bash -c "Service account for rSync" svcrsync
```

### 6.4 Supprimer le compte admin par défaut

Ouvrir une **deuxième session SSH** avec `luca` et tester `sudo -i` avant de continuer.

```bash
sudo userdel -r admin
```

---

## 7. Configuration réseau du container

Le service `networking` standard ne fonctionne pas dans ce container LXC (problème `udevadm`). Un service systemd dédié gère les IPs statiques.

### 7.1 Corriger le lien udevadm

```bash
sudo ln -s /usr/bin/udevadm /bin/udevadm
```

### 7.2 Créer le service réseau statique

```bash
sudo tee /etc/systemd/system/network-static.service << 'EOF'
[Unit]
Description=Static network configuration
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c '/sbin/ip addr add 192.168.1.11/24 dev eth0 2>/dev/null || true'
ExecStart=/bin/sh -c '/sbin/ip route add default via 192.168.1.1 2>/dev/null || true'
ExecStart=/bin/sh -c '/sbin/ip addr add 10.0.50.11/24 dev eth2 2>/dev/null || true'

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable network-static
sudo systemctl start network-static
```

### 7.3 Vérification

```bash
ip addr show
# Attendu :
# eth0 → 192.168.1.11/24 (management)
# eth1 → 10.172.5.2/24  (NAT interne Asustor, ne pas modifier)
# eth2 → 10.0.50.11/24  (backup VLAN 50)

ping 10.0.50.10   # NAS
ping 10.0.50.20   # pve1
```

### 7.4 Désactiver le repo enterprise et configurer no-subscription

```bash
cat > /etc/apt/sources.list.d/pbs-enterprise.sources << EOF
Types: deb
URIs: https://enterprise.proxmox.com/debian/pbs
Suites: trixie
Components: pbs-enterprise
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
Enabled: false
EOF

cat > /etc/apt/sources.list.d/proxmox.sources << EOF
Types: deb
URIs: http://download.proxmox.com/debian/pbs
Suites: trixie
Components: pbs-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

apt update
```

---

## 8. Installation de Proxmox Backup Server

```bash
sudo apt install proxmox-backup-server -y
```

> **Note** : Lors de l'installation, Postfix demande une configuration mail. Choisir **"No configuration"** pour un homelab.

### 8.1 Supprimer le popup subscription

```bash
sed -i "s/res.data.status.toLowerCase() !== 'active'/false/" \
  /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js

systemctl restart proxmox-backup-proxy
```

### 8.2 Configurer le mot de passe root

```bash
sudo passwd root
```

### 8.3 Vérifier que PBS tourne

```bash
systemctl status proxmox-backup proxmox-backup-proxy
```

Les deux services doivent être `active (running)`.

---

## 9. Création du dossier de backup sur le NAS

```bash
sudo mkdir -p /.share/pbs-backups
sudo chown -R backup:backup /.share/pbs-backups
```

> **Note** : PBS vérifie en dur que le répertoire de config `/etc/proxmox-backup` appartient à l'UID 34 (user `backup`). Il n'est pas possible de faire tourner PBS sous un autre utilisateur.

---

## 10. Création du Datastore PBS

Accéder à l'interface web PBS : `https://192.168.1.11:8007`

Login : `root` / mot de passe défini à l'étape 8.2.

**Administration → Datastore → Add Datastore** :

| Champ | Valeur |
|---|---|
| Name | `pbs-backups` |
| Backing Path | `/.share/pbs-backups` |
| GC Schedule | daily |
| Prune Schedule | daily |

Politique de rétention recommandée (**Prune Options**) :

| Paramètre | Valeur |
|---|---|
| Keep Last | 3 |
| Keep Daily | 7 |
| Keep Weekly | 4 |
| Keep Monthly | 3 |

---

## 11. Ajout de PBS dans Proxmox

### 11.1 Récupérer le fingerprint SSL

Depuis pve1 :

```bash
openssl s_client -connect 10.0.50.11:8007 </dev/null 2>/dev/null \
  | openssl x509 -fingerprint -sha256 -noout
```

### 11.2 Ajouter le storage dans Proxmox

**Datacenter → Storage → Add → Proxmox Backup Server** :

| Champ | Valeur |
|---|---|
| ID | `PBS` |
| Server | `10.0.50.11` |
| Username | `root@pam` |
| Password | mot de passe root du container |
| Datastore | `pbs-backups` |
| Fingerprint | (valeur récupérée ci-dessus) |

Répéter pour **pve2** une fois configuré.

---

## 12. Configuration de pve2 (à faire)

Une fois pve2 disponible, les étapes suivantes sont à réaliser :

1. **CRS317** : sfp16 est déjà configuré en trunk VLAN 50 — vérifier que le lien est actif après branchement
2. **pve2** : Configurer `nic2.50` avec IP `10.0.50.21` (cf. section 4)
3. **Proxmox** : Ajouter PBS comme storage sur pve2 (cf. section 11.2)

---

## Résumé des IPs VLAN 50

| Appareil | Interface | IP |
|---|---|---|
| NAS (eth1-br) | bridge sur eth1 | `10.0.50.10/24` |
| `nasctl` (eth2) | veth → eth1-br | `10.0.50.11/24` |
| pve1 (nic2.50) | VLAN 50 sur nic2 | `10.0.50.20/24` |
| pve2 (nic2.50) | VLAN 50 sur nic2 | `10.0.50.21/24` |
