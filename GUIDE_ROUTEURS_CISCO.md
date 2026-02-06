# Guide Complet : Configuration des Routeurs Cisco

## 📚 Table des matières

1. [Introduction](#introduction)
2. [Commandes de base](#commandes-de-base)
3. [Configuration des interfaces](#configuration-des-interfaces)
4. [Routage statique](#routage-statique)
5. [Routage inter-VLAN](#routage-inter-vlan)
6. [Sous-interfaces (Router-on-a-Stick)](#sous-interfaces-router-on-a-stick)
7. [Serveur DHCP sur routeur](#serveur-dhcp-sur-routeur)
8. [Tables de routage](#tables-de-routage)
9. [Analyse du TTL et traceroute](#analyse-du-ttl-et-traceroute)
10. [Dépannage et vérification](#dépannage-et-vérification)

---

## Introduction

Ce guide couvre la configuration complète des routeurs Cisco (modèle 1941) pour le routage statique, le routage inter-VLAN, la configuration de sous-interfaces, et la mise en place de serveurs DHCP.

**Concepts clés :**
- **Routage** : Acheminement des paquets entre différents réseaux IP
- **Table de routage** : Liste des routes connues par le routeur
- **Route statique** : Route configurée manuellement (opposée à dynamique)
- **Sous-interface** : Interface virtuelle permettant de router plusieurs VLANs sur une seule interface physique
- **TTL** : Time To Live - compteur décrémenté à chaque passage par un routeur

---

## Commandes de base

### Accès au routeur

```bash
# Connexion via câble console depuis un PC
# Utiliser l'application "Terminal" sur le PC connecté via RS-232

# Mode utilisateur
Router>

# Passage en mode privilégié
Router> enable
Router#

# Passage en mode configuration globale
Router# configure terminal
Router(config)#

# Versions abrégées (IOS accepte les commandes tronquées)
Router> en
Router# conf t
```

### Navigation et aide

```bash
# Retour au mode précédent
Router(config)# exit

# Retour direct au mode privilégié
Router(config-if)# end

# Afficher l'aide
Router# show ?
Router# interface ?

# Auto-complétion avec TAB
Router# conf [TAB]  # Complète en "configure"
```

### Gestion de la configuration

```bash
# Afficher la configuration en cours (RAM)
Router# show running-config
Router# sh run  # Version abrégée

# Afficher la configuration de démarrage (NVRAM)
Router# show startup-config

# Sauvegarder la configuration
Router# copy running-config startup-config
Router# wr  # Version courte

# Effacer la configuration de démarrage
Router# erase startup-config

# Redémarrer le routeur
Router# reload
```

### Premier démarrage

À la première mise sous tension, le routeur peut proposer un assistant de configuration :

```
--- System Configuration Dialog ---

Would you like to enter the initial configuration dialog? [yes/no]:
```

**Réponse recommandée :** `no` (pour configurer manuellement)

---

## Configuration des interfaces

### Types d'interfaces

| Interface | Description |
|-----------|-------------|
| **FastEthernet (Fa)** | 100 Mbps |
| **GigabitEthernet (Gi, Gig)** | 1 Gbps |
| **Serial** | Liaison WAN série |

### Configuration d'une interface physique

```bash
# Passer en mode configuration de l'interface Gi0/0
Router(config)# interface gigabitethernet0/0
Router(config-if)# 

# Version abrégée
Router(config)# int gig0/0
Router(config-if)#
```

### Attribuer une adresse IP

```bash
Router(config)# interface gig0/0
Router(config-if)# ip address 172.16.1.254 255.255.0.0
Router(config-if)# no shutdown  # Activer l'interface
Router(config-if)# exit
```

**⚠️ IMPORTANT :** Par défaut, les interfaces d'un routeur sont **désactivées** (`shutdown`). Il faut toujours les activer avec `no shutdown`.

### Exemple de configuration complète

```bash
Router# configure terminal

# Interface Gi0/0 - Réseau 172.16.0.0/16
Router(config)# interface gigabitethernet0/0
Router(config-if)# ip address 172.16.1.254 255.255.0.0
Router(config-if)# no shutdown
Router(config-if)# exit

# Interface Gi0/1 - Réseau 172.17.0.0/16
Router(config)# interface gigabitethernet0/1
Router(config-if)# ip address 172.17.2.254 255.255.0.0
Router(config-if)# no shutdown
Router(config-if)# exit

Router(config)# exit
Router#
```

### Vérification des interfaces

```bash
# Afficher toutes les interfaces
Router# show ip interface brief

# Exemple de sortie :
# Interface              IP-Address      OK? Method Status                Protocol
# GigabitEthernet0/0     172.16.1.254    YES manual up                    up
# GigabitEthernet0/1     172.17.2.254    YES manual up                    up
# GigabitEthernet0/2     unassigned      YES unset  administratively down down

# Détails d'une interface
Router# show interface gig0/0

# Configuration IP d'une interface
Router# show ip interface gig0/0
```

### Désactiver une interface

```bash
Router(config)# interface gig0/2
Router(config-if)# shutdown
Router(config-if)# exit
```

---

## Routage statique

### Qu'est-ce qu'une route statique ?

Une **route statique** est une route configurée **manuellement** par l'administrateur. Elle indique au routeur comment atteindre un réseau distant via une passerelle (next hop).

### Syntaxe de base

```bash
Router(config)# ip route <réseau_destination> <masque> <passerelle>
```

### Exemple simple

**Topologie :**
```
[PC1: 172.16.1.1] ---- [Routeur-1] ---- [Routeur-2] ---- [PC4: 172.18.4.4]
                      172.17.2.254    172.17.3.254
```

**Sur Routeur-1 :**
```bash
Router1(config)# ip route 172.18.0.0 255.255.0.0 172.17.3.254
```

Cette route signifie : 
- **Destination** : Réseau 172.18.0.0/16
- **Passerelle** : Passer par le routeur 172.17.3.254 (Routeur-2)

**Sur Routeur-2 :**
```bash
Router2(config)# ip route 172.16.0.0 255.255.0.0 172.17.2.254
```

Cette route signifie :
- **Destination** : Réseau 172.16.0.0/16
- **Passerelle** : Passer par le routeur 172.17.2.254 (Routeur-1)

### Route par défaut

Une **route par défaut** (default route) capture tout le trafic dont la destination n'est pas connue.

```bash
# Route par défaut : 0.0.0.0/0 signifie "tout le reste"
Router(config)# ip route 0.0.0.0 0.0.0.0 184.10.20.254
```

Cette route envoie tout le trafic inconnu vers la passerelle `184.10.20.254`.

### Exemple complet avec 3 réseaux

**Topologie :**
```
[Réseau A]           [Réseau B]           [Réseau C]
172.16.0.0/16 ---- Routeur-1 ---- Routeur-2 ---- 172.18.0.0/16
  (Gi0/0)             (Gi0/1)       (Gi0/0)         (Gi0/1)
172.16.1.254       172.17.2.254  172.17.3.254    172.18.4.254
                   172.17.0.0/16
```

**Configuration Routeur-1 :**
```bash
# Interfaces
Router1(config)# interface gig0/0
Router1(config-if)# ip address 172.16.1.254 255.255.0.0
Router1(config-if)# no shutdown
Router1(config-if)# exit

Router1(config)# interface gig0/1
Router1(config-if)# ip address 172.17.2.254 255.255.0.0
Router1(config-if)# no shutdown
Router1(config-if)# exit

# Route statique vers le réseau 172.18.0.0/16
Router1(config)# ip route 172.18.0.0 255.255.0.0 172.17.3.254
Router1(config)# exit
```

**Configuration Routeur-2 :**
```bash
# Interfaces
Router2(config)# interface gig0/0
Router2(config-if)# ip address 172.17.3.254 255.255.0.0
Router2(config-if)# no shutdown
Router2(config-if)# exit

Router2(config)# interface gig0/1
Router2(config-if)# ip address 172.18.4.254 255.255.0.0
Router2(config-if)# no shutdown
Router2(config-if)# exit

# Route statique vers le réseau 172.16.0.0/16
Router2(config)# ip route 172.16.0.0 255.255.0.0 172.17.2.254
Router2(config)# exit
```

### Vérification des routes

```bash
# Afficher la table de routage
Router# show ip route

# Exemple de sortie :
# C    172.16.0.0/16 is directly connected, GigabitEthernet0/0
# L    172.16.1.254/32 is directly connected, GigabitEthernet0/0
# C    172.17.0.0/16 is directly connected, GigabitEthernet0/1
# L    172.17.2.254/32 is directly connected, GigabitEthernet0/1
# S    172.18.0.0/16 [1/0] via 172.17.3.254
```

**Légende :**
- **C** : Connecté directement (Directly Connected)
- **L** : Local (adresse IP de l'interface)
- **S** : Route statique (Static)
- **[1/0]** : Distance administrative / Métrique

---

## Routage inter-VLAN

### Problématique

Les **VLANs** sont des réseaux IP distincts. Par défaut, ils ne peuvent **pas communiquer** entre eux, même s'ils sont sur le même switch.

**Solution :** Utiliser un **routeur** pour faire communiquer les VLANs.

### Méthode traditionnelle (1 interface physique par VLAN)

**Limite :** Le nombre de VLANs est limité par le nombre d'interfaces physiques du routeur.

**Topologie :**
```
[VLAN 10] ---- Routeur Gi0/0
[VLAN 20] ---- Routeur Gi0/1
[VLAN 30] ---- Routeur Gi0/2
```

Cette méthode n'est **pas recommandée** pour les réseaux avec beaucoup de VLANs.

---

## Sous-interfaces (Router-on-a-Stick)

### Qu'est-ce qu'une sous-interface ?

Une **sous-interface** est une **interface virtuelle** créée à partir d'une interface physique. Elle permet de router **plusieurs VLANs** sur une **seule interface physique**.

### Principe du Router-on-a-Stick

**Topologie :**
```
[Switch avec VLANs 10, 20, 30]
          |
       (Trunk)
          |
    [Routeur Gi0/0]
    /      |      \
Gi0/0.10 Gi0/0.20 Gi0/0.30
```

Le routeur utilise **une seule interface physique** (Gi0/0) connectée au switch via un **lien trunk**. Sur cette interface, plusieurs **sous-interfaces** sont créées, chacune correspondant à un VLAN.

### Configuration des sous-interfaces

#### Étape 1 : Activer l'interface physique

```bash
Router(config)# interface gig0/0
Router(config-if)# no shutdown  # CRUCIAL : Activer l'interface physique
Router(config-if)# exit
```

**⚠️ IMPORTANT :** L'activation de l'interface physique active automatiquement toutes les sous-interfaces.

#### Étape 2 : Créer les sous-interfaces

```bash
# Sous-interface pour le VLAN 10
Router(config)# interface gig0/0.10
Router(config-subif)# encapsulation dot1q 10  # Associer au VLAN 10
Router(config-subif)# ip address 192.168.10.254 255.255.255.0
Router(config-subif)# exit

# Sous-interface pour le VLAN 20
Router(config)# interface gig0/0.20
Router(config-subif)# encapsulation dot1q 20  # Associer au VLAN 20
Router(config-subif)# ip address 192.168.20.254 255.255.255.0
Router(config-subif)# exit

# Sous-interface pour le VLAN 30
Router(config)# interface gig0/0.30
Router(config-subif)# encapsulation dot1q 30  # Associer au VLAN 30
Router(config-subif)# ip address 192.168.30.254 255.255.255.0
Router(config-subif)# exit
```

**Syntaxe :**
- **gig0/0.10** : Interface physique `.` numéro de sous-interface
- **encapsulation dot1q 10** : Active l'encapsulation 802.1q pour le VLAN 10
- Le numéro de sous-interface n'a pas besoin de correspondre au numéro de VLAN, mais c'est une bonne pratique

#### Étape 3 : Configurer le switch en trunk

Sur le switch, le port connecté au routeur doit être en **mode trunk** :

```bash
Switch(config)# interface gig0/1
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 10,20,30
Switch(config-if)# no shutdown
Switch(config-if)# exit
```

### Exemple complet de configuration

**Topologie :**
```
[PC1: VLAN 10, 192.168.10.1] ─┐
[PC2: VLAN 20, 192.168.20.2] ─┼─ Switch (Trunk) ─ Routeur R1
[PC3: VLAN 30, 192.168.30.3] ─┘
```

**Configuration Switch :**
```bash
# VLANs
Switch(config)# vlan 10
Switch(config-vlan)# name DME
Switch(config-vlan)# exit
Switch(config)# vlan 20
Switch(config-vlan)# name BIO
Switch(config-vlan)# exit
Switch(config)# vlan 30
Switch(config-vlan)# name ADMIN
Switch(config-vlan)# exit

# Affectation des ports
Switch(config)# interface fa0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10
Switch(config-if)# exit

Switch(config)# interface fa0/2
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 20
Switch(config-if)# exit

Switch(config)# interface fa0/3
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 30
Switch(config-if)# exit

# Trunk vers le routeur
Switch(config)# interface gig0/1
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 10,20,30
Switch(config-if)# no shutdown
Switch(config-if)# exit
```

**Configuration Routeur :**
```bash
# Activer l'interface physique
Router(config)# interface gig0/0
Router(config-if)# no shutdown
Router(config-if)# exit

# Sous-interfaces
Router(config)# interface gig0/0.10
Router(config-subif)# encapsulation dot1q 10
Router(config-subif)# ip address 192.168.10.254 255.255.255.0
Router(config-subif)# exit

Router(config)# interface gig0/0.20
Router(config-subif)# encapsulation dot1q 20
Router(config-subif)# ip address 192.168.20.254 255.255.255.0
Router(config-subif)# exit

Router(config)# interface gig0/0.30
Router(config-subif)# encapsulation dot1q 30
Router(config-subif)# ip address 192.168.30.254 255.255.255.0
Router(config-subif)# exit

Router(config)# exit
Router#
```

**Configuration des PCs :**
- **PC1** (VLAN 10) : IP `192.168.10.1/24`, Passerelle `192.168.10.254`
- **PC2** (VLAN 20) : IP `192.168.20.2/24`, Passerelle `192.168.20.254`
- **PC3** (VLAN 30) : IP `192.168.30.3/24`, Passerelle `192.168.30.254`

### Vérification du routage inter-VLAN

```bash
# Afficher les sous-interfaces VLAN
Router# show vlans

# Afficher la table de routage
Router# show ip route

# Exemple de sortie :
# C    192.168.10.0/24 is directly connected, GigabitEthernet0/0.10
# L    192.168.10.254/32 is directly connected, GigabitEthernet0/0.10
# C    192.168.20.0/24 is directly connected, GigabitEthernet0/0.20
# L    192.168.20.254/32 is directly connected, GigabitEthernet0/0.20
# C    192.168.30.0/24 is directly connected, GigabitEthernet0/0.30
# L    192.168.30.254/32 is directly connected, GigabitEthernet0/0.30
```

### Test de communication

Depuis PC1 (VLAN 10), faire un ping vers PC2 (VLAN 20) :

```
PC1> ping 192.168.20.2

Reply from 192.168.20.2: bytes=32 time<1ms TTL=127
```

**Observation :** Le TTL est décrémenté car le paquet a traversé le routeur.

---

## Serveur DHCP sur routeur

### Qu'est-ce que le DHCP ?

Le **DHCP** (Dynamic Host Configuration Protocol) attribue automatiquement des adresses IP aux clients du réseau.

### Configuration d'un pool DHCP

```bash
# Créer un pool DHCP nommé "CLIENT_LAN"
Router(config)# ip dhcp pool CLIENT_LAN

# Définir le réseau
Router(dhcp-config)# network 192.168.10.0 255.255.255.0

# Définir le serveur DNS
Router(dhcp-config)# dns-server 172.17.1.1

# Définir la passerelle par défaut
Router(dhcp-config)# default-router 192.168.10.254

Router(dhcp-config)# exit
```

### Exclure des adresses IP

Certaines adresses IP doivent être réservées (passerelle, serveurs, imprimantes).

```bash
# Exclure l'adresse du routeur
Router(config)# ip dhcp excluded-address 192.168.10.254

# Exclure une plage d'adresses
Router(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.10
```

### Exemple complet de configuration DHCP

```bash
# Pool DHCP pour le VLAN 10
Router(config)# ip dhcp pool VLAN10_POOL
Router(dhcp-config)# network 192.168.10.0 255.255.255.0
Router(dhcp-config)# default-router 192.168.10.254
Router(dhcp-config)# dns-server 8.8.8.8
Router(dhcp-config)# exit

# Exclure l'adresse de la passerelle
Router(config)# ip dhcp excluded-address 192.168.10.254

# Pool DHCP pour le VLAN 20
Router(config)# ip dhcp pool VLAN20_POOL
Router(dhcp-config)# network 192.168.20.0 255.255.255.0
Router(dhcp-config)# default-router 192.168.20.254
Router(dhcp-config)# dns-server 8.8.8.8
Router(dhcp-config)# exit

Router(config)# ip dhcp excluded-address 192.168.20.254
```

### Vérification du DHCP

```bash
# Afficher la configuration DHCP
Router# show ip dhcp pool

# Afficher les baux DHCP (adresses attribuées)
Router# show ip dhcp binding

# Exemple de sortie :
# IP address       Client-ID/Hardware address       Lease expiration        Type
# 192.168.10.2     0100.5079.6668.00                Mar 02 2025 12:00 AM    Automatic
```

---

## Tables de routage

### Comprendre la table de routage

La **table de routage** liste toutes les routes connues par le routeur.

```bash
Router# show ip route

# Exemple de sortie :
# Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
#        D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
#
# C    172.16.0.0/16 is directly connected, GigabitEthernet0/0
# L    172.16.1.254/32 is directly connected, GigabitEthernet0/0
# C    172.17.0.0/16 is directly connected, GigabitEthernet0/1
# L    172.17.2.254/32 is directly connected, GigabitEthernet0/1
# S    172.18.0.0/16 [1/0] via 172.17.3.254
# S*   0.0.0.0/0 [1/0] via 184.10.20.254
```

### Détails des codes

| Code | Type de route | Description |
|------|---------------|-------------|
| **C** | Connected | Réseau directement connecté |
| **L** | Local | Adresse IP de l'interface locale |
| **S** | Static | Route statique configurée manuellement |
| **S*** | Default route | Route par défaut (statique) |
| **R** | RIP | Route apprise via RIP |
| **O** | OSPF | Route apprise via OSPF |
| **D** | EIGRP | Route apprise via EIGRP |

### Distance administrative et métrique

**Format :** `[Distance administrative / Métrique]`

**Exemple :** `S    172.18.0.0/16 [1/0] via 172.17.3.254`
- **1** : Distance administrative (pour les routes statiques)
- **0** : Métrique (coût)

**Distance administrative** (plus c'est bas, plus la route est fiable) :
- **0** : Interface directement connectée
- **1** : Route statique
- **90** : EIGRP
- **110** : OSPF
- **120** : RIP

---

## Analyse du TTL et traceroute

### Qu'est-ce que le TTL ?

Le **TTL** (Time To Live) est un compteur dans l'en-tête IP qui est **décrémenté de 1** à chaque passage par un routeur.

- Objectif : Éviter que les paquets tournent en boucle indéfiniment
- Valeur initiale : 128 (Windows) ou 64 (Linux/Mac)
- Quand TTL atteint 0 : Le paquet est détruit

### Exemple d'analyse du TTL

**Ping depuis PC1 vers un serveur :**

```
PC1> ping 192.168.20.10

Reply from 192.168.20.10: bytes=32 time<1ms TTL=123
```

**Analyse :**
- Valeur initiale du TTL : 128 (Windows)
- Valeur reçue : 123
- Nombre de routeurs traversés : 128 - 123 = **5 routeurs**

**⚠️ Important :** Le TTL affiché correspond au paquet de **retour** (echo reply), pas à l'aller.

### Commande traceroute

La commande `traceroute` (ou `tracert` sur Windows) affiche tous les routeurs traversés.

```bash
# Sur un routeur Cisco
Router# traceroute 192.168.20.10

# Sur un PC Windows
PC> tracert 192.168.20.10

# Sur un PC Linux/Mac
PC> traceroute 192.168.20.10
```

**Exemple de sortie :**
```
Tracing route to 192.168.20.10 over a maximum of 30 hops:

  1    <1 ms     <1 ms     <1 ms  192.168.10.254
  2     1 ms      1 ms      1 ms  172.17.3.254
  3     1 ms      1 ms      1 ms  192.168.20.10

Trace complete.
```

**Interprétation :**
- Le paquet a traversé **2 routeurs** (hops 1 et 2) avant d'atteindre la destination (hop 3)

### TTL et nombre de sauts

**Relation :** TTL affiché = TTL initial - Nombre de routeurs traversés

**Exemple :**
```
Ping reply with TTL=126
TTL initial (Windows) = 128
Nombre de routeurs = 128 - 126 = 2
```

---

## Dépannage et vérification

### Commandes essentielles

```bash
# Afficher la table de routage
Router# show ip route

# Afficher les interfaces
Router# show ip interface brief

# Détails d'une interface
Router# show interface gig0/0

# Afficher les VLANs (sous-interfaces)
Router# show vlans

# Afficher la configuration en cours
Router# show running-config

# Afficher la configuration d'une interface
Router# show running-config interface gig0/0

# Tester la connectivité
Router# ping 192.168.10.1

# Tracer le chemin réseau
Router# traceroute 192.168.10.1

# Afficher les statistiques DHCP
Router# show ip dhcp binding
Router# show ip dhcp pool
```

### Diagnostic des problèmes courants

#### Problème : Le ping ne fonctionne pas

**Vérifications :**

1. **L'interface est-elle active ?**
```bash
Router# show ip interface brief
# Vérifier que Status = up et Protocol = up
```

2. **La route existe-t-elle ?**
```bash
Router# show ip route
# Vérifier qu'il existe une route vers la destination
```

3. **La passerelle est-elle correcte sur le PC ?**
```bash
PC> ipconfig
# Vérifier que la passerelle par défaut est bien l'adresse du routeur
```

#### Problème : Interface "administratively down"

**Solution :** Activer l'interface
```bash
Router(config)# interface gig0/0
Router(config-if)# no shutdown
Router(config-if)# exit
```

#### Problème : Routage inter-VLAN ne fonctionne pas

**Vérifications :**

1. **Le trunk est-il configuré sur le switch ?**
```bash
Switch# show interfaces trunk
```

2. **Les sous-interfaces sont-elles correctement configurées ?**
```bash
Router# show vlans
```

3. **L'encapsulation dot1q est-elle activée ?**
```bash
Router# show running-config interface gig0/0.10
# Vérifier la présence de "encapsulation dot1q 10"
```

### Commandes de debug (à utiliser avec précaution)

```bash
# Activer le debug IP routing
Router# debug ip routing

# Afficher les paquets ICMP
Router# debug ip icmp

# Désactiver tous les debugs
Router# undebug all
```

**⚠️ Attention :** Les commandes debug génèrent beaucoup de messages et peuvent ralentir le routeur. À utiliser uniquement en dépannage.

---

## Récapitulatif des commandes clés

| Action | Commande |
|--------|----------|
| Configurer une interface | `interface gig0/0` puis `ip address x.x.x.x y.y.y.y` |
| Activer une interface | `no shutdown` |
| Créer une sous-interface | `interface gig0/0.10` |
| Encapsulation 802.1q | `encapsulation dot1q 10` |
| Ajouter une route statique | `ip route x.x.x.x y.y.y.y z.z.z.z` |
| Route par défaut | `ip route 0.0.0.0 0.0.0.0 w.w.w.w` |
| Créer un pool DHCP | `ip dhcp pool NOM` |
| Réseau DHCP | `network x.x.x.x y.y.y.y` |
| Passerelle DHCP | `default-router x.x.x.x` |
| DNS DHCP | `dns-server x.x.x.x` |
| Exclure des IPs DHCP | `ip dhcp excluded-address x.x.x.x` |
| Afficher la table de routage | `show ip route` |
| Afficher les interfaces | `show ip interface brief` |
| Sauvegarder | `copy running-config startup-config` |

---

## 🎯 Bonnes pratiques

1. **Toujours activer les interfaces physiques** avec `no shutdown`
2. **Numéroter les sous-interfaces** selon le numéro de VLAN (ex: Gi0/0.10 pour VLAN 10)
3. **Documenter les routes statiques** dans un fichier texte
4. **Exclure les adresses critiques** du pool DHCP (routeurs, serveurs, imprimantes)
5. **Utiliser des noms de pool DHCP explicites** (ex: VLAN10_PRODUCTION)
6. **Vérifier la table de routage** après chaque modification
7. **Tester la connectivité** avec ping et traceroute
8. **Sauvegarder régulièrement** la configuration

---

## 📝 Exemple de configuration complète

Voici un exemple complet de routeur avec routage inter-VLAN et DHCP :

```bash
# Configuration de base
Router> enable
Router# configure terminal
Router(config)# hostname R1

# Interface physique vers le réseau externe
Router(config)# interface gig0/2
Router(config-if)# ip address 172.17.1.5 255.255.0.0
Router(config-if)# no shutdown
Router(config-if)# exit

# Interface physique pour les VLANs (trunk)
Router(config)# interface gig0/0
Router(config-if)# no shutdown
Router(config-if)# exit

# Sous-interface VLAN 10 - DME
Router(config)# interface gig0/0.10
Router(config-subif)# encapsulation dot1q 10
Router(config-subif)# ip address 192.168.10.254 255.255.255.0
Router(config-subif)# exit

# Sous-interface VLAN 20 - BIO
Router(config)# interface gig0/0.20
Router(config-subif)# encapsulation dot1q 20
Router(config-subif)# ip address 192.168.20.254 255.255.255.0
Router(config-subif)# exit

# Sous-interface VLAN 30 - ADMIN
Router(config)# interface gig0/0.30
Router(config-subif)# encapsulation dot1q 30
Router(config-subif)# ip address 192.168.30.254 255.255.255.0
Router(config-subif)# exit

# Pool DHCP pour VLAN 10
Router(config)# ip dhcp pool VLAN10_DME
Router(dhcp-config)# network 192.168.10.0 255.255.255.0
Router(dhcp-config)# default-router 192.168.10.254
Router(dhcp-config)# dns-server 172.17.1.1
Router(dhcp-config)# exit
Router(config)# ip dhcp excluded-address 192.168.10.254

# Pool DHCP pour VLAN 20
Router(config)# ip dhcp pool VLAN20_BIO
Router(dhcp-config)# network 192.168.20.0 255.255.255.0
Router(dhcp-config)# default-router 192.168.20.254
Router(dhcp-config)# dns-server 172.17.1.1
Router(dhcp-config)# exit
Router(config)# ip dhcp excluded-address 192.168.20.254

# Pool DHCP pour VLAN 30
Router(config)# ip dhcp pool VLAN30_ADMIN
Router(dhcp-config)# network 192.168.30.0 255.255.255.0
Router(dhcp-config)# default-router 192.168.30.254
Router(dhcp-config)# dns-server 172.17.1.1
Router(dhcp-config)# exit
Router(config)# ip dhcp excluded-address 192.168.30.254

# Routes statiques (si nécessaire)
Router(config)# ip route 172.17.0.0 255.255.0.0 172.17.1.1

# Sauvegarder
Router(config)# exit
Router# copy running-config startup-config
```

---

**Fin du guide Routeurs Cisco** 🎓
