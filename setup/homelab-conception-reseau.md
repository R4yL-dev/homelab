# Conception Réseau — Homelab

> **Phase :** En cours d'implémentation  
> **Statut :** Document de référence — mis à jour au fur et à mesure  
> **Date :** Mars 2026

---

## 1. Plan de VLANs

### 1.1 Tableau des VLANs

| VLAN ID | Nom | Rôle | Type de trafic | MTU | Gateway | Routage | Statut |
|---------|-----|------|----------------|-----|---------|---------|--------|
| 1 | Native | **Non utilisé** (sécurité) | — | — | — | — | — |
| 20 | STORAGE | iSCSI stockage | I/O disque VMs | 9000 | Aucune | Isolé | ✅ En place |
| 30 | MIGRATION | Live migration Proxmox | Transfert mémoire VM | 9000 | Aucune | Isolé | 🔲 À implémenter |
| 40 | CLUSTER | Corosync + Qdevice | Heartbeat cluster | 1500 | Aucune | Isolé | 🔲 À implémenter |
| 50 | BACKUP | Backup vers PBS | Transfert données backup | 9000 | Aucune | Isolé | ✅ En place |
| 10 | MGMT | Management infrastructure | Web UI, SSH | 1500 | 10.0.10.1 | Via OPNsense | 🔲 À implémenter |
| 100 | WAN-TRANSIT | Lien OPNsense ↔ Box FAI | Trafic Internet | 1500 | 192.168.1.1 | Via OPNsense | 🔲 À implémenter |
| 110 | LAN-VMS | Réseau VMs/LXC production | Trafic applicatif VMs | 1500 | 10.0.110.1 | Via OPNsense | 🔲 À implémenter |
| 120 | DMZ | Services exposés (futur) | Trafic public | 1500 | 10.0.120.1 | Via OPNsense | 🔲 À implémenter |

> **Note :** En attendant la mise en place d'OPNsense et du VLAN 10, tout le management transite par le réseau home `192.168.1.0/24` via le CRS310.

### 1.2 Matrice de Flux Inter-VLAN (cible)

| Source ↓ \ Dest → | MGMT | STORAGE | MIGR | CLUSTER | BACKUP | WAN | LAN-VMS | DMZ |
|--------------------|:----:|:-------:|:----:|:-------:|:------:|:---:|:-------:|:---:|
| **MGMT (10)** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **STORAGE (20)** | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **MIGRATION (30)** | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **CLUSTER (40)** | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **BACKUP (50)** | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| **WAN (100)** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | via OPNs | via OPNs |
| **LAN-VMS (110)** | ❌ | ❌ | ❌ | ❌ | ❌ | via OPNs | ✅ | ❌ |
| **DMZ (120)** | ❌ | ❌ | ❌ | ❌ | ❌ | via OPNs | ❌ | ✅ |

> **Principe :** Les VLANs infra (20, 30, 40, 50) sont strictement isolés — aucune route entre eux.
> Seul le VLAN MGMT (10) pourra atteindre tous les autres une fois OPNsense en place.

---

## 2. Plan d'Adressage IP

### 2.1 Schéma de Numérotation

**Réseau global :** `10.0.0.0/16`

**Convention par VLAN :** `10.0.{VLAN_ID}.0/24`

**Convention d'adresses au sein de chaque VLAN :**

| Plage | Usage |
|-------|-------|
| .1 | Gateway (OPNsense, si applicable) |
| .2 — .9 | Équipements réseau (switches) |
| .10 — .19 | Stockage (NAS, PBS) |
| .20 — .29 | Nœuds compute |
| .100 — .250 | DHCP (si applicable) |

### 2.2 Adresses par Équipement

#### Switches

| Équipement | IP actuelle | IP cible (VLAN 10) | Accès actuel |
|---|---|---|---|
| CRS310 | 192.168.1.2/24 | 10.0.10.2 | bridge1 réseau home |
| CRS317 | 192.168.1.3/24 | 10.0.10.3 | bridge1 réseau home |

#### Nœud 1 — Minisforum MS-01 (pve1)

| VLAN | Interface physique | Adresse IP actuelle | Statut |
|------|-------------------|---------------------|--------|
| Home (temp.) | RJ45 nic0 → vmbr0 | 192.168.1.20/24 | ✅ En place |
| 20 — STORAGE | SFP+ nic3 | 10.0.20.20/24 | ✅ En place |
| 50 — BACKUP | SFP+ nic2 → nic2.50 | 10.0.50.20/24 | ✅ En place |
| 10 — MGMT | RJ45 nic0 → vmbr0 | 10.0.10.20/24 | 🔲 À implémenter |
| 30 — MIGRATION | SFP+ nic2 → nic2.30 | 10.0.30.20/24 | 🔲 À implémenter |
| 40 — CLUSTER | RJ45 nic0 → nic0.40 | 10.0.40.20/24 | 🔲 À implémenter |

#### Nœud 2 — Minisforum MS-01 (pve2)

| VLAN | Interface physique | Adresse IP | Statut |
|------|-------------------|------------|--------|
| Home (temp.) | RJ45 nic0 → vmbr0 | 192.168.1.21/24 | ✅ En place |
| 20 — STORAGE | SFP+ nic3 | 10.0.20.21/24 | ✅ En place |
| 50 — BACKUP | SFP+ nic2 → nic2.50 | 10.0.50.21/24 | ✅ En place |
| 10 — MGMT | RJ45 nic0 → vmbr0 | 10.0.10.21/24 | 🔲 À implémenter |
| 30 — MIGRATION | SFP+ nic2 → nic2.30 | 10.0.30.21/24 | 🔲 À implémenter |
| 40 — CLUSTER | RJ45 nic0 → nic0.40 | 10.0.40.21/24 | 🔲 À implémenter |

#### NAS — Asustor AS6806T

| Interface physique | IP actuelle | Réseau | Statut |
|---|---|---|---|
| LAN1 (10G eth1) | 10.0.50.10/24 | Backup VLAN 50 | ✅ En place |
| LAN2 (10G eth0) | 10.0.20.10/24 | iSCSI VLAN 20 | ✅ En place |
| LAN3 (2.5G eth3) | 192.168.1.10/24 | Management home | ✅ En place |
| LAN4 (2.5G eth2) | — | Non utilisé | — |

#### PBS (container LXC nasctl sur NAS)

| Interface | IP | Réseau | Statut |
|---|---|---|---|
| eth0 | 192.168.1.11/24 | Management home | ✅ En place |
| eth2 | 10.0.50.11/24 | Backup VLAN 50 | ✅ En place |

#### OPNsense VM (à implémenter)

| VLAN | Adresse IP | Rôle | Statut |
|------|------------|------|--------|
| 10 — MGMT | 10.0.10.1/24 | Gateway MGMT, Tailscale | 🔲 À implémenter |
| 100 — WAN | DHCP (192.168.1.x) | Interface WAN | 🔲 À implémenter |
| 110 — LAN-VMS | 10.0.110.1/24 | Gateway VMs | 🔲 À implémenter |
| 120 — DMZ | 10.0.120.1/24 | Gateway DMZ | 🔲 À implémenter |

---

## 3. Configuration des Switches

### 3.1 CRS317-1G-16S+RM — Switch Core 10G

**Rôle :** Backbone est-ouest, transport des flux 10G storage et backup.  
**OS :** RouterOS v7  
**IP management actuelle :** `192.168.1.3/24` sur `bridge1`

#### Bridges

| Bridge | MTU | VLAN Filtering | Rôle |
|---|---|---|---|
| `br-vmstore` | 9000 | Non | iSCSI isolé, réseau `10.0.20.0/24` |
| `br-backup` | 9000 | Oui | Backup VLAN 50 + migration VLAN 30 (futur) |
| `bridge1` | 1500 | Oui | Uplink CRS310 + IP management switch |

#### Plan de câblage

| Port | Connecté à | Bridge | Mode | VLAN | MTU | Statut |
|---|---|---|---|---|---|---|
| sfp-sfpplus1 | CRS310 uplink | bridge1 | Trunk admit-all | — | 1500 | ✅ |
| sfp-sfpplus11 | NAS LAN2 (iSCSI, eth0) | br-vmstore | Access direct | Aucun | 9000 | ✅ |
| sfp-sfpplus12 | NAS LAN1 (backup, eth1) | br-backup | Access untagged | VLAN 50 | 9000 | ✅ |
| sfp-sfpplus13 | pve1 nic3 (iSCSI) | br-vmstore | Access direct | Aucun | 9000 | ✅ |
| sfp-sfpplus14 | pve1 nic2 (backup) | br-backup | Trunk tagged | VLAN 50 | 9000 | ✅ |
| sfp-sfpplus15 | pve2 nic3 (iSCSI) | br-vmstore | Access direct | Aucun | 9000 | ✅ |
| sfp-sfpplus16 | pve2 nic2 (backup) | br-backup | Trunk tagged | VLAN 50 | 9000 | ✅ |
| sfp-sfpplus2–10 | Non connectés | — | — | — | — | — |

#### Table VLAN br-backup

| VLAN | Tagged | Untagged |
|---|---|---|
| 50 | sfp14, sfp16 | sfp12 |
| 30 | sfp14, sfp16 | — |

> sfp12 (NAS) reçoit VLAN 50 en untagged — le NAS ne fait pas de VLAN tagging.  
> sfp14/sfp16 (hosts) reçoivent VLAN 50 en tagged — Proxmox fait le tagging (nic2.50).  
> sfp1 (uplink CRS310) **ne porte pas** le VLAN 50 — le trafic backup est local au CRS317.

---

### 3.2 CRS310-8G+2S+IN — Switch Access 2.5G

**Rôle :** Switch d'accès management. Tout le management transite par ce switch.  
**OS :** RouterOS v7  
**IP management actuelle :** `192.168.1.2/24` sur `bridge1`

> **Note :** Le CRS310 n'a pas encore de VLAN filtering configuré. Tout est dans un seul bridge flat pour l'instant. La configuration VLAN sera mise en place lors de l'implémentation du VLAN 10 management et d'OPNsense.

#### Plan de câblage actuel

| Port | Connecté à | Statut |
|---|---|---|
| sfp-sfpplus2 | CRS317 uplink | ✅ |
| ether1 | Réseau home (box FAI) | ✅ |
| ether7 | NAS LAN3 (management) | ✅ |
| ether8 | pve1 nic0 (management) | ✅ |
| ether2–6, sfp-sfpplus1 | Non connectés | — |

#### Plan de câblage cible (après implémentation VLAN)

| Port | Connecté à | VLANs | Mode | Statut |
|---|---|---|---|---|
| sfp-sfpplus2 | CRS317 uplink | 10, 100, 110, 120 | Trunk | 🔲 |
| ether1 | Box FAI | 100 | Access | 🔲 |
| ether2 | pve1 RJ45 #1 | 10 (untagged), 40 (tagged) | Trunk | 🔲 |
| ether3 | pve2 RJ45 #1 | 10 (untagged), 40 (tagged) | Trunk | 🔲 |
| ether4 | pve1 RJ45 #2 | 100, 110, 120 | Trunk | 🔲 |
| ether5 | pve2 RJ45 #2 | 100, 110, 120 | Trunk | 🔲 |
| ether6 | NAS LAN3 | 10 | Access | 🔲 |
| ether7 | NAS LAN4 | 40 | Access | 🔲 |

---

## 4. Trunk Inter-Switch (CRS317 ↔ CRS310)

### 4.1 État actuel

Le trunk sfp1↔sfp1 transporte tout le trafic sans VLAN filtering (bridge flat des deux côtés).

### 4.2 État cible

| VLAN | Raison de traverser le trunk |
|------|------------------------------|
| 10 — MGMT | Management des équipements des deux côtés |
| 100 — WAN | Box FAI sur CRS310, OPNsense sur nœud CRS317 |
| 110 — LAN-VMS | VMs accèdent à Internet via OPNsense/box FAI |
| 120 — DMZ | Idem LAN-VMS |

**VLANs qui ne traversent PAS le trunk :**

| VLAN | Raison |
|------|--------|
| 20 — STORAGE | Local CRS317 uniquement |
| 30 — MIGRATION | Local CRS317 uniquement |
| 50 — BACKUP | Local CRS317 uniquement |
| 40 — CLUSTER | Local CRS310 uniquement |

---

## 5. Schéma de Câblage Physique

### 5.1 État actuel

```
                     ┌──────────────┐
                     │   Box FAI    │
                     │ 192.168.1.1  │
                     └──────┬───────┘
                            │ ether1
   ┌────────────────────────┴──────────────────────────┐
   │           CRS310-8G+2S+IN                         │
   │           IP: 192.168.1.2                         │
   │                                                   │
   │  ether1    ether7    ether8    sfp+2               │
   │  Box FAI   NAS       pve1      CRS317              │
   │            LAN3      nic0      uplink              │
   └───────────────────────┬───────────────────────────┘
                           │ sfp+2 ↔ sfp+1
   ┌───────────────────────┴───────────────────────────┐
   │           CRS317-1G-16S+RM                        │
   │           IP: 192.168.1.3                         │
   │                                                   │
   │  sfp+1   sfp+11  sfp+12  sfp+13  sfp+14           │
   │  CRS310  NAS     NAS     pve1    pve1              │
   │  uplink  LAN2    LAN1    nic3    nic2              │
   │          iSCSI   backup  iSCSI   backup            │
   └──────┬──────┬───────┬────────┬───────────────────┘
          │      │       │        │
     ┌────┴──┐ ┌─┴────┐ ┌┴─────┐ ┌┴─────┐
     │  NAS  │ │  NAS │ │ pve1 │ │ pve1 │
     │ LAN2  │ │ LAN1 │ │ nic3 │ │ nic2 │
     │iSCSI  │ │backup│ │iSCSI │ │backup│
     │10.0.  │ │10.0. │ │10.0. │ │10.0. │
     │20.10  │ │50.10 │ │20.20 │ │50.20 │
     └───────┘ └──────┘ └──────┘ └──────┘
```

### 5.2 Câblage cible (après ajout pve2)

sfp+15 → pve2 nic3 (iSCSI), sfp+16 → pve2 nic2 (backup) seront ajoutés.

---

## 6. Résumé des Interfaces par Équipement

### 6.1 pve1 (état actuel)

| Interface | Bridge/VLAN | IP | Commentaire |
|---|---|---|---|
| nic0 | vmbr0 | 192.168.1.20/24 | Management home (temporaire) |
| nic2 | — | — | Port physique backup, MTU 9000 |
| nic2.50 | — | 10.0.50.20/24 | Backup VLAN 50, MTU 9000 |
| nic3 | — | 10.0.20.20/24 | iSCSI, MTU 9000 |

### 6.2 NAS Asustor AS6806T (état actuel)

| Interface | IP | Réseau | Connecté à |
|---|---|---|---|
| eth0 (LAN2) | 10.0.20.10/24 | iSCSI | CRS317 sfp11 |
| eth1 (LAN1) | 10.0.50.10/24 via eth1-br | Backup VLAN 50 | CRS317 sfp12 |
| eth3 (LAN3) | 192.168.1.10/24 | Management home | CRS310 ether7 |

### 6.3 Container nasctl (PBS)

| Interface | IP | Réseau |
|---|---|---|
| eth0 | 192.168.1.11/24 | Management home |
| eth2 | 10.0.50.11/24 | Backup VLAN 50 (via eth1-br) |

---

## 7. Points Techniques Notables

### 7.1 Route parasite NAS au boot

ADM assigne l'IP `10.0.50.10` directement sur `eth1` au démarrage, créant une route parasite qui
empêche la communication backup. Deux mécanismes corrigent cela :

- **Script S45eth1-bridge** (`/usr/local/etc/init.d/S45eth1-bridge`) : crée le bridge `eth1-br`,
  déplace l'IP sur `eth1-br`, monte le MTU à 9000 sur eth1/eth1-br et sur le veth du container.
- **Cron `@reboot` en boucle** : ADM peut réassigner l'IP après le script. Un cron tente de
  supprimer l'IP et la route parasite toutes les 10 secondes pendant 2 minutes :
  ```
  @reboot for i in $(seq 1 12); do sleep 10 && ip addr del 10.0.50.10/24 dev eth1 2>/dev/null; ip route del 10.0.50.0/24 dev eth1 2>/dev/null; done
  ```

### 7.2 Veth container LXC et MTU

Le veth connectant le container nasctl à `eth1-br` est recréé à chaque démarrage du container avec
un nom aléatoire et MTU 1500. Le script S45eth1-bridge contient une boucle qui monte le MTU de
tous les membres de eth1-br (hors eth1) à 9000 après chaque démarrage de container.

### 7.3 iSCSI — connexion manuelle obligatoire

ADM annonce le target iSCSI sur toutes ses interfaces. Les portals management et NAT interne doivent être supprimés manuellement pour forcer le trafic sur `10.0.20.0/24` uniquement. Voir `iscsi-proxmox-asustor.md`.

### 7.4 LVM Thin sur iSCSI partagé

Le VG `vg-vmstore` est partagé entre pve1 et pve2 via iSCSI. Les disques VM sont sur un thin pool LVM — les snapshots live ne sont pas supportés sur ce type de storage.

---

## 8. Checklist d'État

| Composant | Statut |
|---|---|
| CRS317 — bridge br-vmstore (iSCSI) | ✅ |
| CRS317 — bridge br-backup (backup MTU 9000) | ✅ |
| CRS317 — bridge1 (uplink uniquement) | ✅ |
| CRS317 — MTU 9000 iSCSI | ✅ |
| CRS317 — MTU 9000 backup | ✅ |
| CRS310 — bridge flat management | ✅ |
| NAS — iSCSI LUN + target | ✅ |
| NAS — bridge eth1-br backup MTU 9000 | ✅ |
| pve1 — iSCSI + VG vg-vmstore | ✅ |
| pve1 — VLAN 50 backup MTU 9000 | ✅ |
| PBS (nasctl) — installé et opérationnel | ✅ |
| pve2 — branchement et configuration | ✅ |
| Cluster Proxmox (pve1 + pve2) | 🔲 |
| OPNsense VM | 🔲 |
| VLAN 10 management | 🔲 |
| VLAN 30 live migration | 🔲 |
| VLAN 40 Corosync | 🔲 |
| VLAN 100/110/120 trafic VMs | 🔲 |
