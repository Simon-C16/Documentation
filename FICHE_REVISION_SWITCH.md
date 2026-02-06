# FICHE RÉVISION SWITCH CISCO - RECTO

## 🔧 COMMANDES DE BASE
```bash
Switch> enable              # Mode privilégié
Switch# conf t              # Mode config
Switch(config)# hostname SW1
Switch# show run            # Config actuelle
Switch# wr                  # Sauvegarder (copy run start)
Switch# reload              # Redémarrer
Switch# erase startup-config  # Reset
Switch# delete flash:vlan.dat # Supprimer VLANs
```

## 📦 CRÉATION VLAN
```bash
Switch(config)# vlan 10
Switch(config-vlan)# name PRODUCTION
Switch(config-vlan)# exit

Switch(config)# vlan 20
Switch(config-vlan)# name COMMERCE
```

## 🔌 AFFECTATION PORTS → VLAN
```bash
# Port access (1 VLAN)
Switch(config)# int fa0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10
Switch(config-if)# no shutdown

# Plusieurs ports
Switch(config)# int range fa0/3-4
Switch(config-if)# switchport access vlan 20

# Ports inutilisés (BlackHole)
Switch(config)# int range fa0/5-24
Switch(config-if)# switchport access vlan 40
Switch(config-if)# shutdown
```

## 🌐 TRUNK (802.1q)
```bash
Switch(config)# int fa0/24
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk native vlan 99
Switch(config-if)# switchport trunk allowed vlan 10,20,30,99
Switch(config-if)# no shutdown
```

## 🔄 VTP (VLAN Trunking Protocol)
```bash
# SERVEUR
Switch(config)# vtp domain OSIVTP
Switch(config)# vtp mode server
Switch(config)# vtp password mdpvtp
Switch(config)# vtp version 2

# CLIENT
Switch(config)# vtp domain OSIVTP
Switch(config)# vtp mode client
Switch(config)# vtp password mdpvtp

# TRANSPARENT
Switch(config)# vtp mode transparent
```

## 🌍 IP DE GESTION
```bash
Switch(config)# int vlan 30
Switch(config-if)# ip address 192.168.30.254 255.255.255.0
Switch(config-if)# no shutdown
Switch(config-if)# exit
Switch(config)# ip default-gateway 192.168.30.1
```

## 🔒 SÉCURITÉ - MOTS DE PASSE
```bash
# Comptes utilisateurs
Switch(config)# username admin privilege 15 secret Rootsio2017
Switch(config)# username etudiant password Btssio2017
Switch(config)# enable secret Rootsio2017

# Chiffrer les mots de passe
Switch(config)# service password-encryption

# Console
Switch(config)# line con 0
Switch(config-line)# password Btssio2017
Switch(config-line)# login local
Switch(config-line)# exit

# Bannière
Switch(config)# banner motd #ACCES RESERVE#
```

---

# FICHE RÉVISION SWITCH CISCO - VERSO

## 🔐 SSH (Sécurisé)
```bash
# Configuration SSH
Switch(config)# ip domain-name bts-sio.local
Switch(config)# crypto key generate rsa general-keys modulus 1024
Switch(config)# ip ssh version 2
Switch(config)# ip ssh time-out 60
Switch(config)# ip ssh authentication-retries 3

# Lignes VTY
Switch(config)# line vty 0 15
Switch(config-line)# login local
Switch(config-line)# transport input ssh
Switch(config-line)# exit
```

## 🛡️ PORT SECURITY
```bash
Switch(config)# int fa0/1
Switch(config-if)# switchport port-security
Switch(config-if)# switchport port-security maximum 4
Switch(config-if)# switchport port-security mac-address sticky
Switch(config-if)# switchport port-security violation shutdown

# Réactiver port désactivé
Switch(config-if)# shutdown
Switch(config-if)# no shutdown
```

## 🚨 PROTECTIONS ATTAQUES

### DHCP Snooping
```bash
Switch(config)# ip dhcp snooping
Switch(config)# ip dhcp snooping vlan 10,20
Switch(config)# int range fa0/1-24
Switch(config-if)# ip dhcp snooping limit rate 4
Switch(config-if)# exit

# Port du serveur DHCP légitime
Switch(config)# int gig0/1
Switch(config-if)# ip dhcp snooping trust
```

### IP Source Guard
```bash
Switch(config)# int range fa0/1-24
Switch(config-if)# ip verify source port-security
```

### ARP Inspection
```bash
Switch(config)# ip arp inspection vlan 10
Switch(config)# int gig0/1
Switch(config-if)# ip arp inspection trust
```

## 🌳 SPANNING TREE
```bash
# Désactiver STP (DANGEREUX)
Switch(config)# no spanning-tree vlan 1

# Réactiver
Switch(config)# spanning-tree vlan 1
```

## 📊 VÉRIFICATIONS
```bash
Switch# show vlan brief           # VLANs
Switch# show vtp status           # VTP
Switch# show int trunk            # Trunks
Switch# show port-security        # Port Security
Switch# show ip dhcp snooping     # DHCP Snooping
Switch# show mac address-table    # Table MAC
Switch# show spanning-tree        # STP
Switch# show ip int brief         # Interfaces IP
```

## 🎯 TABLEAU RÉCAP VTP
| Mode | Peut créer VLAN | Propage | Reçoit |
|------|----------------|---------|--------|
| Server | ✅ | ✅ | ✅ |
| Client | ❌ | ✅ | ✅ |
| Transparent | ✅ (local) | ✅ (transit) | ❌ |

## 🔑 CODES TABLE ROUTAGE
| Code | Signification |
|------|---------------|
| C | Connected (directement connecté) |
| L | Local (IP interface) |
| S | Static (route statique) |

## ⚡ TIPS EXAMEN
- **TOUJOURS** `no shutdown` après config interface
- **VLAN natif** = 99 (sécurité)
- **BlackHole** = VLAN 40 + shutdown
- **Trunk** = Autorise plusieurs VLANs (802.1q)
- **Access** = 1 seul VLAN par port
- **VTP** nécessite trunks + même domaine + mot de passe
- **SSH > Telnet** (chiffré vs clair)
- **secret > password** (MD5 vs type 7)
