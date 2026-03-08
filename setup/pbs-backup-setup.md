# PBS Backup Setup — Homelab

## Vue d'ensemble

Ce document décrit la mise en place complète de l'infrastructure de backup basée sur **Proxmox Backup Server (PBS)** dans un container LXC Debian sur le NAS Asustor, avec un réseau dédié VLAN 50 à MTU 9000.

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
                          ├── br-backup (VLAN 50, MTU 9000) ── CRS317
NAS (eth1)  ──── sfp12 ──┘
                          │
                    eth1-br (bridge NAS, MTU 9000)
                          │
                   nasctl (eth2, MTU 9000) → 10.0.50.11
```

---

## 1. Configuration MikroTik CRS317 — bridge br-backup

Le réseau backup utilise un bridge dédié `br-backup` à MTU 9000, sur le modèle du bridge iSCSI
`br-vmstore`. Le VLAN filtering est activé pour permettre d'y ajouter le futur VLAN 30
(live migration) sur le même lien physique.

Le port `sfp12` est connecté au NAS (LAN1, eth1) en mode **access VLAN 50** (untagged).  
Le port `sfp14` est connecté à pve1 (nic2) en mode **trunk** (tagged VLAN 50).

### 1.1 Configurer le MTU des ports backup

```routeros
/interface set sfp-sfpplus12 l2mtu=9018 mtu=9000
/interface set sfp-sfpplus14 l2mtu=9018 mtu=9000
/interface set sfp-sfpplus16 l2mtu=9018 mtu=9000
```

### 1.2 Créer le bridge br-backup

```routeros
/interface bridge add name=br-backup mtu=9000 vlan-filtering=yes protocol-mode=rstp
```

### 1.3 Ajouter les ports dans br-backup

```routeros
# sfp12 — access VLAN 50 (NAS backup, untagged)
/interface bridge port add bridge=br-backup interface=sfp-sfpplus12 \
  frame-types=admit-only-untagged-and-priority-tagged pvid=50 hw=yes

# sfp14 — trunk VLAN 50 (pve1 nic2, tagged)
/interface bridge port add bridge=br-backup interface=sfp-sfpplus14 \
  frame-types=admit-only-vlan-tagged pvid=1 hw=yes

# sfp16 — trunk VLAN 50 (pve2 nic2, tagged) — inactif jusqu'au branchement de pve2
/interface bridge port add bridge=br-backup interface=sfp-sfpplus16 \
  frame-types=admit-only-vlan-tagged pvid=1 hw=yes
```

### 1.4 Configurer le VLAN 50 sur br-backup

```routeros
/interface bridge vlan add bridge=br-backup vlan-ids=50 \
  tagged=sfp-sfpplus14,sfp-sfpplus16 \
  untagged=sfp-sfpplus12
```

> sfp16 (pve2 backup) est inclus dans les tagged dès maintenant — il sera inactif jusqu'au branchement de pve2.  
> sfp1 (uplink CRS310) **ne porte pas** le VLAN 50 — le trafic backup est local au CRS317.

### 1.5 Vérification

```routeros
/interface bridge print detail where name=br-backup
# Attendu : mtu=9000 actual-mtu=9000 vlan-filtering=yes

/interface bridge port print detail where bridge=br-backup
# sfp12 : pvid=50, admit-only-untagged-and-priority-tagged
# sfp14 : pvid=1, admit-only-vlan-tagged

/interface bridge vlan print where bridge=br-backup
# VLAN 50 : tagged=sfp14,sfp16 / untagged=sfp12
```

---

## 2. Configuration réseau NAS — Interface backup

L'interface `eth1` (LAN1 du NAS) est dédiée au backup. Elle reçoit l'IP `10.0.50.10` via un bridge
interne nécessaire pour le container LXC. Le tout fonctionne en MTU 9000.

### 2.1 Configurer l'IP via ADM

Dans **ADM → Réglages → Réseau → LAN1** :

- Adresse IP : `10.0.50.10`
- Masque : `255.255.255.0`
- Passerelle : vide (réseau isolé)
- MTU : 9000
- VLAN Tag : **désactivé** (le switch gère le tag, port access)

### 2.2 Créer le bridge eth1-br (persistant au reboot)

Linux Center nécessite un bridge sur `eth1` pour y attacher le container. Ce bridge est créé via un
script de démarrage Asustor. Le script gère également le MTU 9000 sur eth1, eth1-br, et le veth
du container (dont le nom change à chaque démarrage).

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
        ip link set eth1 mtu 9000
        ip link set eth1-br mtu 9000
        for veth in $(ls /sys/class/net/eth1-br/brif/ 2>/dev/null | grep -v eth1); do
            ip link set "$veth" mtu 9000
        done
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

> **Note** : Le script est nommé `S45eth1-bridge` pour s'exécuter **avant** `S50linux-center`
> (qui démarre les containers LXC). Cela garantit que `eth1-br` existe avant que le container
> essaie d'y connecter son interface, et que le MTU est déjà à 9000 avant la boucle veth.

### 2.3 Corriger la route parasite au boot

ADM assigne l'IP `10.0.50.10` directement sur `eth1` au démarrage. Après l'exécution du script
S45, l'IP est déplacée sur `eth1-br` mais ADM peut réassigner l'IP sur `eth1` après le script.
Un cron en boucle nettoie l'IP et la route toutes les 10 secondes pendant 2 minutes :

```bash
sudo crontab -e
```

Ajouter :
```
@reboot for i in $(seq 1 12); do sleep 10 && ip addr del 10.0.50.10/24 dev eth1 2>/dev/null; ip route del 10.0.50.0/24 dev eth1 2>/dev/null; done
```

> Un simple `sleep 30` one-shot ne suffit pas — ADM peut réassigner l'IP après l'exécution.
> La boucle couvre toute la phase d'initialisation d'ADM (~2 minutes).

### 2.4 Vérification

```bash
sudo ip addr show eth1-br
# Attendu : inet 10.0.50.10/24, mtu 9000

ip route show | grep 10.0.50
# Attendu : une seule ligne via eth1-br

ip addr show eth1 | grep inet
# Attendu : aucune IP sur eth1

sudo ping -c 3 10.0.50.20   # ping pve1
```

---

## 3. Configuration Proxmox pve1 — Interface VLAN 50

Le nic2 de pve1 est branché sur `sfp14` du CRS317 (trunk br-backup). Il faut créer une interface
VLAN 50 dessus avec MTU 9000.

### 3.1 Configurer via l'UI Proxmox

**pve1 → System → Network** :

- Double-cliquer sur **nic2** → MTU : `9000` → OK
- Double-cliquer sur **nic2.50** → MTU : `9000` → OK
- Cliquer **Apply Configuration**

> Ne pas modifier `/etc/network/interfaces` manuellement — Proxmox gère ce fichier.

La configuration générée dans `/etc/network/interfaces` :
```
auto nic2
iface nic2 inet manual
        mtu 9000

auto nic2.50
iface nic2.50 inet static
        address 10.0.50.20/24
        vlan-raw-device nic2
        mtu 9000
#BACKUP
```

### 3.2 Vérification

```bash
ip addr show nic2.50
# Attendu : inet 10.0.50.20/24, mtu 9000

ping -M do -s 8972 10.0.50.10   # ping NAS jumbo frames
```

---

## 4. Configuration Proxmox pve2 — Interface VLAN 50 ✅

Même procédure que pve1. pve2 utilise son nic2 branché sur `sfp16` du CRS317.

### 4.1 Configurer via l'UI Proxmox sur pve2

- **nic2** → MTU : `9000`
- **nic2.50** → IP : `10.0.50.21/24`, MTU : `9000`
- **Apply Configuration**

### 4.2 Vérification

```bash
ping -M do -s 8972 10.0.50.10   # ping NAS
ping -M do -s 8972 10.0.50.11   # ping PBS container
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

Le service `networking` standard ne fonctionne pas dans ce container LXC (problème `udevadm`). Un service systemd dédié gère les IPs statiques et le MTU.

### 7.1 Corriger le lien udevadm

```bash
sudo ln -s /usr/bin/udevadm /bin/udevadm
```

### 7.2 Créer le service réseau statique

Le MTU doit être configuré **avant** l'addr add sur eth2 :

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
ExecStart=/bin/sh -c '/sbin/ip link set eth2 mtu 9000 2>/dev/null || true'
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
# eth2 → 10.0.50.11/24, mtu 9000 (backup VLAN 50)

ping -M do -s 8972 10.0.50.10   # NAS
ping -M do -s 8972 10.0.50.20   # pve1
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

## 12. Configuration de pve2 ✅

pve2 est configuré. Les étapes suivantes ont été réalisées :

1. **CRS317** : sfp16 est déjà membre de br-backup en trunk VLAN 50 — vérifier que le lien est actif après branchement
2. **pve2** : Configurer `nic2.50` avec IP `10.0.50.21` et MTU 9000 (cf. section 4)
3. **Proxmox** : Ajouter PBS comme storage sur pve2 (cf. section 11.2)

---

## Résumé des IPs VLAN 50

| Appareil | Interface | IP | MTU |
|---|---|---|---|
| NAS (eth1-br) | bridge sur eth1 | `10.0.50.10/24` | 9000 |
| `nasctl` (eth2) | veth → eth1-br | `10.0.50.11/24` | 9000 |
| pve1 (nic2.50) | VLAN 50 sur nic2 | `10.0.50.20/24` | 9000 |
| pve2 (nic2.50) | VLAN 50 sur nic2 | `10.0.50.21/24` | 9000 | ✅ |
