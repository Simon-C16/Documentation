# FICHE RÉVISION ROUTEUR CISCO - RECTO

## 🔧 COMMANDES DE BASE
```bash
Router> enable              # Mode privilégié
Router# conf t              # Mode config
Router(config)# hostname R1
Router# show run            # Config actuelle
Router# wr                  # Sauvegarder
Router# reload              # Redémarrer
Router# show ip route       # Table de routage
```

## 🌐 CONFIGURATION INTERFACE
```bash
Router(config)# int gig0/0
Router(config-if)# ip address 172.16.1.254 255.255.0.0
Router(config-if)# no shutdown   # ⚠️ IMPORTANT !
Router(config-if)# exit

Router(config)# int gig0/1
Router(config-if)# ip address 172.17.2.254 255.255.0.0
Router(config-if)# no shutdown
```

## 📍 ROUTAGE STATIQUE
```bash
# Route vers réseau distant
Router(config)# ip route 172.18.0.0 255.255.0.0 172.17.3.254
#                       └─ Réseau dest  └─ Masque  └─ Passerelle

# Route par défaut (tout le reste)
Router(config)# ip route 0.0.0.0 0.0.0.0 184.10.20.254
```

## 🔀 SOUS-INTERFACES (Router-on-a-Stick)
```bash
# Activer interface physique
Router(config)# int gig0/0
Router(config-if)# no shutdown
Router(config-if)# exit

# Créer sous-interfaces pour chaque VLAN
Router(config)# int gig0/0.10
Router(config-subif)# encapsulation dot1q 10
Router(config-subif)# ip address 192.168.10.254 255.255.255.0
Router(config-subif)# exit

Router(config)# int gig0/0.20
Router(config-subif)# encapsulation dot1q 20
Router(config-subif)# ip address 192.168.20.254 255.255.255.0
Router(config-subif)# exit

Router(config)# int gig0/0.30
Router(config-subif)# encapsulation dot1q 30
Router(config-subif)# ip address 192.168.30.254 255.255.255.0
```

## 📡 SERVEUR DHCP
```bash
# Créer pool DHCP
Router(config)# ip dhcp pool CLIENT_LAN
Router(dhcp-config)# network 192.168.10.0 255.255.255.0
Router(dhcp-config)# default-router 192.168.10.254
Router(dhcp-config)# dns-server 8.8.8.8
Router(dhcp-config)# exit

# Exclure adresses (passerelle, serveurs)
Router(config)# ip dhcp excluded-address 192.168.10.254
Router(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.10
```

## 📊 TABLE DE ROUTAGE
```bash
Router# show ip route

# Exemple sortie :
# C    172.16.0.0/16 is directly connected, Gig0/0
# L    172.16.1.254/32 is directly connected, Gig0/0
# S    172.18.0.0/16 [1/0] via 172.17.3.254
# S*   0.0.0.0/0 [1/0] via 184.10.20.254
```

## 🔑 CODES ROUTES
| Code | Type | Description |
|------|------|-------------|
| **C** | Connected | Réseau directement connecté |
| **L** | Local | Adresse IP de l'interface |
| **S** | Static | Route statique manuelle |
| **S*** | Default | Route par défaut |
| **R** | RIP | Route dynamique RIP |
| **O** | OSPF | Route dynamique OSPF |

---

# FICHE RÉVISION ROUTEUR CISCO - VERSO

## 🎯 EXEMPLE COMPLET - ROUTAGE INTER-VLAN

### Topologie
```
[PC1 VLAN10] ─┐
[PC2 VLAN20] ─┼─ Switch (Trunk) ─ Routeur R1
[PC3 VLAN30] ─┘
```

### Configuration Switch
```bash
# VLANs
Switch(config)# vlan 10
Switch(config-vlan)# name DME
Switch(config)# vlan 20
Switch(config-vlan)# name BIO
Switch(config)# vlan 30
Switch(config-vlan)# name ADMIN

# Ports access
Switch(config)# int fa0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10

# Trunk vers routeur
Switch(config)# int gig0/1
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 10,20,30
Switch(config-if)# no shutdown
```

### Configuration Routeur
```bash
# Interface physique
Router(config)# int gig0/0
Router(config-if)# no shutdown
Router(config-if)# exit

# Sous-interfaces
Router(config)# int gig0/0.10
Router(config-subif)# encapsulation dot1q 10
Router(config-subif)# ip address 192.168.10.254 255.255.255.0

Router(config)# int gig0/0.20
Router(config-subif)# encapsulation dot1q 20
Router(config-subif)# ip address 192.168.20.254 255.255.255.0

Router(config)# int gig0/0.30
Router(config-subif)# encapsulation dot1q 30
Router(config-subif)# ip address 192.168.30.254 255.255.255.0

# DHCP
Router(config)# ip dhcp pool VLAN10
Router(dhcp-config)# network 192.168.10.0 255.255.255.0
Router(dhcp-config)# default-router 192.168.10.254
Router(dhcp-config)# dns-server 8.8.8.8
Router(dhcp-config)# exit
Router(config)# ip dhcp excluded-address 192.168.10.254
```

## 🔍 TTL (Time To Live)
```
TTL = Compteur décrémenté à chaque routeur

Exemple ping :
Reply TTL=123
→ TTL initial (Windows) = 128
→ Routeurs traversés = 128 - 123 = 5

⚠️ TTL affiché = paquet de RETOUR (echo reply)
```

## 🛤️ TRACEROUTE
```bash
Router# traceroute 192.168.20.10
PC> tracert 192.168.20.10

# Affiche tous les routeurs traversés
```

## 📋 VÉRIFICATIONS
```bash
Router# show ip interface brief   # État interfaces
Router# show ip route              # Table routage
Router# show vlans                 # Sous-interfaces VLAN
Router# show interface gig0/0      # Détails interface
Router# show ip dhcp binding       # Baux DHCP
Router# show ip dhcp pool          # Pools DHCP
Router# ping 192.168.10.1          # Test connectivité
```

## 🔧 DÉPANNAGE

### Interface "down" ?
```bash
Router# show ip int brief
# Vérifier : Status = up, Protocol = up

# Si "administratively down"
Router(config)# int gig0/0
Router(config-if)# no shutdown
```

### Pas de route ?
```bash
Router# show ip route
# Vérifier qu'il existe une route vers destination

# Ajouter route si manquante
Router(config)# ip route 172.18.0.0 255.255.0.0 172.17.3.254
```

### Inter-VLAN ne marche pas ?
```bash
# 1. Vérifier trunk sur switch
Switch# show int trunk

# 2. Vérifier sous-interfaces routeur
Router# show vlans

# 3. Vérifier encapsulation dot1q
Router# show run int gig0/0.10
```

## ⚡ TIPS EXAMEN

### Toujours faire :
- ✅ `no shutdown` sur interface physique ET logique
- ✅ Vérifier passerelle sur PCs = IP du routeur
- ✅ Exclure IP routeur du pool DHCP
- ✅ Numéro sous-interface = numéro VLAN (bonne pratique)

### Ordre config Router-on-a-Stick :
1. Activer interface physique (`no shutdown`)
2. Créer sous-interfaces (`.10`, `.20`, `.30`)
3. `encapsulation dot1q X` sur chaque sous-interface
4. Attribuer IP à chaque sous-interface
5. Switch en trunk avec VLANs autorisés

### Distance Administrative
| Type | Distance |
|------|----------|
| Connected | 0 |
| Static | 1 |
| EIGRP | 90 |
| OSPF | 110 |
| RIP | 120 |

*Plus c'est bas, plus c'est prioritaire*

## 📐 FORMULES IMPORTANTES
```
Nombre de routeurs = TTL initial - TTL reçu

Sous-réseau valide ? → Vérifier masque

Passerelle = Dernière IP utilisable du réseau
(Sauf si spécifié autrement)
```

## 🎯 ERREURS FRÉQUENTES
❌ Oublier `no shutdown`
❌ Oublier `encapsulation dot1q`
❌ Mauvaise passerelle sur PC
❌ VLANs non autorisés sur trunk
❌ Route statique manquante
❌ Pool DHCP sans exclusion
