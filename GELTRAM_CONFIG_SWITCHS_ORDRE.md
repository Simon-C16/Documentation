# 🎯 Configuration des Switchs Cisco - Projet GELTRAM
## Guide Pas à Pas dans l'ORDRE

---

## 📋 Vue d'ensemble de l'infrastructure

**VLANs à créer :**
- VLAN 100 : Administrateurs (DSI)
- VLAN 30 : Commerciaux (Vente)
- VLAN 20 : Production
- VLAN 10 : Serveurs
- VLAN 99 : Sauvegarde

**Architecture :**
- Switch LAN CLIENTS (pour les postes utilisateurs)
- Switch LAN SERVEURS (pour les serveurs)
- Trunk entre les deux switchs
- Trunk vers le routeur pfSense

---

## 🚀 ÉTAPE 1 : PRÉPARATION ET RÉINITIALISATION

### 1.1 Réinitialiser le switch (si nécessaire)

```bash
Switch> enable
Switch# erase startup-config
Switch# delete flash:vlan.dat
Switch# reload
```

**⏱️ Le switch redémarre (attendre 2-3 minutes)**

---

## 🔧 ÉTAPE 2 : CONFIGURATION DE BASE

### 2.1 Nommer le switch

```bash
Switch> enable
Switch# configure terminal

# Pour le switch CLIENTS
Switch(config)# hostname SW-CLIENTS

# Pour le switch SERVEURS
Switch(config)# hostname SW-SERVEURS
```

### 2.2 Bannière de sécurité

```bash
SW-CLIENTS(config)# banner motd #
*************************************
* ACCES RESERVE - GELTRAM          *
* Infrastructure reseau protegee   *
*************************************
#
```

### 2.3 Créer les comptes utilisateurs

```bash
# Compte utilisateur standard
SW-CLIENTS(config)# username etudiant password Btssio2017

# Compte administrateur
SW-CLIENTS(config)# username admin privilege 15 secret AdminGeltram2025

# Mot de passe enable
SW-CLIENTS(config)# enable secret RootGeltram2025

# Chiffrer tous les mots de passe
SW-CLIENTS(config)# service password-encryption
```

### 2.4 Sécuriser l'accès console

```bash
SW-CLIENTS(config)# line con 0
SW-CLIENTS(config-line)# password Btssio2017
SW-CLIENTS(config-line)# login local
SW-CLIENTS(config-line)# logging synchronous
SW-CLIENTS(config-line)# exec-timeout 5 0
SW-CLIENTS(config-line)# exit
```

---

## 🌐 ÉTAPE 3 : CONFIGURATION SSH (ACCÈS DISTANT SÉCURISÉ)

### 3.1 Configurer SSH

```bash
# Définir le nom de domaine
SW-CLIENTS(config)# ip domain-name geltram.intra

# Générer les clés RSA
SW-CLIENTS(config)# crypto key generate rsa general-keys modulus 2048
# (Choisir 2048 bits pour plus de sécurité)

# Configurer SSH version 2
SW-CLIENTS(config)# ip ssh version 2
SW-CLIENTS(config)# ip ssh time-out 60
SW-CLIENTS(config)# ip ssh authentication-retries 3
```

### 3.2 Sécuriser les lignes VTY (accès distant)

```bash
SW-CLIENTS(config)# line vty 0 15
SW-CLIENTS(config-line)# login local
SW-CLIENTS(config-line)# transport input ssh
SW-CLIENTS(config-line)# exec-timeout 5 0
SW-CLIENTS(config-line)# exit
```

---

## 📦 ÉTAPE 4 : CRÉATION DES VLANs

### 4.1 Créer tous les VLANs

```bash
# VLAN 10 - Serveurs
SW-CLIENTS(config)# vlan 10
SW-CLIENTS(config-vlan)# name SERVEURS
SW-CLIENTS(config-vlan)# exit

# VLAN 20 - Production
SW-CLIENTS(config)# vlan 20
SW-CLIENTS(config-vlan)# name PRODUCTION
SW-CLIENTS(config-vlan)# exit

# VLAN 30 - Commerciaux
SW-CLIENTS(config)# vlan 30
SW-CLIENTS(config-vlan)# name COMMERCIAUX
SW-CLIENTS(config-vlan)# exit

# VLAN 99 - Sauvegarde
SW-CLIENTS(config)# vlan 99
SW-CLIENTS(config-vlan)# name SAUVEGARDE
SW-CLIENTS(config-vlan)# exit

# VLAN 100 - Administrateurs
SW-CLIENTS(config)# vlan 100
SW-CLIENTS(config-vlan)# name ADMINISTRATEURS
SW-CLIENTS(config-vlan)# exit

# VLAN 999 - BlackHole (ports inutilisés)
SW-CLIENTS(config)# vlan 999
SW-CLIENTS(config-vlan)# name BLACKHOLE
SW-CLIENTS(config-vlan)# exit

# VLAN 1000 - Native (pour le trunking)
SW-CLIENTS(config)# vlan 1000
SW-CLIENTS(config-vlan)# name NATIVE
SW-CLIENTS(config-vlan)# exit
```

### 4.2 Vérifier les VLANs

```bash
SW-CLIENTS(config)# exit
SW-CLIENTS# show vlan brief
```

---

## 🔌 ÉTAPE 5 : AFFECTER LES PORTS AUX VLANs

### 5.1 Sur SW-CLIENTS (exemple d'affectation)

```bash
SW-CLIENTS(config)# interface range fa0/1-5
SW-CLIENTS(config-if-range)# switchport mode access
SW-CLIENTS(config-if-range)# switchport access vlan 100
SW-CLIENTS(config-if-range)# no shutdown
SW-CLIENTS(config-if-range)# exit

SW-CLIENTS(config)# interface range fa0/6-10
SW-CLIENTS(config-if-range)# switchport mode access
SW-CLIENTS(config-if-range)# switchport access vlan 30
SW-CLIENTS(config-if-range)# no shutdown
SW-CLIENTS(config-if-range)# exit

SW-CLIENTS(config)# interface range fa0/11-15
SW-CLIENTS(config-if-range)# switchport mode access
SW-CLIENTS(config-if-range)# switchport access vlan 20
SW-CLIENTS(config-if-range)# no shutdown
SW-CLIENTS(config-if-range)# exit
```

### 5.2 Mettre les ports inutilisés dans le BlackHole

```bash
SW-CLIENTS(config)# interface range fa0/16-22
SW-CLIENTS(config-if-range)# switchport mode access
SW-CLIENTS(config-if-range)# switchport access vlan 999
SW-CLIENTS(config-if-range)# shutdown
SW-CLIENTS(config-if-range)# exit
```

---

## 🔗 ÉTAPE 6 : CONFIGURATION DU TRUNKING

### 6.1 Trunk entre SW-CLIENTS et SW-SERVEURS (port Fa0/24)

```bash
# Sur SW-CLIENTS
SW-CLIENTS(config)# interface fa0/24
SW-CLIENTS(config-if)# switchport mode trunk
SW-CLIENTS(config-if)# switchport trunk native vlan 1000
SW-CLIENTS(config-if)# switchport trunk allowed vlan 10,20,30,99,100
SW-CLIENTS(config-if)# no shutdown
SW-CLIENTS(config-if)# exit
```

```bash
# Sur SW-SERVEURS (même configuration)
SW-SERVEURS(config)# interface fa0/24
SW-SERVEURS(config-if)# switchport mode trunk
SW-SERVEURS(config-if)# switchport trunk native vlan 1000
SW-SERVEURS(config-if)# switchport trunk allowed vlan 10,20,30,99,100
SW-SERVEURS(config-if)# no shutdown
SW-SERVEURS(config-if)# exit
```

### 6.2 Trunk vers le routeur pfSense (port Fa0/23)

```bash
# Sur SW-CLIENTS (ou SW-SERVEURS selon architecture)
SW-CLIENTS(config)# interface fa0/23
SW-CLIENTS(config-if)# switchport mode trunk
SW-CLIENTS(config-if)# switchport trunk native vlan 1000
SW-CLIENTS(config-if)# switchport trunk allowed vlan 10,20,30,99,100
SW-CLIENTS(config-if)# no shutdown
SW-CLIENTS(config-if)# exit
```

### 6.3 Vérifier les trunks

```bash
SW-CLIENTS# show interfaces trunk
```

---

## 🛡️ ÉTAPE 7 : CONFIGURATION DE LA SÉCURITÉ

### 7.1 DHCP Snooping

```bash
# Activer DHCP Snooping globalement
SW-CLIENTS(config)# ip dhcp snooping

# Activer sur les VLANs clients
SW-CLIENTS(config)# ip dhcp snooping vlan 20,30,100

# Ne pas insérer l'option 82
SW-CLIENTS(config)# no ip dhcp snooping information option

# Limiter le taux sur les ports clients
SW-CLIENTS(config)# interface range fa0/1-22
SW-CLIENTS(config-if-range)# ip dhcp snooping limit rate 10
SW-CLIENTS(config-if-range)# exit

# Marquer les ports trusted (trunk et serveurs DHCP)
SW-CLIENTS(config)# interface fa0/23
SW-CLIENTS(config-if)# ip dhcp snooping trust
SW-CLIENTS(config-if)# exit

SW-CLIENTS(config)# interface fa0/24
SW-CLIENTS(config-if)# ip dhcp snooping trust
SW-CLIENTS(config-if)# exit
```

### 7.2 Port Security

**⚠️ IMPORTANT : Ne PAS appliquer le Port Security sur les ports trunk !**

```bash
# Port Security sur les ports ADMINISTRATEURS (VLAN 100)
SW-CLIENTS(config)# interface range fa0/1-5
SW-CLIENTS(config-if-range)# switchport port-security
SW-CLIENTS(config-if-range)# switchport port-security maximum 2
SW-CLIENTS(config-if-range)# switchport port-security mac-address sticky
SW-CLIENTS(config-if-range)# switchport port-security violation shutdown
SW-CLIENTS(config-if-range)# exit

# Port Security sur les ports COMMERCIAUX (VLAN 30)
SW-CLIENTS(config)# interface range fa0/6-10
SW-CLIENTS(config-if-range)# switchport port-security
SW-CLIENTS(config-if-range)# switchport port-security maximum 2
SW-CLIENTS(config-if-range)# switchport port-security mac-address sticky
SW-CLIENTS(config-if-range)# switchport port-security violation shutdown
SW-CLIENTS(config-if-range)# exit

# Port Security sur les ports PRODUCTION (VLAN 20)
SW-CLIENTS(config)# interface range fa0/11-15
SW-CLIENTS(config-if-range)# switchport port-security
SW-CLIENTS(config-if-range)# switchport port-security maximum 2
SW-CLIENTS(config-if-range)# switchport port-security mac-address sticky
SW-CLIENTS(config-if-range)# switchport port-security violation shutdown
SW-CLIENTS(config-if-range)# exit
```

### 7.3 Vérifier le Port Security

```bash
SW-CLIENTS# show port-security
SW-CLIENTS# show port-security interface fa0/1
```

---

## 🌍 ÉTAPE 8 : ADRESSE IP DE GESTION DU SWITCH

### 8.1 Configurer l'IP de gestion

```bash
# Sur SW-CLIENTS (dans le VLAN 100 - Administrateurs)
SW-CLIENTS(config)# interface vlan 100
SW-CLIENTS(config-if)# ip address 192.168.100.252 255.255.255.0
SW-CLIENTS(config-if)# no shutdown
SW-CLIENTS(config-if)# exit

# Définir la passerelle par défaut
SW-CLIENTS(config)# ip default-gateway 192.168.100.1

# Définir le serveur DNS
SW-CLIENTS(config)# ip name-server 192.168.10.2
```

```bash
# Sur SW-SERVEURS (dans le VLAN 10 - Serveurs)
SW-SERVEURS(config)# interface vlan 10
SW-SERVEURS(config-if)# ip address 192.168.10.252 255.255.255.0
SW-SERVEURS(config-if)# no shutdown
SW-SERVEURS(config-if)# exit

SW-SERVEURS(config)# ip default-gateway 192.168.10.1
SW-SERVEURS(config)# ip name-server 192.168.10.2
```

---

## 📊 ÉTAPE 9 : PROTECTIONS SUPPLÉMENTAIRES

### 9.1 Protection BPDU Guard (empêche les attaques STP)

```bash
# Activer BPDU Guard sur tous les ports access
SW-CLIENTS(config)# interface range fa0/1-22
SW-CLIENTS(config-if-range)# spanning-tree bpduguard enable
SW-CLIENTS(config-if-range)# exit
```

### 9.2 Protection contre les attaques CAM Table

```bash
# Limiter le nombre d'adresses MAC apprises
SW-CLIENTS(config)# interface range fa0/1-22
SW-CLIENTS(config-if-range)# switchport port-security maximum 3
SW-CLIENTS(config-if-range)# exit
```

---

## 💾 ÉTAPE 10 : SAUVEGARDER LA CONFIGURATION

### 10.1 Sauvegarder

```bash
SW-CLIENTS(config)# exit
SW-CLIENTS# copy running-config startup-config
# ou version courte :
SW-CLIENTS# wr
```

### 10.2 Vérifier la configuration sauvegardée

```bash
SW-CLIENTS# show startup-config
```

---

## ✅ ÉTAPE 11 : VÉRIFICATIONS FINALES

### 11.1 Checklist de vérification

```bash
# 1. Vérifier les VLANs
SW-CLIENTS# show vlan brief

# 2. Vérifier les trunks
SW-CLIENTS# show interfaces trunk

# 3. Vérifier le Port Security
SW-CLIENTS# show port-security

# 4. Vérifier DHCP Snooping
SW-CLIENTS# show ip dhcp snooping

# 5. Vérifier l'IP de gestion
SW-CLIENTS# show ip interface brief

# 6. Vérifier la configuration SSH
SW-CLIENTS# show ip ssh

# 7. Tester l'accès SSH depuis un PC du VLAN 100
# ssh admin@192.168.100.252
```

---

## 📝 RÉSUMÉ DE LA CONFIGURATION

### VLANs créés :
- ✅ VLAN 10 : SERVEURS
- ✅ VLAN 20 : PRODUCTION
- ✅ VLAN 30 : COMMERCIAUX
- ✅ VLAN 99 : SAUVEGARDE
- ✅ VLAN 100 : ADMINISTRATEURS
- ✅ VLAN 999 : BLACKHOLE
- ✅ VLAN 1000 : NATIVE

### Sécurité activée :
- ✅ SSH version 2
- ✅ Chiffrement des mots de passe
- ✅ DHCP Snooping
- ✅ Port Security
- ✅ BPDU Guard
- ✅ Ports inutilisés shutdown

### Trunking configuré :
- ✅ Trunk SW-CLIENTS ↔ SW-SERVEURS
- ✅ Trunk vers routeur pfSense
- ✅ VLAN natif 1000

---

## ⚠️ POINTS D'ATTENTION

### ❌ NE JAMAIS FAIRE :
- ❌ Mettre du Port Security sur les ports trunk
- ❌ Oublier de sauvegarder avec `wr`
- ❌ Laisser les ports inutilisés actifs
- ❌ Utiliser le VLAN 1 (défaut)

### ✅ TOUJOURS FAIRE :
- ✅ Forcer `switchport mode access` avant Port Security
- ✅ Définir un VLAN natif dédié pour les trunks
- ✅ Activer DHCP Snooping sur les VLANs clients
- ✅ Marquer les trunks en `trust` pour DHCP Snooping
- ✅ Documenter toutes les modifications

---

## 🎯 ORDRE DE CONFIGURATION RAPIDE (RÉSUMÉ)

1. **Préparation** : Réinitialiser le switch
2. **Base** : Hostname, bannière, comptes utilisateurs
3. **SSH** : Domaine, clés RSA, ligne VTY
4. **VLANs** : Créer tous les VLANs
5. **Ports** : Affecter les ports aux VLANs
6. **Trunk** : Configurer les liens trunk
7. **Sécurité** : DHCP Snooping, Port Security, BPDU Guard
8. **IP Gestion** : Interface VLAN + passerelle
9. **Sauvegarde** : `copy run start`
10. **Vérifications** : Tout tester !

---

**🎓 Configuration terminée ! Ton infrastructure Geltram est sécurisée et opérationnelle.**
