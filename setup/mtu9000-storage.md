# Jumbo Frames MTU 9000 — MikroTik CRS317 + Proxmox + Asustor AS6806T

## Vue d'ensemble

Ce document décrit la procédure pour activer les jumbo frames (MTU 9000) sur le réseau storage dédié `10.0.20.x` entre le NAS Asustor AS6806T et les nœuds Proxmox, via le switch MikroTik CRS317.

### Infrastructure concernée

| Équipement | Interface | Rôle |
|---|---|---|
| MikroTik CRS317 | `sfp-sfpplus11` | Port NAS |
| MikroTik CRS317 | `sfp-sfpplus14` | Port Proxmox pve1 |
| MikroTik CRS317 | `br-vmstore` | Bridge storage dédié |
| Asustor AS6806T | `eth0` | Interface 10G storage |
| Proxmox pve1 | `nic3` | Interface 10G storage |

### Ordre des opérations critique

Le chip **Marvell-98DX8216** du CRS317 a une limite hardware de `l2mtu=1584` par défaut. RouterOS refuse de setter le MTU sur le bridge si les interfaces membres ont un l2mtu insuffisant. L'ordre correct est :

1. Setter le `l2mtu` sur les interfaces physiques
2. Setter le `mtu` sur les interfaces physiques
3. Setter le `mtu` sur le bridge

Le HW offload reste **actif** (`hw=yes`) — il n'est pas nécessaire de le désactiver avec cette approche.

---

## Étape 1 — MikroTik CRS317 : setter le l2mtu et MTU sur les ports physiques

```routeros
/interface ethernet set sfp-sfpplus11 l2mtu=9018 mtu=9000
/interface ethernet set sfp-sfpplus14 l2mtu=9018 mtu=9000
```

> Le l2mtu doit être à **9018** (9000 + 18 bytes d'headers Ethernet/VLAN) pour permettre le transport de frames MTU 9000.

---

## Étape 2 — MikroTik CRS317 : setter le MTU sur le bridge

```routeros
/interface bridge set br-vmstore mtu=9000
```

---

## Étape 3 — Vérification côté MikroTik

```routeros
# Vérifier les ports physiques
/interface ethernet print detail where name=sfp-sfpplus11
/interface ethernet print detail where name=sfp-sfpplus14

# Vérifier le bridge
/interface bridge print detail where name=br-vmstore
```

Résultats attendus pour les ports :
```
mtu=9000 l2mtu=9018
```

Résultat attendu pour le bridge :
```
mtu=9000 actual-mtu=9000
```

---

## Étape 4 — NAS Asustor AS6806T : setter le MTU sur LAN 2

1. Ouvrir **Réglages → Réseau → Interface réseau**
2. Sélectionner **LAN 2** (interface `eth0`, IP `10.0.20.10`)
3. Cliquer **Configurer**
4. Changer le champ **MTU** à `9000`
5. Valider

Vérification attendue dans la vue LAN 2 :
```
MTU : 9000
Etat du réseau : 10000Mb/s, Full-duplex
```

---

## Étape 5 — Proxmox : setter le MTU sur nic3 via l'UI

1. Aller dans **pve1 → System → Network**
2. Double-cliquer sur **nic3** (interface storage `10.0.20.20/24`, commentaire `ISCSI`)
3. Dans le champ **MTU**, saisir `9000`
4. Cliquer **OK**
5. Cliquer **Apply Configuration**

Proxmox génère automatiquement dans `/etc/network/interfaces` :
```
auto nic3
iface nic3 inet static
        address 10.0.20.20/24
        mtu 9000
#ISCSI
```

Vérification :
```bash
ip link show nic3
# Doit afficher : mtu 9000
```

---

## Étape 6 — Vérification end-to-end

Depuis Proxmox, tester que les jumbo frames passent sur tout le chemin :

```bash
# 8972 = 9000 - 28 bytes (headers IP + ICMP)
ping -M do -s 8972 10.0.20.10
```

Résultat attendu :
```
8980 bytes from 10.0.20.10: icmp_seq=1 ttl=64 time=0.6 ms
```

Si le ping passe → MTU 9000 opérationnel sur tout le chemin ✅

---

## Étape 7 — Sauvegarder la config MikroTik

```routeros
/system backup save name=backup-mtu9000-storage
```

---

## Dépannage

### `failure: could not set mtu` sur le bridge
**Cause** : Le l2mtu des interfaces physiques est encore à `1584`, insuffisant pour supporter MTU 9000.  
**Solution** : S'assurer d'avoir d'abord setté `l2mtu=9018` sur `sfp-sfpplus11` et `sfp-sfpplus14` avant de toucher le bridge.

### Les ports restent à `l2mtu=1584` malgré la commande
**Cause** : Il faut setter le `l2mtu` **en premier** sur l'interface physique, avant le `mtu`.  
**Solution** : Respecter l'ordre — `l2mtu=9018` puis `mtu=9000` sur chaque port.

### Le ping avec `-s 8972` échoue encore
Vérifier chaque maillon individuellement :
```bash
# Test MTU 1500 standard (doit toujours passer)
ping -M do -s 1472 10.0.20.10

# Test intermédiaire
ping -M do -s 4000 10.0.20.10

# Test MTU 9000
ping -M do -s 8972 10.0.20.10
```

Si le 4000 passe mais pas le 8972, le l2mtu n'est pas correctement configuré sur un des équipements.
