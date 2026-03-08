# Configuration Switch — MikroTik CRS317-1G-16S+RM

> **Statut :** En production  
> **Date :** Mars 2026  
> **OS :** RouterOS v7

---

## 1. Vue d'ensemble

Le CRS317 est le switch core 10G du homelab. Il gère exclusivement le trafic storage et backup entre les nœuds Proxmox et le NAS. Le trafic management ne passe pas par ce switch — il est entièrement géré par le CRS310.

### 1.1 Rôle dans l'infrastructure

| Rôle | Détail |
|---|---|
| Switch core 10G | Backbone est-ouest storage/backup |
| Uplink vers CRS310 | sfp-sfpplus1 (trunk) |
| Isolation storage | Bridge dédié `br-vmstore` sans VLAN |
| Backup VLAN 50 | `bridge1` avec VLAN filtering |

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
| sfp-sfpplus11 | NAS LAN1 (backup) | VLAN 50 access | bridge1 | 1500 |
| sfp-sfpplus12 | NAS LAN2 (iSCSI) | Storage isolé | br-vmstore | 9000 |
| sfp-sfpplus13 | pve1 nic3 (iSCSI) | Storage isolé | br-vmstore | 9000 |
| sfp-sfpplus14 | pve1 nic2 (backup) | VLAN 50 trunk | bridge1 | 9000 |
| sfp-sfpplus15 | pve2 nic3 (iSCSI) — *à brancher* | Storage isolé | br-vmstore | 9000 |
| sfp-sfpplus16 | pve2 nic2 (backup) — *à brancher* | VLAN 50 trunk | bridge1 | 1500* |

> **Note sfp16 :** MTU à passer à 9000/l2mtu 9018 lors du branchement de pve2.

---

## 3. Architecture des bridges

### 3.1 br-vmstore — Storage iSCSI isolé

Bridge dédié au trafic iSCSI. Pas de VLAN filtering, pas de routage. Réseau `10.0.20.0/24`.

- MTU : 9000
- VLAN filtering : non
- Membres : sfp12 (NAS), sfp13 (pve1), sfp15 (pve2 — inactif)

### 3.2 bridge1 — Backup + uplink

Bridge principal gérant le VLAN 50 (backup) et l'uplink vers le CRS310. VLAN filtering activé.

- MTU : 1500
- VLAN filtering : oui
- Membres : sfp1 (uplink), sfp11 (NAS backup), sfp14 (pve1), sfp16 (pve2 — inactif)

### 3.3 Table VLAN sur bridge1

| VLAN | Tagged | Untagged | Usage |
|---|---|---|---|
| 50 | sfp1, sfp14, sfp16 | sfp11 | Backup PBS |

> sfp11 reçoit le VLAN 50 en **untagged** car le NAS ne fait pas de VLAN tagging.  
> sfp14 et sfp16 reçoivent le VLAN 50 en **tagged** car Proxmox fait le tagging (nic2.50).

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

### 4.3 Créer le bridge br-vmstore (iSCSI, MTU 9000)

```routeros
/interface bridge add name=br-vmstore mtu=9000 vlan-filtering=no protocol-mode=rstp
```

### 4.4 Créer le bridge1 (backup + uplink, VLAN filtering)

```routeros
/interface bridge add name=bridge1 mtu=1500 vlan-filtering=yes protocol-mode=rstp
```

### 4.5 Configurer le MTU et l2mtu sur les ports iSCSI

> L'ordre est critique : l2mtu d'abord, puis mtu.

```routeros
/interface set sfp-sfpplus12 l2mtu=9018 mtu=9000
/interface set sfp-sfpplus13 l2mtu=9018 mtu=9000
/interface set sfp-sfpplus14 l2mtu=9018 mtu=9000
/interface set sfp-sfpplus15 l2mtu=9018 mtu=9000
```

> sfp14 (pve1 backup) est aussi en 9000 pour anticiper la live migration future.  
> sfp16 (pve2 backup) sera configuré lors du branchement de pve2.

### 4.6 Ajouter les ports dans les bridges

```routeros
# br-vmstore — iSCSI (pas de VLAN)
/interface bridge port add bridge=br-vmstore interface=sfp-sfpplus12 hw=yes
/interface bridge port add bridge=br-vmstore interface=sfp-sfpplus13 hw=yes
/interface bridge port add bridge=br-vmstore interface=sfp-sfpplus15 hw=yes

# bridge1 — uplink CRS310 (trunk, admit-all)
/interface bridge port add bridge=bridge1 interface=sfp-sfpplus1 \
  frame-types=admit-all pvid=1 hw=yes

# bridge1 — NAS backup (access VLAN 50, untagged)
/interface bridge port add bridge=bridge1 interface=sfp-sfpplus11 \
  frame-types=admit-only-untagged-and-priority-tagged pvid=50 hw=yes

# bridge1 — pve1 backup (trunk VLAN 50, tagged)
/interface bridge port add bridge=bridge1 interface=sfp-sfpplus14 \
  frame-types=admit-only-vlan-tagged pvid=1 hw=yes

# bridge1 — pve2 backup (trunk VLAN 50, tagged) — inactif jusqu'au branchement
/interface bridge port add bridge=bridge1 interface=sfp-sfpplus16 \
  frame-types=admit-only-vlan-tagged pvid=1 hw=yes
```

### 4.7 Configurer la table VLAN sur bridge1

```routeros
/interface bridge vlan add bridge=bridge1 vlan-ids=50 \
  tagged=sfp-sfpplus1,sfp-sfpplus14,sfp-sfpplus16 \
  untagged=sfp-sfpplus11
```

### 4.8 Vérification

```routeros
# Vérifier les bridges
/interface bridge print

# Vérifier les ports
/interface bridge port print

# Vérifier les VLANs
/interface bridge vlan print

# Vérifier les MTU des interfaces
/interface print where name~"sfp-sfpplus"
```

Résultats attendus :

- `br-vmstore` : `mtu=9000 actual-mtu=9000`
- `bridge1` : `mtu=1500 vlan-filtering=yes`
- sfp12, sfp13, sfp14, sfp15 : `actual-mtu=9000 l2mtu=9018`
- sfp11 : `actual-mtu=1500`
- VLAN 50 : tagged=sfp1,sfp14,sfp16 / untagged=sfp11

### 4.9 Sauvegarder la configuration

```routeros
/system backup save name=crs317-config-initial
```

---

## 5. Ajout de pve2 (à faire)

Quand pve2 sera branché sur sfp15 (iSCSI) et sfp16 (backup), effectuer :

```routeros
# Passer sfp16 en MTU 9000
/interface set sfp-sfpplus16 l2mtu=9018 mtu=9000
```

La config bridge et VLAN est déjà en place — sfp15 et sfp16 sont déjà membres des bridges respectifs. Il suffira de vérifier que les liens sont actifs (flag `R`) après branchement.

---

## 6. Dépannage

### br-vmstore affiche `MTU > L2MTU`

Cause : Un port membre a un l2mtu insuffisant (1584 au lieu de 9018).

```routeros
# Identifier le port fautif
/interface print where name~"sfp-sfpplus"
# Corriger
/interface set sfp-sfpplus<X> l2mtu=9018 mtu=9000
# Retirer et remettre le port dans le bridge pour forcer la mise à jour
/interface bridge port remove [find interface=sfp-sfpplus<X>]
/interface bridge port add bridge=br-vmstore interface=sfp-sfpplus<X> hw=yes
```

### Perte d'accès SSH au switch pendant la configuration

Ne jamais retirer sfp-sfpplus1 de bridge1 — c'est le port uplink vers le CRS310 qui porte l'accès management. Pour retirer tous les autres ports sans couper la session :

```routeros
/interface bridge port remove [find bridge=bridge1 interface!=sfp-sfpplus1]
```

### VLAN 50 ne passe pas vers le NAS

Vérifier que sfp11 est bien en `admit-only-untagged-and-priority-tagged` avec `pvid=50` et qu'il apparaît dans `current-untagged` du VLAN 50 :

```routeros
/interface bridge vlan print detail where vlan-ids=50
/interface bridge port print detail where interface=sfp-sfpplus11
```
