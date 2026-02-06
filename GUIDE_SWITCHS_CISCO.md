# Guide Complet : Configuration des Switchs Cisco

## 📚 Table des matières

1. [Introduction](#introduction)
2. [Commandes de base](#commandes-de-base)
3. [Configuration des VLANs](#configuration-des-vlans)
4. [Configuration du Trunking (802.1q)](#configuration-du-trunking-8021q)
5. [Protocole VTP (VLAN Trunking Protocol)](#protocole-vtp-vlan-trunking-protocol)
6. [Sécurisation des Switchs](#sécurisation-des-switchs)
7. [Port Security](#port-security)
8. [Protections contre les attaques](#protections-contre-les-attaques)
9. [Spanning Tree Protocol](#spanning-tree-protocol)
10. [Dépannage et vérification](#dépannage-et-vérification)

---

## Introduction

Ce guide regroupe l'ensemble des commandes et concepts pour configurer et sécuriser des switchs Cisco (modèle 2960). Il couvre les VLANs, le trunking, le VTP, la sécurité des accès et les protections contre les attaques réseau.

**Concepts clés :**
- **VLAN** : Virtual Local Area Network - segmentation logique du réseau
- **Trunk** : Lien transportant plusieurs VLANs (étiquetage 802.1q)
- **VTP** : Protocole de propagation automatique des VLANs
- **Port Security** : Limitation des adresses MAC par port

---

## Commandes de base

### Accès au switch

```bash
# Connexion en mode console depuis un PC
# Utiliser l'application "Terminal" sur le PC connecté via câble console (RS-232)

# Mode utilisateur
Switch>

# Passage en mode privilégié
Switch> enable
Switch#

# Passage en mode configuration globale
Switch# configure terminal
Switch(config)#
```

### Navigation et aide

```bash
# Retour au mode précédent
Switch(config)# exit

# Afficher l'aide sur une commande
Switch# show ?

# Auto-complétion avec la touche TAB
Switch# conf [TAB]  # Complète en "configure"
```

### Gestion de la configuration

```bash
# Afficher la configuration en cours (RAM)
Switch# show running-config

# Afficher la configuration de démarrage (NVRAM)
Switch# show startup-config

# Sauvegarder la configuration en cours vers la NVRAM
Switch# copy running-config startup-config
# ou version courte :
Switch# wr

# Effacer la configuration de démarrage
Switch# erase startup-config

# Redémarrer le switch
Switch# reload
```

### Réinitialisation complète du switch

```bash
Switch> enable
Switch# erase startup-config
Switch# delete flash:vlan.dat
Switch# delete flash:config.text
Switch# reload
```

---

## Configuration des VLANs

### Qu'est-ce qu'un VLAN ?

Un VLAN (Virtual LAN) permet de segmenter un réseau physique en plusieurs réseaux logiques. Chaque VLAN est isolé des autres, ce qui améliore la sécurité et réduit les broadcasts.

### Création de VLANs

```bash
Switch# configure terminal

# Créer un VLAN
Switch(config)# vlan 10
Switch(config-vlan)# name PRODUCTION
Switch(config-vlan)# exit

Switch(config)# vlan 20
Switch(config-vlan)# name COMMERCE
Switch(config-vlan)# exit

Switch(config)# vlan 30
Switch(config-vlan)# name GESTION
Switch(config-vlan)# exit

# VLAN pour les ports inutilisés (sécurité)
Switch(config)# vlan 40
Switch(config-vlan)# name BlackHole
Switch(config-vlan)# exit

# VLAN natif (pour le trunking)
Switch(config)# vlan 99
Switch(config-vlan)# name Native
Switch(config-vlan)# exit
```

### Affectation de ports aux VLANs

```bash
# Affecter un port au VLAN 10
Switch(config)# interface fa0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10
Switch(config-if)# no shutdown
Switch(config-if)# exit

# Affecter plusieurs ports au VLAN 20
Switch(config)# interface range fa0/3-4
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 20
Switch(config-if)# no shutdown
Switch(config-if)# exit

# Mettre tous les ports inutilisés dans le VLAN BlackHole et les éteindre
Switch(config)# interface range fa0/5-24
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 40
Switch(config-if)# shutdown
Switch(config-if)# exit
```

### Configuration de l'adresse IP du switch (pour la gestion)

```bash
# Le switch a besoin d'une IP pour être administré à distance
Switch(config)# interface vlan 30
Switch(config-if)# ip address 192.168.30.254 255.255.255.0
Switch(config-if)# no shutdown
Switch(config-if)# exit

# Définir la passerelle par défaut
Switch(config)# ip default-gateway 192.168.30.1

# Définir le serveur DNS (optionnel)
Switch(config)# ip name-server 172.17.0.1
```

### Vérification des VLANs

```bash
# Afficher tous les VLANs configurés
Switch# show vlan

# Afficher un VLAN spécifique
Switch# show vlan id 10

# Afficher les VLANs en bref
Switch# show vlan brief
```

---

## Configuration du Trunking (802.1q)

### Qu'est-ce qu'un Trunk ?

Un **trunk** est un lien entre deux équipements (généralement deux switchs) configuré pour transporter plusieurs VLANs. Les trames sont **étiquetées** (tagged) selon la norme **802.1q** avec l'ID du VLAN.

### Modes de port

| Mode | Description |
|------|-------------|
| **access** | Port pour un PC/serveur, appartient à un seul VLAN |
| **trunk** | Force le mode trunk, transporte plusieurs VLANs |
| **dynamic auto** | Négocie le mode trunk (passif) |
| **dynamic desirable** | Négocie le mode trunk (actif) |

### Configuration d'un port trunk

```bash
# Configurer le port Fa0/24 en trunk
Switch(config)# interface fa0/24
Switch(config-if)# switchport mode trunk

# Définir le VLAN natif (important pour la sécurité)
Switch(config-if)# switchport trunk native vlan 99

# Spécifier les VLANs autorisés sur le trunk
Switch(config-if)# switchport trunk allowed vlan 10,20,30,99

# Activer le port
Switch(config-if)# no shutdown
Switch(config-if)# exit
```

**⚠️ VLAN natif :** Les trames non étiquetées sur un trunk sont assignées au VLAN natif. Pour la sécurité, utilisez un VLAN dédié (ex: VLAN 99) qui n'est utilisé nulle part ailleurs.

### Gestion des VLANs autorisés sur un trunk

```bash
# Ajouter des VLANs à la liste autorisée
Switch(config-if)# switchport trunk allowed vlan add 40,50

# Retirer des VLANs de la liste
Switch(config-if)# switchport trunk allowed vlan remove 50

# Autoriser tous les VLANs
Switch(config-if)# switchport trunk allowed vlan all

# N'autoriser aucun VLAN (puis ajouter manuellement)
Switch(config-if)# switchport trunk allowed vlan none
```

### Configuration trunk entre switchs

**Exemple : Switch1 ↔ Switch2**

```bash
# Sur Switch1
Switch1(config)# interface gig0/1
Switch1(config-if)# switchport mode trunk
Switch1(config-if)# switchport trunk native vlan 99
Switch1(config-if)# switchport trunk allowed vlan 10,20,30,99
Switch1(config-if)# no shutdown
Switch1(config-if)# exit

# Sur Switch2 (même configuration)
Switch2(config)# interface gig0/1
Switch2(config-if)# switchport mode trunk
Switch2(config-if)# switchport trunk native vlan 99
Switch2(config-if)# switchport trunk allowed vlan 10,20,30,99
Switch2(config-if)# no shutdown
Switch2(config-if)# exit
```

### Vérification du trunking

```bash
# Afficher l'état des trunks
Switch# show interfaces trunk

# Exemple de sortie :
# Port        Mode         Encapsulation  Status        Native vlan
# Fa0/24      on           802.1q         trunking      99
#
# Port        Vlans allowed on trunk
# Fa0/24      10,20,30,99
#
# Port        Vlans allowed and active in management domain
# Fa0/24      10,20,30,99

# Afficher les détails d'une interface
Switch# show interface fa0/24 switchport
```

---

## Protocole VTP (VLAN Trunking Protocol)

### Qu'est-ce que le VTP ?

Le **VTP** (VLAN Trunking Protocol) est un protocole Cisco qui permet de **synchroniser automatiquement** la configuration des VLANs entre plusieurs switchs.

### Les 3 modes VTP

| Mode | Description |
|------|-------------|
| **Server** | Peut créer/modifier/supprimer des VLANs. Propage les changements |
| **Client** | Reçoit les VLANs du serveur. Ne peut pas modifier les VLANs |
| **Transparent** | Ne participe pas au VTP. Possède ses propres VLANs. Transmet les messages VTP |

### Architecture VTP typique

```
        [SW4 - VTP Server]
               |
    +----------+----------+
    |          |          |
[SW1-Client] [SW2-Client] [SW3-Client]
```

### Configuration VTP - Mode Serveur

```bash
# Sur le switch serveur (SW4)
Switch(config)# vtp domain OSIVTP
Switch(config)# vtp mode server
Switch(config)# vtp password mdpvtp
Switch(config)# vtp version 2
Switch(config)# vtp pruning  # Optionnel : économise la bande passante
```

### Configuration VTP - Mode Client

```bash
# Sur les switchs clients (SW1, SW2, SW3)
Switch(config)# vtp domain OSIVTP
Switch(config)# vtp mode client
Switch(config)# vtp password mdpvtp
Switch(config)# vtp version 2
```

### Configuration VTP - Mode Transparent

```bash
# Sur un switch transparent (SWT)
Switch(config)# vtp domain OSIVTP
Switch(config)# vtp mode transparent
Switch(config)# vtp password mdpvtp

# En mode transparent, les VLANs doivent être créés manuellement
Switch(config)# vlan 10
Switch(config-vlan)# name PRODUCTION
Switch(config-vlan)# exit
```

### Vérification VTP

```bash
# Afficher le statut VTP
Switch# show vtp status

# Exemple de sortie :
# VTP Version                     : 2
# Configuration Revision          : 5
# Maximum VLANs supported locally : 255
# Number of existing VLANs        : 7
# VTP Operating Mode              : Server
# VTP Domain Name                 : OSIVTP
# VTP Pruning Mode                : Enabled
# VTP V2 Mode                     : Enabled

# Comparer les tables de VLANs entre switchs
Switch# show vlan brief
```

### ⚠️ Points importants sur le VTP

1. **Numéro de révision** : Le switch avec le numéro de révision le plus élevé écrase la config des autres
2. **Même domaine et mot de passe** : Tous les switchs doivent avoir le même domaine et mot de passe
3. **Trunks requis** : VTP ne fonctionne que sur les liens trunk
4. **Limitation** : VTP ne gère que les VLANs 1-1005 (pas les VLANs étendus 1006-4094)
5. **Stockage** : La config VTP n'est PAS dans la running-config mais dans le fichier `vlan.dat`

### Ajouter un second serveur VTP

```bash
# Passer un client en mode serveur
Switch(config)# vtp mode server

# Vérifier la synchronisation
Switch# show vtp status
```

Les deux serveurs se synchronisent automatiquement. Le dernier changement effectué prévaut.

---

## Sécurisation des Switchs

### Configuration d'une bannière de sécurité

```bash
Switch(config)# banner motd #ACCES RESERVE#
```

### Donner un nom au switch

```bash
Switch(config)# hostname Switch_VOSREVES
```

### Créer des comptes utilisateurs

```bash
# Compte utilisateur standard (mot de passe non chiffré)
Switch(config)# username etudiant password Btssio2017

# Compte administrateur avec privilèges élevés (mot de passe chiffré)
Switch(config)# username admin privilege 15 secret Rootsio2017
```

### Sécuriser l'accès en mode enable

```bash
# Mot de passe pour le mode enable (à éviter, utiliser secret)
Switch(config)# enable password Rootsio2017

# Mot de passe chiffré pour le mode enable (recommandé)
Switch(config)# enable secret Rootsio2017
```

### Sécuriser l'accès console

```bash
Switch(config)# line con 0
Switch(config-line)# password Btssio2017
Switch(config-line)# login local
Switch(config-line)# exit
```

### Chiffrer tous les mots de passe dans la configuration

```bash
# Active le chiffrement de type 7 (faible) pour les mots de passe
Switch(config)# service password-encryption
```

**⚠️ Attention :** Le chiffrement type 7 est facilement décryptable. Utilisez toujours `secret` au lieu de `password` pour un chiffrement MD5 (type 5).

### Sécuriser l'accès Telnet

```bash
Switch(config)# line vty 0 15
Switch(config-line)# password Btssio2017
Switch(config-line)# login local
Switch(config-line)# transport input telnet  # Autoriser Telnet
Switch(config-line)# exit
```

**⚠️ Problème :** Telnet n'est **PAS sécurisé** car les données circulent en clair.

### Désactiver Telnet et activer SSH (recommandé)

#### Étape 1 : Configurer le domaine et générer la clé RSA

```bash
# Définir un nom de domaine
Switch(config)# ip domain-name bts-sio.local

# Générer une clé RSA pour SSH
Switch(config)# crypto key generate rsa general-keys modulus 1024
```

#### Étape 2 : Activer SSH version 2

```bash
Switch(config)# ip ssh version 2
Switch(config)# ip ssh time-out 60
Switch(config)# ip ssh authentication-retries 3
```

#### Étape 3 : Configurer l'authentification AAA (optionnel)

```bash
Switch(config)# aaa new-model
Switch(config)# aaa authentication login default local
```

#### Étape 4 : Configurer les lignes VTY pour SSH uniquement

```bash
# Autoriser uniquement SSH (lignes 0-2)
Switch(config)# line vty 0 2
Switch(config-line)# login local
Switch(config-line)# transport input ssh
Switch(config-line)# transport output ssh
Switch(config-line)# exit

# Désactiver les autres lignes VTY (lignes 3-15)
Switch(config)# line vty 3 15
Switch(config-line)# transport input none
Switch(config-line)# exit
```

#### Étape 5 : Restreindre SSH à certaines adresses IP (optionnel)

```bash
Switch(config)# ip access-list extended SSH_ACCESS
Switch(config-ext-nacl)# permit tcp host 172.17.1.2 any eq 22
Switch(config-ext-nacl)# exit

Switch(config)# line vty 0 2
Switch(config-line)# access-class SSH_ACCESS in
Switch(config-line)# exit
```

### Test de connexion SSH depuis un PC

```bash
# Depuis un PC avec Packet Tracer
PC> ssh -l admin 192.168.30.254
```

---

## Port Security

### Qu'est-ce que le Port Security ?

Le **Port Security** permet de **limiter le nombre d'adresses MAC** autorisées sur un port et de définir des actions en cas de violation.

### Configuration de base

```bash
Switch(config)# interface fa0/1

# Activer le Port Security
Switch(config-if)# switchport port-security

# Définir le nombre maximum d'adresses MAC autorisées
Switch(config-if)# switchport port-security maximum 4

# Apprendre automatiquement les adresses MAC et les sauvegarder
Switch(config-if)# switchport port-security mac-address sticky

# Définir l'action en cas de violation
Switch(config-if)# switchport port-security violation shutdown

Switch(config-if)# exit
```

### Actions en cas de violation

| Action | Description |
|--------|-------------|
| **shutdown** | Désactive le port (err-disabled) |
| **restrict** | Bloque le trafic mais maintient le port actif, génère un log |
| **protect** | Bloque le trafic silencieusement |

### Apprendre manuellement une adresse MAC

```bash
Switch(config-if)# switchport port-security mac-address aaaa.bbbb.cccc
```

### Réactiver un port désactivé

```bash
Switch(config)# interface fa0/1
Switch(config-if)# shutdown
Switch(config-if)# no shutdown
Switch(config-if)# exit
```

### Vérification

```bash
# Afficher l'état du Port Security
Switch# show port-security

# Afficher les détails d'une interface
Switch# show port-security interface fa0/1
```

---

## Protections contre les attaques

### DHCP Snooping (contre DHCP Spoofing et Starvation)

Le **DHCP Snooping** protège contre les serveurs DHCP non autorisés et les attaques d'épuisement du pool DHCP.

#### Configuration

```bash
# Activer DHCP Snooping globalement
Switch(config)# ip dhcp snooping

# Activer DHCP Snooping sur le VLAN 1
Switch(config)# ip dhcp snooping vlan 1

# Limiter le taux de requêtes DHCP par port (4 paquets/seconde)
Switch(config)# interface range fa0/1-24
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport access vlan 1
Switch(config-if-range)# ip dhcp snooping limit rate 4
Switch(config-if-range)# exit

# Marquer le port connecté au serveur DHCP légitime comme "trusted"
Switch(config)# interface gig0/1
Switch(config-if)# ip dhcp snooping trust
Switch(config-if)# exit
```

#### Vérification

```bash
Switch# show ip dhcp snooping
Switch# show ip dhcp snooping binding
```

### IP Source Guard (contre IP Spoofing)

L'**IP Source Guard** bloque les paquets avec une adresse IP source usurpée. Il s'appuie sur DHCP Snooping.

#### Configuration

```bash
# DHCP Snooping doit être activé au préalable

# Activer IP Source Guard sur les ports d'accès
Switch(config)# interface range fa0/1-24
Switch(config-if-range)# ip verify source port-security
Switch(config-if-range)# exit
```

⚠️ **Ne PAS activer** sur le port trusted du serveur DHCP.

#### Vérification

```bash
Switch# show ip verify source
Switch# show ip dhcp snooping binding
```

### ARP Inspection (contre ARP Flooding/Spoofing)

L'**ARP Inspection** valide les paquets ARP pour éviter l'empoisonnement ARP et la saturation de la table CAM.

#### Configuration

```bash
# Activer Dynamic ARP Inspection sur le VLAN 1
Switch(config)# ip arp inspection vlan 1

# Marquer les ports trusted (serveurs, routeurs)
Switch(config)# interface gig0/1
Switch(config-if)# ip arp inspection trust
Switch(config-if)# exit
```

#### Vérification

```bash
Switch# show ip arp inspection
Switch# show ip arp inspection statistics
```

---

## Spanning Tree Protocol

### Qu'est-ce que le Spanning Tree ?

Le **Spanning Tree Protocol (STP)** évite les **boucles réseau** en bloquant certains ports redondants. Il garantit qu'il n'y a qu'un seul chemin actif entre deux switchs.

### Désactiver Spanning Tree (⚠️ DANGEREUX)

```bash
# Désactiver STP sur le VLAN 1 (NE PAS FAIRE EN PRODUCTION)
Switch(config)# no spanning-tree vlan 1
```

**Conséquence :** Si une boucle physique existe, cela provoque une **tempête de broadcast** qui sature le réseau et fait monter le CPU à 100%.

### Réactiver Spanning Tree

```bash
Switch(config)# spanning-tree vlan 1
```

### Vérification

```bash
# Afficher l'état de Spanning Tree
Switch# show spanning-tree

# Détails sur un VLAN spécifique
Switch# show spanning-tree vlan 1
```

### États des ports STP

| État | Description |
|------|-------------|
| **Forwarding (FWD)** | Port actif, transmet le trafic |
| **Blocking (BLK)** | Port bloqué pour éviter une boucle |
| **Listening** | Transition, écoute des BPDUs |
| **Learning** | Apprend les adresses MAC |
| **Disabled** | Port administrativement désactivé |

---

## Dépannage et vérification

### Commandes essentielles

```bash
# Afficher la table d'adresses MAC
Switch# show mac address-table

# Compter les entrées dans la table MAC
Switch# show mac address-table count

# Afficher les VLANs
Switch# show vlan brief

# Afficher les trunks
Switch# show interfaces trunk

# Afficher les détails d'une interface
Switch# show interface fa0/1

# Afficher le statut VTP
Switch# show vtp status

# Afficher le Port Security
Switch# show port-security

# Afficher les erreurs sur une interface
Switch# show interface fa0/1 | include errors

# Afficher les processus CPU
Switch# show processes cpu

# Afficher les logs
Switch# show logging
```

### Analyser les trames 802.1q dans Packet Tracer

1. Passer en **mode simulation**
2. Filtrer le protocole **ICMP**
3. Envoyer un ping entre deux machines sur des VLANs différents
4. Cliquer sur une trame sur un lien trunk
5. Observer le champ **802.1Q Tag** :
   - **TPID** (Tag Protocol Identifier) : `0x8100`
   - **TCI** (Tag Control Information) : contient le **VLAN ID**

### Commandes de diagnostic

```bash
# Tester la connectivité
Switch# ping 192.168.1.1

# Tracer le chemin réseau
Switch# traceroute 192.168.1.1

# Afficher la configuration d'une interface
Switch# show running-config interface fa0/1
```

---

## Récapitulatif des commandes clés

| Action | Commande |
|--------|----------|
| Créer un VLAN | `vlan 10` puis `name PRODUCTION` |
| Affecter un port au VLAN | `switchport access vlan 10` |
| Configurer un trunk | `switchport mode trunk` |
| VLANs autorisés sur trunk | `switchport trunk allowed vlan 10,20,30` |
| VLAN natif | `switchport trunk native vlan 99` |
| Mode VTP serveur | `vtp mode server` |
| Mode VTP client | `vtp mode client` |
| Domaine VTP | `vtp domain OSIVTP` |
| Activer SSH | `crypto key generate rsa` puis `ip ssh version 2` |
| DHCP Snooping | `ip dhcp snooping vlan 1` |
| IP Source Guard | `ip verify source port-security` |
| Port Security | `switchport port-security` |
| Sauvegarder la config | `copy running-config startup-config` |

---

## 🎯 Bonnes pratiques

1. **Toujours sauvegarder** la configuration après chaque modification
2. **Utiliser SSH** au lieu de Telnet pour l'accès à distance
3. **Chiffrer les mots de passe** avec `secret` au lieu de `password`
4. **Créer un VLAN BlackHole** pour les ports inutilisés et les shutdown
5. **Définir un VLAN natif dédié** (ex: VLAN 99) pour la sécurité
6. **Activer DHCP Snooping** pour protéger le réseau
7. **Utiliser le Port Security** sur les ports d'accès
8. **Documenter toutes les modifications** dans un fichier texte

---

## 📝 Exemple de configuration complète

Voici un exemple de configuration complète d'un switch sécurisé avec VLANs et VTP :

```bash
# Réinitialiser le switch
Switch> enable
Switch# erase startup-config
Switch# delete flash:vlan.dat
Switch# reload

# Configuration de base
Switch> enable
Switch# configure terminal
Switch(config)# hostname Switch1
Switch(config)# banner motd #ACCES RESERVE#

# Comptes utilisateurs
Switch(config)# username etudiant password Btssio2017
Switch(config)# username admin privilege 15 secret Rootsio2017
Switch(config)# enable secret Rootsio2017
Switch(config)# service password-encryption

# Sécuriser l'accès console
Switch(config)# line con 0
Switch(config-line)# password Btssio2017
Switch(config-line)# login local
Switch(config-line)# exit

# SSH
Switch(config)# ip domain-name bts-sio.local
Switch(config)# crypto key generate rsa general-keys modulus 1024
Switch(config)# ip ssh version 2
Switch(config)# ip ssh time-out 60
Switch(config)# ip ssh authentication-retries 3

Switch(config)# line vty 0 15
Switch(config-line)# login local
Switch(config-line)# transport input ssh
Switch(config-line)# exit

# VLANs
Switch(config)# vlan 10
Switch(config-vlan)# name PRODUCTION
Switch(config-vlan)# exit
Switch(config)# vlan 20
Switch(config-vlan)# name COMMERCE
Switch(config-vlan)# exit
Switch(config)# vlan 30
Switch(config-vlan)# name GESTION
Switch(config-vlan)# exit
Switch(config)# vlan 40
Switch(config-vlan)# name BlackHole
Switch(config-vlan)# exit
Switch(config)# vlan 99
Switch(config-vlan)# name Native
Switch(config-vlan)# exit

# VTP
Switch(config)# vtp domain OSIVTP
Switch(config)# vtp mode client
Switch(config)# vtp password mdpvtp
Switch(config)# vtp version 2

# IP de gestion
Switch(config)# interface vlan 30
Switch(config-if)# ip address 192.168.30.254 255.255.255.0
Switch(config-if)# no shutdown
Switch(config-if)# exit
Switch(config)# ip default-gateway 192.168.30.1

# Affectation des ports
Switch(config)# interface fa0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10
Switch(config-if)# no shutdown
Switch(config-if)# exit

Switch(config)# interface range fa0/3-4
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport access vlan 20
Switch(config-if-range)# no shutdown
Switch(config-if-range)# exit

Switch(config)# interface fa0/20
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 30
Switch(config-if)# no shutdown
Switch(config-if)# exit

# Ports inutilisés
Switch(config)# interface range fa0/5-19, fa0/21-23
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport access vlan 40
Switch(config-if-range)# shutdown
Switch(config-if-range)# exit

# Trunk
Switch(config)# interface fa0/24
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk native vlan 99
Switch(config-if)# switchport trunk allowed vlan 10,20,30,99
Switch(config-if)# no shutdown
Switch(config-if)# exit

# DHCP Snooping
Switch(config)# ip dhcp snooping
Switch(config)# ip dhcp snooping vlan 10,20
Switch(config)# interface range fa0/1-23
Switch(config-if-range)# ip dhcp snooping limit rate 4
Switch(config-if-range)# exit
Switch(config)# interface gig0/1
Switch(config-if)# ip dhcp snooping trust
Switch(config-if)# exit

# Port Security
Switch(config)# interface range fa0/1-4
Switch(config-if-range)# switchport port-security
Switch(config-if-range)# switchport port-security maximum 4
Switch(config-if-range)# switchport port-security mac-address sticky
Switch(config-if-range)# switchport port-security violation shutdown
Switch(config-if-range)# exit

# Sauvegarder
Switch(config)# exit
Switch# copy running-config startup-config
```

---

**Fin du guide Switchs Cisco** 🎓
