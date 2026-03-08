# Configuration Switch — MikroTik CRS317-1G-16S+RM

> **Statut :** En production  
> **Date :** Mars 2026  
> **OS :** RouterOS v7

---

## 1. Vue d'ensemble

Le CRS317 est le switch core 10G du homelab. Il gère le trafic backup entre les nœuds Proxmox et le NAS. Le trafic management ne passe pas par ce switch — il est entièrement géré par le CRS310.

### 1.1 Rôle dans l'infrastructure

| Rôle | Détail |
|---|---|
| Switch core 10G | Backbone est-ouest backup |
| Uplink vers CRS310 | sfp-sfpplus1 (trunk) |
| Backup + migration | Bridge dédié `br-backup` avec VLAN filtering |

### 1.2 Accès management

Le CRS317 est accessible via SSH ou Winbox à l'adresse `192.168.1.3` (réseau home, temporaire — sera migré vers VLAN 10 lors de la mise en place d'OPNsense).

L'IP est configurée sur `bridge1` :
```
192.168.1.3/24 sur bridge1
```

---

## 2. Plan de câblage physique

| Port | Connecté à | Réseau | Bridge | MTU |
|---|---|---|---|---|
| sfp-sfpplus1 | CRS310 (uplink) | Trunk management | bridge1 | 1500 |
| sfp-sfpplus2–10 | Non connectés | — | — | — |
| sfp-sfpplus11 | NAS LAN2 (VMStore NFS, eth0) | VMStore isolé | br-vmstore | 9000 |
| sfp-sfpplus12 | NAS LAN1 (backup, eth1) | Backup isolé | br-backup | 9000 |
| sfp-sfpplus13 | pve1 nic3 (VMStore NFS) | VMStore isolé | br-vmstore | 9000 |
| sfp-sfpplus14 | pve1 nic2 (backup) | Backup isolé | br-backup | 9000 |
| sfp-sfpplus15 | pve2 nic3 (VMStore NFS) | VMStore isolé | br-vmstore | 9000 |
| sfp-sfpplus16 | pve2 nic2 (backup) | Backup isolé | br-backup | 9000 |

---

## 3. Architecture des bridges

### 3.1 br-vmstore — VMStore NFS isolé

Bridge dédié au trafic NFS VMStore. Pas de VLAN filtering, pas de routage. Réseau `10.0.20.0/24`, MTU 9000.

- Membres : sfp11 (NAS LAN2), sfp13 (pve1 nic3), sfp15 (pve2 nic3)

### 3.2 br-backup — Backup + migration

Bridge dédié au trafic backup (VLAN 50) et futur migration (VLAN 30). VLAN filtering activé pour
permettre plusieurs VLANs sur le même lien physique nic2 des nœuds Proxmox.

- MTU : 9000
- VLAN filtering : oui
- Membres : sfp12 (NAS), sfp14 (pve1), sfp16 (pve2)

### 3.3 bridge1 — Uplink management uniquement

Bridge minimal portant uniquement l'uplink vers le CRS310 et l'IP de management du switch.

- MTU : 1500
- VLAN filtering : oui
- Membres : sfp1 (uplink CRS310)

### 3.4 Table VLAN sur br-backup

| VLAN | Tagged | Untagged | Usage |
|---|---|---|---|
| 50 | sfp14, sfp16 | sfp12 | Backup PBS |
| 30 | sfp14, sfp16 | — | Live migration (à implémenter) |

> sfp12 reçoit le VLAN 50 en **untagged** car le NAS ne fait pas de VLAN tagging.  
> sfp14 et sfp16 reçoivent le VLAN 50 en **tagged** car Proxmox fait le tagging (nic2.50).  
> sfp1 (uplink CRS310) **ne porte pas** le VLAN 50 — le trafic backup reste local au CRS317.

---

## 4. Procédure de configuration pas à pas

> Cette procédure part d'un CRS317 avec RouterOS v7 fraîchement installé ou réinitialisé.

### 4.1 Accès initial

Connecter un câble sur le port `ether1` (RJ45 1G) du CRS317. Par défaut, RouterOS démarre avec une IP sur ce port. Se connecter via Winbox ou SSH.

### 4.2 Configurer l'IP de management

```routeros
/ip address add address=192.168.1.3/24 interface=bridge1
/ip route add gateway=192.168.1.1
```

### 4.3 Créer le bridge br-vmstore (VMStore NFS, MTU 9000)

```routeros
/interface bridge add name=br-vmstore mtu=9000 vlan-filtering=no protocol-mode=rstp
```

### 4.4 Créer le bridge br-backup (backup + migration, MTU 9000)

```routeros
/interface bridge add name=br-backup mtu=9000 vlan-filtering=yes protocol-mode=rstp
```

### 4.5 Créer le bridge1 (uplink management uniquement)

```routeros
/interface bridge add name=bridge1 mtu=1500 vlan-filtering=yes protocol-mode=rstp
```

### 4.6 Configurer le MTU et l2mtu sur les ports 10G

> L'ordre est critique : l2mtu d'abord, puis mtu.

```routeros
# VMStore NFS
/interface set sfp-sfpplus11 l2mtu=9018 mtu=9000
/interface set sfp-sfpplus13 l2mtu=9018 mtu=9000
/interface set sfp-sfpplus15 l2mtu=9018 mtu=9000

# Backup + migration
/interface set sfp-sfpplus12 l2mtu=9018 mtu=9000
/interface set sfp-sfpplus14 l2mtu=9018 mtu=9000
/interface set sfp-sfpplus16 l2mtu=9018 mtu=9000
```

### 4.7 Ajouter les ports dans les bridges

```routeros
# br-vmstore — VMStore NFS (flat, pas de VLAN)
/interface bridge port add bridge=br-vmstore interface=sfp-sfpplus11 hw=yes
/interface bridge port add bridge=br-vmstore interface=sfp-sfpplus13 hw=yes
/interface bridge port add bridge=br-vmstore interface=sfp-sfpplus15 hw=yes

# br-backup — backup VLAN 50 (NAS untagged, hosts tagged)
/interface bridge port add bridge=br-backup interface=sfp-sfpplus12 \
  frame-types=admit-only-untagged-and-priority-tagged pvid=50 hw=yes

/interface bridge port add bridge=br-backup interface=sfp-sfpplus14 \
  frame-types=admit-only-vlan-tagged pvid=1 hw=yes

/interface bridge port add bridge=br-backup interface=sfp-sfpplus16 \
  frame-types=admit-only-vlan-tagged pvid=1 hw=yes

# bridge1 — uplink CRS310 uniquement
/interface bridge port add bridge=bridge1 interface=sfp-sfpplus1 \
  frame-types=admit-all pvid=1 hw=yes
```

### 4.8 Configurer la table VLAN sur br-backup

```routeros
/interface bridge vlan add bridge=br-backup vlan-ids=50 \
  tagged=sfp-sfpplus14,sfp-sfpplus16 \
  untagged=sfp-sfpplus12
```

### 4.9 Vérification

```routeros
# Vérifier les bridges
/interface bridge print

# Vérifier les ports de br-vmstore
/interface bridge port print detail where bridge=br-vmstore

# Vérifier les ports de br-backup
/interface bridge port print detail where bridge=br-backup

# Vérifier les VLANs de br-backup
/interface bridge vlan print where bridge=br-backup

# Vérifier les MTU des interfaces
/interface print detail where name~"sfp-sfpplus1[1-6]"
```

Résultats attendus :

- `br-vmstore` : `mtu=9000 actual-mtu=9000 vlan-filtering=no`
- `br-backup` : `mtu=9000 actual-mtu=9000 vlan-filtering=yes`
- `bridge1` : `mtu=1500 vlan-filtering=yes`
- sfp11–sfp16 : `mtu=9000 l2mtu=9018`
- VLAN 50 sur br-backup : tagged=sfp14,sfp16 / untagged=sfp12

### 4.10 Sauvegarder la configuration

```routeros
/system backup save name=crs317-config-br-backup
```

---

## 5. Ajout de pve2 ✅

pve2 est branché sur sfp16 (backup). La config bridge et VLAN était déjà en place. Le lien est actif.

---

## 6. Dépannage

### br-backup affiche `MTU > L2MTU`

Cause : Un port membre a un l2mtu insuffisant (1584 au lieu de 9018).

```routeros
# Identifier le port fautif
/interface print detail where name~"sfp-sfpplus1[2,4,6]"
# Corriger
/interface set sfp-sfpplus<X> l2mtu=9018 mtu=9000
# Retirer et remettre le port dans le bridge pour forcer la mise à jour
/interface bridge port remove [find interface=sfp-sfpplus<X>]
/interface bridge port add bridge=br-backup interface=sfp-sfpplus<X> hw=yes
```

### Perte d'accès SSH au switch pendant la configuration

Ne jamais retirer sfp-sfpplus1 de bridge1 — c'est le port uplink vers le CRS310 qui porte
l'accès management.

### VLAN 50 ne passe pas vers le NAS

Vérifier que sfp12 est bien en `admit-only-untagged-and-priority-tagged` avec `pvid=50` et
qu'il apparaît dans `current-untagged` du VLAN 50 sur br-backup :

```routeros
/interface bridge vlan print detail where vlan-ids=50
/interface bridge port print detail where interface=sfp-sfpplus12
```

