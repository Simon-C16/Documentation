# 📚 TP Active Directory — Guide Complet

> **Objectif** : Installer, configurer et administrer un Active Directory sur Windows Server. Ce guide couvre tout, de l'installation du rôle AD DS jusqu'à la gestion avancée des utilisateurs, GPO, et DNS.

---

## Sommaire

1. [Prérequis et environnement](#1-prérequis-et-environnement)
2. [Installation de Windows Server](#2-installation-de-windows-server)
3. [Configuration réseau avant AD](#3-configuration-réseau-avant-ad)
4. [Installation du rôle AD DS](#4-installation-du-rôle-ad-ds)
5. [Promotion en contrôleur de domaine](#5-promotion-en-contrôleur-de-domaine)
6. [Vérification post-installation](#6-vérification-post-installation)
7. [Gestion des Unités d'Organisation (OU)](#7-gestion-des-unités-dorganisation-ou)
8. [Gestion des utilisateurs](#8-gestion-des-utilisateurs)
9. [Gestion des groupes](#9-gestion-des-groupes)
10. [Joindre un client Windows au domaine](#10-joindre-un-client-windows-au-domaine)
11. [Stratégies de groupe (GPO)](#11-stratégies-de-groupe-gpo)
12. [DNS intégré à l'AD](#12-dns-intégré-à-lad)
13. [DHCP avec Active Directory](#13-dhcp-avec-active-directory)
14. [Gestion des droits et permissions NTFS](#14-gestion-des-droits-et-permissions-ntfs)
15. [Profils itinérants et dossiers redirigés](#15-profils-itinérants-et-dossiers-redirigés)
16. [Contrôleur de domaine secondaire (réplication)](#16-contrôleur-de-domaine-secondaire-réplication)
17. [Commandes PowerShell utiles](#17-commandes-powershell-utiles)
18. [Dépannage courant](#18-dépannage-courant)
19. [Lexique Active Directory](#19-lexique-active-directory)

---

## 1. Prérequis et environnement

### Matériel recommandé (VM ou physique)

| Composant | Minimum | Recommandé |
|-----------|---------|------------|
| CPU | 1,4 GHz 64 bits | 2 cœurs+ |
| RAM | 2 Go | 4 Go+ |
| Disque | 32 Go | 60 Go+ |
| Réseau | 1 carte réseau | 2 cartes (interne + externe) |

### Logiciels nécessaires

- **Windows Server 2019 / 2022** (ISO téléchargeable sur le site Microsoft ou MSDN)
- **Hyperviseur** (si VM) : VMware Workstation, VirtualBox, Hyper-V
- **Client Windows** : Windows 10 ou 11 Pro (pour rejoindre le domaine)

### Topologie du TP

```
┌─────────────────────────────────────────────────┐
│                   Réseau LAN                    │
│                192.168.1.0/24                   │
│                                                 │
│  ┌─────────────────┐    ┌──────────────────┐   │
│  │  Windows Server │    │  Client Windows  │   │
│  │  (DC01)         │    │  (PC-CLIENT01)   │   │
│  │  192.168.1.10   │    │  192.168.1.20    │   │
│  │  DNS: lui-même  │    │  DNS: 192.168.1.10│  │
│  └─────────────────┘    └──────────────────┘   │
└─────────────────────────────────────────────────┘
```

> 💡 **Note** : Le serveur AD doit toujours avoir une **IP fixe**. Ne jamais utiliser le DHCP pour le DC.

---

## 2. Installation de Windows Server

### Étapes d'installation

1. Démarrer depuis l'ISO Windows Server
2. Sélectionner **"Windows Server 2022 Standard (Desktop Experience)"** pour avoir l'interface graphique
3. Choisir **"Personnalisé"** pour une installation propre
4. Laisser le partitionnement automatique ou créer manuellement une partition système

### Après l'installation

- Définir le mot de passe administrateur local (doit être complexe : maj + min + chiffre + spécial)
- Se connecter avec `Administrateur` + le mot de passe défini

### Renommer le serveur (obligatoire avant AD)

```powershell
# Via PowerShell
Rename-Computer -NewName "DC01" -Restart
```

ou via :
> **Panneau de configuration** → **Système** → **Modifier les paramètres** → **Modifier**

> ⚠️ **Important** : Renommer le serveur AVANT d'installer l'AD. Après, c'est plus compliqué.

---

## 3. Configuration réseau avant AD

L'Active Directory repose entièrement sur le DNS. Il faut absolument configurer une IP fixe.

### Configurer l'IP statique via PowerShell

```powershell
# Identifier le nom de la carte réseau
Get-NetAdapter

# Supprimer l'adresse DHCP actuelle
Remove-NetIPAddress -InterfaceAlias "Ethernet" -Confirm:$false

# Assigner une IP fixe
New-NetIPAddress `
  -InterfaceAlias "Ethernet" `
  -IPAddress 192.168.1.10 `
  -PrefixLength 24 `
  -DefaultGateway 192.168.1.1

# Configurer le DNS sur lui-même (loopback)
Set-DnsClientServerAddress `
  -InterfaceAlias "Ethernet" `
  -ServerAddresses ("192.168.1.10", "127.0.0.1")
```

### Via l'interface graphique

1. Clic droit sur l'icône réseau → **Paramètres réseau et Internet**
2. **Modifier les options d'adaptateur**
3. Clic droit sur la carte → **Propriétés**
4. **Protocole Internet version 4 (TCP/IPv4)** → **Propriétés**
5. Remplir :
   - Adresse IP : `192.168.1.10`
   - Masque : `255.255.255.0`
   - Passerelle : `192.168.1.1`
   - DNS préféré : `192.168.1.10` (lui-même)
   - DNS auxiliaire : `127.0.0.1`

### Vérifier la configuration

```powershell
ipconfig /all
ping 192.168.1.10
```

---

## 4. Installation du rôle AD DS

AD DS = **Active Directory Domain Services** — c'est le rôle principal.

### Via le Gestionnaire de serveur (GUI)

1. Ouvrir **Gestionnaire de serveur** (Server Manager)
2. Cliquer sur **"Gérer"** → **"Ajouter des rôles et fonctionnalités"**
3. Cliquer **Suivant** jusqu'à **"Rôles de serveur"**
4. Cocher **"Services de domaine Active Directory"**
5. Une fenêtre apparaît pour ajouter les fonctionnalités requises → Cliquer **"Ajouter des fonctionnalités"**
6. Cliquer **Suivant** plusieurs fois puis **Installer**
7. Attendre la fin de l'installation (ne pas redémarrer encore)

### Via PowerShell

```powershell
# Installer le rôle AD DS et les outils d'administration
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Vérifier l'installation
Get-WindowsFeature AD-Domain-Services
```

---

## 5. Promotion en contrôleur de domaine

C'est l'étape clé : transformer le serveur en **contrôleur de domaine (DC)**.

### Via le Gestionnaire de serveur

1. Dans le Gestionnaire de serveur, cliquer sur le **drapeau jaune** (notification) en haut
2. Cliquer sur **"Promouvoir ce serveur en contrôleur de domaine"**

### Assistant de configuration

#### Étape 1 — Choisir le type de déploiement

Sélectionner : **"Ajouter une nouvelle forêt"**

- **Nom du domaine racine** : `mondomaine.local` *(remplacez par votre nom de domaine)*

> 💡 Utilisez `.local` pour un réseau interne. Évitez les domaines publics existants.

#### Étape 2 — Options du contrôleur de domaine

| Paramètre | Valeur recommandée |
|-----------|-------------------|
| Niveau fonctionnel forêt | Windows Server 2016 ou 2019 |
| Niveau fonctionnel domaine | Windows Server 2016 ou 2019 |
| Serveur DNS | ✅ Coché |
| Catalogue global (GC) | ✅ Coché |
| Mot de passe DSRM | Définir un mot de passe fort |

> **DSRM** = Directory Services Restore Mode — mot de passe de récupération d'urgence.

#### Étape 3 — Options DNS

Une alerte sur la délégation DNS peut apparaître → **Ignorer**, c'est normal pour un nouveau domaine.

#### Étape 4 — Options supplémentaires

- **Nom NetBIOS** : se génère automatiquement (ex: `MONDOMAINE`)

#### Étape 5 — Chemins

Laisser les valeurs par défaut :
- Base de données AD : `C:\Windows\NTDS`
- Fichiers journaux : `C:\Windows\NTDS`
- SYSVOL : `C:\Windows\SYSVOL`

#### Étape 6 — Vérification des prérequis

Cliquer **"Vérifier les prérequis"** — Des avertissements peuvent apparaître, c'est normal. Si tout est en **vert/orange** (pas de rouge), cliquer **"Installer"**.

Le serveur **redémarre automatiquement** à la fin.

### Via PowerShell (promotion complète)

```powershell
# Installer AD DS puis promouvoir en une seule fois
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

Import-Module ADDSDeployment

Install-ADDSForest `
  -DomainName "mondomaine.local" `
  -DomainNetbiosName "MONDOMAINE" `
  -DomainMode "WinThreshold" `
  -ForestMode "WinThreshold" `
  -InstallDns:$true `
  -Force:$true
```

---

## 6. Vérification post-installation

Après le redémarrage, se connecter avec `MONDOMAINE\Administrateur`.

### Vérifications essentielles

```powershell
# Vérifier que l'AD tourne
Get-Service ADWS, KDC, Netlogon, NTDS | Select Name, Status

# Vérifier le domaine
Get-ADDomain

# Vérifier la forêt
Get-ADForest

# Tester le DNS
nslookup mondomaine.local
nslookup dc01.mondomaine.local

# Vérifier les partitions AD
repadmin /showrepl

# Vérifier les rôles FSMO
netdom query fsmo
```

### Outils d'administration AD

Après installation, les outils suivants sont disponibles :

| Outil | Description |
|-------|-------------|
| `dsa.msc` | Utilisateurs et ordinateurs Active Directory |
| `dnsmgmt.msc` | Gestionnaire DNS |
| `gpmc.msc` | Gestion des stratégies de groupe |
| `adsi.msc` | Éditeur ADSI (avancé) |
| `dssite.msc` | Sites et services Active Directory |

---

## 7. Gestion des Unités d'Organisation (OU)

Les **OU (Organizational Units)** permettent d'organiser les objets AD (utilisateurs, groupes, ordinateurs) et d'y appliquer des GPO.

### Créer une OU via la console

1. Ouvrir `dsa.msc` (Utilisateurs et ordinateurs AD)
2. Clic droit sur le domaine → **Nouveau** → **Unité d'organisation**
3. Nommer l'OU (ex: `Informatique`, `RH`, `Direction`)
4. Cocher **"Protéger contre la suppression accidentelle"**

### Structure recommandée pour le TP

```
mondomaine.local
├── OU=Utilisateurs
│   ├── OU=Informatique
│   ├── OU=Ressources Humaines
│   └── OU=Direction
├── OU=Groupes
├── OU=Ordinateurs
│   ├── OU=Postes
│   └── OU=Serveurs
└── OU=Comptes de service
```

### Via PowerShell

```powershell
# Créer une OU
New-ADOrganizationalUnit -Name "Informatique" -Path "DC=mondomaine,DC=local" -ProtectedFromAccidentalDeletion $true

# Créer une sous-OU
New-ADOrganizationalUnit -Name "Developpeurs" -Path "OU=Informatique,DC=mondomaine,DC=local"

# Lister toutes les OU
Get-ADOrganizationalUnit -Filter * | Select Name, DistinguishedName

# Supprimer une OU (désactiver la protection d'abord)
Set-ADOrganizationalUnit -Identity "OU=Test,DC=mondomaine,DC=local" -ProtectedFromAccidentalDeletion $false
Remove-ADOrganizationalUnit -Identity "OU=Test,DC=mondomaine,DC=local" -Confirm:$false
```

---

## 8. Gestion des utilisateurs

### Créer un utilisateur via la console

1. Ouvrir `dsa.msc`
2. Naviguer dans l'OU souhaitée
3. Clic droit → **Nouveau** → **Utilisateur**
4. Remplir :
   - Prénom / Nom
   - Nom d'ouverture de session (ex: `j.dupont`)
5. Définir le mot de passe
6. Options recommandées :
   - ✅ L'utilisateur doit changer son mot de passe à la prochaine ouverture de session
   - ☐ Le mot de passe n'expire jamais (décocher en prod)

### Via PowerShell

```powershell
# Créer un utilisateur
New-ADUser `
  -Name "Jean Dupont" `
  -GivenName "Jean" `
  -Surname "Dupont" `
  -SamAccountName "j.dupont" `
  -UserPrincipalName "j.dupont@mondomaine.local" `
  -Path "OU=Informatique,OU=Utilisateurs,DC=mondomaine,DC=local" `
  -AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) `
  -ChangePasswordAtLogon $true `
  -Enabled $true

# Modifier un utilisateur
Set-ADUser -Identity "j.dupont" -Title "Administrateur Système" -Department "Informatique"

# Désactiver un compte
Disable-ADAccount -Identity "j.dupont"

# Réactiver un compte
Enable-ADAccount -Identity "j.dupont"

# Réinitialiser le mot de passe
Set-ADAccountPassword -Identity "j.dupont" `
  -NewPassword (ConvertTo-SecureString "Nouveau@Pass123!" -AsPlainText -Force) `
  -Reset

# Supprimer un utilisateur
Remove-ADUser -Identity "j.dupont" -Confirm:$false

# Lister tous les utilisateurs d'une OU
Get-ADUser -Filter * -SearchBase "OU=Informatique,OU=Utilisateurs,DC=mondomaine,DC=local" | Select Name, SamAccountName, Enabled

# Créer plusieurs utilisateurs en masse (exemple avec CSV)
Import-Csv "utilisateurs.csv" | ForEach-Object {
  New-ADUser `
    -Name "$($_.Prenom) $($_.Nom)" `
    -GivenName $_.Prenom `
    -Surname $_.Nom `
    -SamAccountName $_.Login `
    -AccountPassword (ConvertTo-SecureString $_.MotDePasse -AsPlainText -Force) `
    -Enabled $true `
    -Path "OU=Utilisateurs,DC=mondomaine,DC=local"
}
```

### Propriétés importantes d'un compte utilisateur

| Onglet | Paramètres clés |
|--------|----------------|
| Général | Nom, prénom, email, téléphone |
| Compte | Login, UPN, options de mot de passe, expiration |
| Profil | Chemin de profil, script d'ouverture de session, lecteur réseau |
| Membre de | Groupes d'appartenance |
| Horaires d'accès | Plages horaires autorisées |
| Se connecter à | Postes autorisés |

---

## 9. Gestion des groupes

### Types de groupes

| Type | Usage |
|------|-------|
| **Sécurité** | Contrôle d'accès aux ressources |
| **Distribution** | Listes de diffusion email (Exchange) |

### Étendues des groupes

| Étendue | Membres | Utilisé dans |
|---------|---------|-------------|
| **Local domaine** | N'importe quel domaine | Ressources du domaine local |
| **Global** | Même domaine | N'importe quel domaine |
| **Universel** | N'importe quel domaine | N'importe quel domaine |

> 💡 **Bonne pratique AGDLP** :
> **A**ccount → **G**lobal group → **D**omain **L**ocal group → **P**ermission

### Créer et gérer des groupes via PowerShell

```powershell
# Créer un groupe
New-ADGroup `
  -Name "GRP_Informatique" `
  -GroupScope Global `
  -GroupCategory Security `
  -Path "OU=Groupes,DC=mondomaine,DC=local" `
  -Description "Groupe sécurité équipe informatique"

# Ajouter un utilisateur à un groupe
Add-ADGroupMember -Identity "GRP_Informatique" -Members "j.dupont"

# Ajouter plusieurs utilisateurs
Add-ADGroupMember -Identity "GRP_Informatique" -Members "j.dupont", "m.martin", "a.bernard"

# Retirer un utilisateur d'un groupe
Remove-ADGroupMember -Identity "GRP_Informatique" -Members "j.dupont" -Confirm:$false

# Voir les membres d'un groupe
Get-ADGroupMember -Identity "GRP_Informatique" | Select Name, SamAccountName

# Voir les groupes d'un utilisateur
Get-ADPrincipalGroupMembership -Identity "j.dupont" | Select Name
```

---

## 10. Joindre un client Windows au domaine

### Prérequis sur le client

1. **IP statique ou DHCP** avec le DNS pointant vers le DC
2. **Connectivité réseau** vers le DC (pingable)

### Configurer le DNS sur le client

```powershell
# Sur le client Windows
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "192.168.1.10"

# Vérifier
nslookup mondomaine.local
```

### Joindre le domaine via PowerShell

```powershell
# Sur le client
Add-Computer `
  -DomainName "mondomaine.local" `
  -Credential (Get-Credential) `
  -OUPath "OU=Postes,OU=Ordinateurs,DC=mondomaine,DC=local" `
  -Restart
```

### Via l'interface graphique

1. Clic droit sur **Ce PC** → **Propriétés**
2. Cliquer sur **"Modifier les paramètres"** (ou Renommer ce PC avancé)
3. Cliquer sur **"Modifier"**
4. Sélectionner **"Domaine"** et entrer `mondomaine.local`
5. Entrer les identifiants d'un compte avec droits de jonction (Administrateur du domaine)
6. **Redémarrer**

### Vérifier la jonction au domaine

```powershell
# Sur le client (après redémarrage)
systeminfo | findstr /i "domain"

# Sur le DC
Get-ADComputer -Filter * | Select Name, DistinguishedName
```

---

## 11. Stratégies de groupe (GPO)

Les GPO (Group Policy Objects) permettent d'appliquer des configurations à des utilisateurs et des ordinateurs.

### Ouvrir la console GPMC

```
gpmc.msc
```

### Créer une GPO

1. Dans `gpmc.msc`, clic droit sur une OU → **"Créer un objet GPO dans ce domaine, et le lier ici"**
2. Donner un nom explicite (ex: `GPO_Fond_Ecran`, `GPO_Mot_De_Passe`)
3. Clic droit sur la GPO → **"Modifier"** pour ouvrir l'éditeur

### Structure d'une GPO

```
GPO
├── Configuration ordinateur
│   ├── Stratégies
│   │   ├── Paramètres Windows
│   │   └── Modèles d'administration
│   └── Préférences
└── Configuration utilisateur
    ├── Stratégies
    │   ├── Paramètres Windows
    │   └── Modèles d'administration
    └── Préférences
```

### Exemples de GPO utiles

#### GPO — Politique de mot de passe

> Via : `Configration ordinateur` → `Stratégies` → `Paramètres Windows` → `Paramètres de sécurité` → `Stratégies de compte` → `Stratégie de mot de passe`

| Paramètre | Valeur recommandée |
|-----------|-------------------|
| Longueur minimale | 8 caractères |
| Complexité | Activé |
| Durée de vie maximale | 90 jours |
| Historique | 10 derniers mots de passe |

#### GPO — Verrouillage de compte

> `Stratégies de compte` → `Stratégie de verrouillage du compte`

| Paramètre | Valeur |
|-----------|--------|
| Seuil de verrouillage | 5 tentatives |
| Durée de verrouillage | 30 minutes |
| Réinitialisation du compteur | 30 minutes |

#### GPO — Fond d'écran commun

> `Configuration utilisateur` → `Stratégies` → `Modèles d'administration` → `Bureau` → `Bureau`

- **Papier peint du bureau** : Activer et spécifier le chemin `\\DC01\Partages\fond_ecran.jpg`

#### GPO — Mappage de lecteur réseau

> `Configuration utilisateur` → `Préférences` → `Paramètres Windows` → `Mappages de lecteurs`

1. Clic droit → **Nouveau** → **Lecteur mappé**
2. Action : **Créer**
3. Emplacement : `\\DC01\Partages\%username%`
4. Lettre : `Z:`

#### GPO — Désactiver le Panneau de configuration

> `Configuration utilisateur` → `Modèles d'administration` → `Panneau de configuration`

- **Interdire l'accès au panneau de configuration** : Activé

### Appliquer et tester une GPO

```powershell
# Forcer l'application des GPO sur le client
gpupdate /force

# Voir les GPO appliquées
gpresult /r

# Rapport GPO détaillé en HTML
gpresult /H C:\Temp\rapport_gpo.html
```

### Ordre d'application des GPO (LSDOU)

1. **L**ocal — Stratégie locale de la machine
2. **S**ite — GPO liée au site AD
3. **D**omaine — GPO liée au domaine
4. **O**U — GPO liée aux OU (de la plus haute à la plus basse)

> 💡 En cas de conflit, **la dernière appliquée gagne** (sauf si "Aucun remplacement" est activé).

---

## 12. DNS intégré à l'AD

Le DNS est installé automatiquement avec l'AD. C'est un composant critique.

### Types de zones DNS

| Type | Description |
|------|-------------|
| **Zone principale** | Contient les enregistrements éditables |
| **Zone secondaire** | Copie en lecture seule d'une zone principale |
| **Zone de stub** | Contient seulement les NS records |
| **Zone intégrée AD** | Stockée dans AD, répliquée automatiquement ✅ |

### Enregistrements DNS importants

| Type | Description | Exemple |
|------|-------------|---------|
| **A** | Nom → IPv4 | `dc01 → 192.168.1.10` |
| **AAAA** | Nom → IPv6 | |
| **CNAME** | Alias | `www → serveur.mondomaine.local` |
| **MX** | Serveur mail | |
| **PTR** | IP → Nom (reverse) | `192.168.1.10 → dc01` |
| **SRV** | Service locator (utilisé par AD) | `_ldap._tcp.mondomaine.local` |

### Gestion DNS via PowerShell

```powershell
# Voir les zones DNS
Get-DnsServerZone

# Ajouter un enregistrement A
Add-DnsServerResourceRecordA -ZoneName "mondomaine.local" -Name "serveur2" -IPv4Address "192.168.1.20"

# Ajouter un alias CNAME
Add-DnsServerResourceRecordCName -ZoneName "mondomaine.local" -Name "www" -HostNameAlias "serveur2.mondomaine.local"

# Créer la zone de recherche inversée
Add-DnsServerPrimaryZone -NetworkID "192.168.1.0/24" -ReplicationScope "Forest"

# Tester la résolution DNS
Resolve-DnsName dc01.mondomaine.local
nslookup dc01.mondomaine.local 192.168.1.10
```

---

## 13. DHCP avec Active Directory

Le DHCP peut être intégré à l'AD pour l'attribution automatique des IP aux clients.

### Installer le rôle DHCP

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```

### Autoriser le serveur DHCP dans l'AD

> ⚠️ Obligatoire ! Un DHCP non autorisé dans l'AD est automatiquement désactivé.

```powershell
Add-DhcpServerInDC -DnsName "DC01.mondomaine.local" -IPAddress 192.168.1.10
Get-DhcpServerInDC  # Vérification
```

### Créer une étendue DHCP

```powershell
# Créer l'étendue
Add-DhcpServerv4Scope `
  -Name "LAN Bureau" `
  -StartRange 192.168.1.100 `
  -EndRange 192.168.1.200 `
  -SubnetMask 255.255.255.0 `
  -State Active

# Définir les options
Set-DhcpServerv4OptionValue `
  -ScopeId 192.168.1.0 `
  -Router 192.168.1.1 `
  -DnsServer 192.168.1.10 `
  -DnsDomain "mondomaine.local"

# Exclure des adresses
Add-DhcpServerv4ExclusionRange `
  -ScopeId 192.168.1.0 `
  -StartRange 192.168.1.100 `
  -EndRange 192.168.1.110

# Vérifier
Get-DhcpServerv4Scope
Get-DhcpServerv4Lease -ScopeId 192.168.1.0
```

---

## 14. Gestion des droits et permissions NTFS

### Partager un dossier

```powershell
# Créer le dossier
New-Item -Path "C:\Partages\Informatique" -ItemType Directory

# Partager le dossier en réseau
New-SmbShare `
  -Name "Informatique" `
  -Path "C:\Partages\Informatique" `
  -FullAccess "MONDOMAINE\Administrateurs" `
  -ChangeAccess "MONDOMAINE\GRP_Informatique" `
  -ReadAccess "MONDOMAINE\Utilisateurs du domaine"
```

### Permissions NTFS via PowerShell

```powershell
# Voir les permissions actuelles
Get-Acl -Path "C:\Partages\Informatique" | Format-List

# Ajouter une permission
$acl = Get-Acl "C:\Partages\Informatique"
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
  "MONDOMAINE\GRP_Informatique",
  "Modify",
  "ContainerInherit,ObjectInherit",
  "None",
  "Allow"
)
$acl.SetAccessRule($rule)
Set-Acl -Path "C:\Partages\Informatique" -AclObject $acl
```

### Niveaux de permissions NTFS

| Permission | Description |
|-----------|-------------|
| **Contrôle total** | Tout faire, y compris changer les permissions |
| **Modifier** | Lire, écrire, supprimer |
| **Lecture et exécution** | Lire et exécuter des programmes |
| **Lecture** | Voir les fichiers et dossiers |
| **Écriture** | Créer des fichiers, modifier le contenu |

> ⚠️ **Règle** : La permission effective = intersection entre permissions réseau (partage) et permissions NTFS. La plus restrictive l'emporte.

---

## 15. Profils itinérants et dossiers redirigés

### Profils itinérants

Permettent à l'utilisateur de retrouver son profil sur n'importe quel PC du domaine.

```powershell
# Créer le partage pour les profils
New-SmbShare -Name "Profils$" -Path "C:\Profils" -FullAccess "Tout le monde"

# Affecter le profil à un utilisateur
Set-ADUser -Identity "j.dupont" -ProfilePath "\\DC01\Profils$\j.dupont"
```

### Redirection de dossiers (GPO)

> `Configuration utilisateur` → `Stratégies` → `Paramètres Windows` → `Redirection de dossiers`

Clic droit sur **Documents** → **Propriétés** :
- Paramètre : **"De base - Rediriger tout le monde vers le même emplacement"**
- Emplacement : `\\DC01\Profils$\%username%\Documents`

---

## 16. Contrôleur de domaine secondaire (réplication)

Ajouter un deuxième DC pour la **haute disponibilité** et la **redondance**.

### Sur le second serveur

```powershell
# Installer AD DS
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Rejoindre le domaine existant (pas créer une nouvelle forêt !)
Install-ADDSDomainController `
  -DomainName "mondomaine.local" `
  -InstallDns:$true `
  -Credential (Get-Credential "MONDOMAINE\Administrateur") `
  -Force:$true
```

### Vérifier la réplication

```powershell
# Sur n'importe quel DC
repadmin /showrepl
repadmin /replsummary
repadmin /syncall /AdeP

# Forcer la réplication
repadmin /syncall DC02 /AdeP
```

### Rôles FSMO

Il existe 5 rôles FSMO (maîtres d'opérations) :

| Rôle | Périmètre | Description |
|------|-----------|-------------|
| **Schema Master** | Forêt | Gère le schéma AD |
| **Domain Naming Master** | Forêt | Gère l'ajout/suppression de domaines |
| **RID Master** | Domaine | Distribue les pools RID |
| **PDC Emulator** | Domaine | Synchronisation temps, verrouillage de comptes |
| **Infrastructure Master** | Domaine | Gère les références entre domaines |

```powershell
# Voir les détenteurs des rôles FSMO
netdom query fsmo

# Transférer un rôle FSMO (exemple PDC)
Move-ADDirectoryServerOperationMasterRole -Identity "DC02" -OperationMasterRole PDCEmulator
```

---

## 17. Commandes PowerShell utiles

### Module Active Directory

```powershell
# Importer le module (normalement auto sur un DC)
Import-Module ActiveDirectory

# Rechercher un utilisateur
Get-ADUser -Identity "j.dupont" -Properties *

# Recherche avec filtre
Get-ADUser -Filter {Department -eq "Informatique"} | Select Name, SamAccountName

# Trouver les comptes inactifs depuis 90 jours
Search-ADAccount -AccountInactive -TimeSpan 90.00:00:00 -UsersOnly | Select Name, LastLogonDate

# Trouver les comptes désactivés
Search-ADAccount -AccountDisabled -UsersOnly | Select Name

# Lister tous les ordinateurs
Get-ADComputer -Filter * | Select Name, OperatingSystem

# Exporter en CSV
Get-ADUser -Filter * -Properties * | Export-Csv -Path "C:\utilisateurs_ad.csv" -NoTypeInformation -Encoding UTF8
```

### Commandes de diagnostic

```powershell
# Test de connectivité AD
dcdiag /test:DNS
dcdiag /test:Connectivity
dcdiag /test:Replications
dcdiag /v  # Rapport complet

# Vérifier les événements AD
Get-EventLog -LogName "Directory Service" -Newest 20

# Voir l'état du service Netlogon
nltest /dsgetdc:mondomaine.local
nltest /sc_verify:mondomaine.local
```

---

## 18. Dépannage courant

### Le client ne peut pas rejoindre le domaine

```
❌ "Le nom de domaine spécifié n'existe pas ou est inaccessible"
```

✅ **Solutions :**
1. Vérifier que le DNS du client pointe vers `192.168.1.10`
2. `ping dc01.mondomaine.local` doit fonctionner
3. `nslookup mondomaine.local` doit retourner l'IP du DC
4. Vérifier le pare-feu Windows sur le DC (autoriser le port 389, 88, 445, 53)

---

### Impossible de se connecter avec un compte de domaine

```
❌ "Le nom d'utilisateur ou le mot de passe est incorrect"
```

✅ **Solutions :**
1. Vérifier que le compte est actif : `Get-ADUser j.dupont -Properties Enabled`
2. Vérifier le verrouillage : `Get-ADUser j.dupont -Properties LockedOut`
3. Déverrouiller : `Unlock-ADAccount -Identity "j.dupont"`
4. Vérifier l'horloge du client (l'AD est sensible au décalage horaire > 5 min)

---

### La GPO ne s'applique pas

```
❌ Paramètre GPO non pris en compte sur le client
```

✅ **Solutions :**
1. `gpupdate /force` sur le client
2. Vérifier que la GPO est bien liée à la bonne OU
3. Vérifier les filtres de sécurité : l'utilisateur doit avoir "Lecture" et "Appliquer la stratégie de groupe"
4. `gpresult /r` pour voir quelles GPO sont appliquées
5. `rsop.msc` pour voir le résultat résultant de stratégie

---

### La réplication AD échoue

```
❌ Erreur repadmin : "The naming context is in the process of being removed or is not replicated"
```

✅ **Solutions :**
1. `repadmin /showrepl` pour identifier le problème
2. Vérifier la connectivité réseau entre les DC
3. Synchroniser l'heure : `w32tm /resync /force`
4. `netdom resetpwd /server:DC01 /userd:MONDOMAINE\Administrateur /passwordd:*`

---

### Erreur DNS — Le DC ne se résout pas

✅ **Solutions :**
1. `ipconfig /flushdns` sur le client
2. `ipconfig /registerdns` sur le DC
3. Vérifier les enregistrements SRV : `nslookup -type=SRV _ldap._tcp.mondomaine.local`
4. Forcer l'enregistrement DNS du DC : `nltest /dsregdns`

---

## 19. Lexique Active Directory

| Terme | Définition |
|-------|-----------|
| **AD / AD DS** | Active Directory Domain Services — annuaire centralisé Microsoft |
| **DC** | Domain Controller — serveur qui héberge et gère l'AD |
| **Domaine** | Périmètre d'administration AD (ex: `mondomaine.local`) |
| **Forêt** | Ensemble de domaines AD partageant le même schéma |
| **Arbre** | Ensemble de domaines avec un espace de noms DNS contigu |
| **OU** | Organizational Unit — conteneur d'objets AD |
| **GPO** | Group Policy Object — objet de stratégie de groupe |
| **DN** | Distinguished Name — chemin unique d'un objet dans l'AD |
| **UPN** | User Principal Name — login@domaine (format email) |
| **SAM** | Security Account Manager — nom court de login (NetBIOS) |
| **SID** | Security Identifier — identifiant unique de sécurité |
| **LDAP** | Lightweight Directory Access Protocol — protocole d'accès à l'AD |
| **Kerberos** | Protocole d'authentification utilisé par l'AD |
| **FSMO** | Flexible Single Master Operation — rôles uniques dans l'AD |
| **SYSVOL** | Dossier partagé répliqué contenant les GPO et scripts |
| **NTDS.DIT** | Fichier de base de données de l'AD |
| **Catalogue global** | Index de tous les objets de la forêt |
| **Trust** | Relation de confiance entre domaines |
| **DSRM** | Directory Services Restore Mode — mode de récupération |
| **AGDLP** | Best practice : Account → Global → Domain Local → Permission |

---

## Annexe — Ports réseau utilisés par l'AD

| Port | Protocole | Service |
|------|-----------|---------|
| 53 | TCP/UDP | DNS |
| 88 | TCP/UDP | Kerberos |
| 135 | TCP | RPC Endpoint Mapper |
| 139 | TCP | NetBIOS |
| 389 | TCP/UDP | LDAP |
| 445 | TCP | SMB (SYSVOL, NETLOGON) |
| 464 | TCP/UDP | Kerberos password change |
| 636 | TCP | LDAPS (LDAP sécurisé) |
| 3268 | TCP | Catalogue global |
| 3269 | TCP | Catalogue global SSL |
| 49152-65535 | TCP | RPC dynamique |

---

*Document réalisé pour un TP de formation — Active Directory sur Windows Server 2019/2022*
*Tous les exemples utilisent le domaine `mondomaine.local` et le réseau `192.168.1.0/24` — à adapter selon votre environnement.*
