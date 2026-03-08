# Jumbo Frames MTU 9000 — Réseau Backup (br-backup)

## Vue d'ensemble

Ce document décrit la procédure pour activer les jumbo frames (MTU 9000) sur le réseau backup
dédié `10.0.50.x` entre les nœuds Proxmox, le NAS Asustor AS6806T et le container PBS (nasctl),
via le switch MikroTik CRS317.

### Infrastructure concernée

| Équipement | Interface | Rôle |
|---|---|---|
| MikroTik CRS317 | `sfp-sfpplus12` | Port NAS (backup) |
| MikroTik CRS317 | `sfp-sfpplus14` | Port pve1 (backup) |
| MikroTik CRS317 | `sfp-sfpplus16` | Port pve2 (backup) |
| MikroTik CRS317 | `br-backup` | Bridge backup dédié |
| Asustor AS6806T | `eth1` + `eth1-br` | Interface 10G backup |
| Container nasctl | `eth2` | Interface backup PBS |
| Proxmox pve1 | `nic2` + `nic2.50` | Interface 10G backup |

### Points techniques importants

**Ordre des opérations sur le CRS317 :** le chip Marvell-98DX8216 a une limite hardware de
`l2mtu=1584` par défaut. Il faut setter le `l2mtu` sur les interfaces physiques **avant** le
`mtu`, sinon RouterOS refuse de monter le bridge à MTU 9000.

**Bridge dédié avec VLAN filtering :** `br-backup` porte le VLAN 50 (backup) et est préparé
pour le futur VLAN 30 (live migration). Le VLAN filtering est activé — sfp12 (NAS) reçoit
le VLAN 50 en untagged, sfp14/sfp16 (Proxmox) en tagged via subinterface `nic2.50`.

**Veth du container LXC :** le veth connectant nasctl à `eth1-br` est recréé à chaque démarrage
du container avec un nom aléatoire et MTU 1500 par défaut. Il faut le monter à 9000 via une
boucle dans le script de démarrage du NAS.

---

## Étape 1 — CRS317 : setter le l2mtu et MTU sur les ports backup

> L'ordre est critique : l2mtu d'abord, puis mtu.

```routeros
/interface set sfp-sfpplus12 l2mtu=9018 mtu=9000
/interface set sfp-sfpplus14 l2mtu=9018 mtu=9000
/interface set sfp-sfpplus16 l2mtu=9018 mtu=9000
```

> Le l2mtu doit être à **9018** (9000 + 18 bytes d'headers Ethernet/VLAN).

Vérifier :
```routeros
/interface print detail where name~"sfp-sfpplus1[2-6]"
# Attendu : mtu=9000 l2mtu=9018 sur sfp12, sfp14, sfp16
```

---

## Étape 2 — CRS317 : créer le bridge br-backup à MTU 9000

```routeros
/interface bridge add name=br-backup mtu=9000 vlan-filtering=yes protocol-mode=rstp
```

---

## Étape 3 — CRS317 : ajouter les ports dans br-backup

```routeros
# sfp12 — NAS backup (untagged VLAN 50)
/interface bridge port add bridge=br-backup interface=sfp-sfpplus12 \
  frame-types=admit-only-untagged-and-priority-tagged pvid=50 hw=yes

# sfp14 — pve1 backup (tagged VLAN 50)
/interface bridge port add bridge=br-backup interface=sfp-sfpplus14 \
  frame-types=admit-only-vlan-tagged pvid=1 hw=yes

# sfp16 — pve2 backup (tagged VLAN 50)
/interface bridge port add bridge=br-backup interface=sfp-sfpplus16 \
  frame-types=admit-only-vlan-tagged pvid=1 hw=yes

# Table VLAN 50
/interface bridge vlan add bridge=br-backup vlan-ids=50 \
  tagged=sfp-sfpplus14,sfp-sfpplus16 \
  untagged=sfp-sfpplus12
```

---

## Étape 4 — Vérification côté CRS317

```routeros
/interface bridge print detail where name=br-backup
# Attendu : mtu=9000 actual-mtu=9000 vlan-filtering=yes

/interface bridge port print detail where bridge=br-backup
# sfp12 : pvid=50, admit-only-untagged-and-priority-tagged
# sfp14 : pvid=1, admit-only-vlan-tagged
# sfp16 : pvid=1, admit-only-vlan-tagged

/interface bridge vlan print where bridge=br-backup
# VLAN 50 : tagged=sfp14,sfp16 / untagged=sfp12
```

---

## Étape 5 — NAS : setter le MTU sur LAN 1 via ADM

Dans **ADM → Réglages → Réseau → LAN1** (interface `eth1`, IP `10.0.50.10`) :

- Champ **MTU** : `9000`
- Valider

---

## Étape 6 — NAS : configurer le bridge eth1-br avec MTU 9000

Linux Center nécessite un bridge sur `eth1` pour y attacher le container. Le script de démarrage
`/usr/local/etc/init.d/S45eth1-bridge` crée ce bridge, y déplace l'IP, et monte le MTU à 9000
sur toutes les interfaces concernées — y compris le veth du container dont le nom change à chaque
démarrage.

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

> Le script est nommé `S45eth1-bridge` pour s'exécuter avant `S50linux-center` (démarrage des
> containers LXC), garantissant que `eth1-br` et ses MTU sont prêts avant l'attachement du veth.

---

## Étape 7 — NAS : corriger la route parasite au boot

ADM réassigne l'IP `10.0.50.10` sur `eth1` après le démarrage, créant une route parasite. Un cron
en boucle la supprime toutes les 10 secondes pendant 2 minutes, couvrant toute la phase
d'initialisation d'ADM :

```bash
sudo crontab -e
```

Ajouter :
```
@reboot for i in $(seq 1 12); do sleep 10 && ip addr del 10.0.50.10/24 dev eth1 2>/dev/null; ip route del 10.0.50.0/24 dev eth1 2>/dev/null; done
```

---

## Étape 8 — nasctl : setter le MTU sur eth2

Dans `/etc/systemd/system/network-static.service`, le MTU doit être configuré **avant** l'addr
add sur eth2 :

```ini
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c '/sbin/ip addr add 192.168.1.11/24 dev eth0 2>/dev/null || true'
ExecStart=/bin/sh -c '/sbin/ip route add default via 192.168.1.1 2>/dev/null || true'
ExecStart=/bin/sh -c '/sbin/ip link set eth2 mtu 9000 2>/dev/null || true'
ExecStart=/bin/sh -c '/sbin/ip addr add 10.0.50.11/24 dev eth2 2>/dev/null || true'
```

```bash
sudo systemctl daemon-reload
```

---

## Étape 9 — Proxmox : setter le MTU sur nic2 et nic2.50

Via l'UI Proxmox : **pve1 → System → Network**

1. Double-cliquer sur **nic2** → MTU : `9000` → OK
2. Double-cliquer sur **nic2.50** → MTU : `9000` → OK
3. Cliquer **Apply Configuration**

Proxmox génère dans `/etc/network/interfaces` :
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

Vérifier :
```bash
ip link show nic2 | grep mtu
ip link show nic2.50 | grep mtu
# Attendu : mtu 9000 sur les deux
```

> Répéter pour **pve2** avec l'IP `10.0.50.21`.

---

## Étape 10 — Vérification end-to-end

```bash
# Depuis pve1 — 8972 = 9000 - 28 bytes (headers IP + ICMP)
ping -M do -s 8972 10.0.50.10   # NAS
ping -M do -s 8972 10.0.50.11   # nasctl PBS
```

```bash
# Depuis nasctl
ping -M do -s 8972 10.0.50.10   # NAS
ping -M do -s 8972 10.0.50.20   # pve1
```

Résultat attendu :
```
8980 bytes from 10.0.50.x: icmp_seq=1 ttl=64 time=0.x ms
```

Si tous les pings passent → MTU 9000 opérationnel sur tout le chemin ✅

---

## Étape 11 — Sauvegarder la config MikroTik

```routeros
/system backup save name=crs317-config-br-backup-mtu9000
```

---

## Dépannage

### br-backup affiche `actual-mtu=1500` malgré `mtu=9000`

**Cause :** Un port membre a un l2mtu insuffisant (1584). sfp16 (inactif) est souvent en cause car
il est membre du bridge même sans lien actif.

```routeros
/interface print detail where name~"sfp-sfpplus1[2-6]"
# Identifier le port à l2mtu=1584
/interface set sfp-sfpplus<X> l2mtu=9018 mtu=9000
```

### Veth container à MTU 1500 après redémarrage du container

Le nom du veth change à chaque démarrage. Vérifier que la boucle `for veth` est bien présente
dans S45eth1-bridge. Pour corriger manuellement :

```bash
# Sur le NAS
ls /sys/class/net/eth1-br/brif/
sudo ip link set <nom_veth> mtu 9000
```

### Route parasite 10.0.50.0/24 sur eth1 après reboot

```bash
sudo ip addr del 10.0.50.10/24 dev eth1 2>/dev/null
sudo ip route del 10.0.50.0/24 dev eth1 2>/dev/null
```

Vérifier que le cron `@reboot` avec la boucle est bien en place :
```bash
sudo crontab -l | grep reboot
```

### Ping jumbo frames échoue sur un segment

Tester par maillon pour isoler le problème :

```bash
ping -M do -s 1472 10.0.50.10   # MTU 1500 standard — doit toujours passer
ping -M do -s 4000 10.0.50.10   # Test intermédiaire
ping -M do -s 8972 10.0.50.10   # MTU 9000
```

Si le 4000 passe mais pas le 8972, vérifier le MTU de chaque interface sur le chemin :
`eth1` → `eth1-br` → veth → `eth2` (nasctl) / `nic2` → `nic2.50` (pve1) / `sfp12` → `sfp14` → `br-backup` (CRS317).
