# 📘 Documentation Complète – Active Directory sous Windows Server 2022

> **Usage :** Cette documentation compile toutes les étapes nécessaires pour configurer un environnement Active Directory complet (AD DS, DNS, DHCP, IPAM, partage de fichiers, GPO, GLPI). Suivez les sections dans l'ordre.

---

## 📋 Table des matières

1. [Prérequis avant de commencer](#1-prérequis-avant-de-commencer)
2. [Installation du rôle AD DS et création du domaine](#2-installation-du-rôle-ad-ds-et-création-du-domaine)
3. [Configuration du DNS](#3-configuration-du-dns)
4. [Configuration du DHCP](#4-configuration-du-dhcp)
5. [Création des utilisateurs, groupes et UO](#5-création-des-utilisateurs-groupes-et-uo)
6. [Gestion des profils et dossiers personnels](#6-gestion-des-profils-et-dossiers-personnels)
7. [Partage de disques et permissions NTFS](#7-partage-de-disques-et-permissions-ntfs)
8. [Scripts de connexion réseau](#8-scripts-de-connexion-réseau)
9. [Stratégies de groupe (GPO)](#9-stratégies-de-groupe-gpo)
10. [Installation et configuration d'IPAM](#10-installation-et-configuration-dipam)
11. [Intégration GLPI avec Active Directory (LDAP)](#11-intégration-glpi-avec-active-directory-ldap)
12. [Intégration d'un poste client dans le domaine](#12-intégration-dun-poste-client-dans-le-domaine)
13. [Commandes utiles et vérifications](#13-commandes-utiles-et-vérifications)
14. [Checklist finale de validation](#14-checklist-finale-de-validation)

---

## 1. Prérequis avant de commencer

Avant toute installation, effectuez ces actions sur votre serveur Windows Server 2022 :

- [ ] Définir un **nom d'hôte** fixe (ex: `AD-GELTRAM` ou `SRV-DC`)
- [ ] Configurer une **adresse IP statique** (ex: `192.168.0.1`, masque `/24`)
- [ ] DNS primaire pointant sur **lui-même** (`127.0.0.1` ou son IP statique)
- [ ] Appliquer les dernières **mises à jour Windows**
- [ ] Définir un **mot de passe complexe** pour le compte Administrateur local

> ⚠️ **Important :** IPAM ne doit **jamais** être installé sur un serveur DHCP. Utilisez un serveur applicatif dédié pour IPAM.

---

## 2. Installation du rôle AD DS et création du domaine

### 2.1 Installer le rôle AD DS

1. Ouvrir le **Gestionnaire de serveur**
2. Cliquer sur **Gérer > Ajouter des rôles et fonctionnalités**
3. Choisir : **Installation basée sur un rôle ou une fonctionnalité**
4. Sélectionner votre serveur local
5. Cocher **Services AD DS**
6. Valider l'ajout des outils de gestion (console ADUC, module PowerShell AD)
7. Cliquer sur **Installer** et attendre la fin

### 2.2 Promouvoir le serveur en contrôleur de domaine

1. Dans le Gestionnaire de serveur, cliquer sur le **drapeau orange** > **Promouvoir ce serveur en contrôleur de domaine**
2. Choisir **Ajouter une nouvelle forêt**
3. Renseigner le nom de domaine (ex: `geltram.intra`)

> ⚠️ Éviter les TLD publics comme `.com` pour un domaine interne. Préférer `.intra`, `.local`, ou `.lan`.

4. Niveau fonctionnel : **Windows Server 2016** (ou plus récent)
5. Cocher : **Serveur DNS** et **Catalogue global**
6. Définir le **mot de passe DSRM** (restauration annuaire – différent du mot de passe admin)
7. Laisser le **nom NETBIOS** généré automatiquement (ex: `GELTRAM`)
8. Laisser les **chemins par défaut** pour la base NTDS
9. Vérifier le résumé, puis cliquer sur **Installer**
10. Le serveur **redémarre automatiquement** à la fin

### 2.3 Vérification après redémarrage

```powershell
# Vérifier que le DC est opérationnel
Get-ADDomain
Get-ADForest
dcdiag /test:services
```

---

## 3. Configuration du DNS

### 3.1 Zone de recherche directe

La zone de recherche directe est créée automatiquement lors de la promotion AD.

1. Ouvrir **Gestionnaire DNS** (Outils > DNS)
2. Vérifier la présence de la zone `geltram.intra` sous **Zones de recherche directe**

### 3.2 Créer un enregistrement hôte (A)

1. Clic droit sur la zone `geltram.intra` > **Nouvel hôte (A ou AAAA)**
2. Nom : `web` (ou le nom souhaité)
3. Adresse IP : `192.168.0.10`
4. Cocher **Créer un enregistrement PTR associé**
5. Cliquer sur **Ajouter un hôte**

### 3.3 Zone de recherche inversée

1. Clic droit sur **Zones de recherche inversée** > **Nouvelle zone**
2. Type : **Zone principale**
3. Réseau : `192.168.0` (pour un réseau /24)
4. Laisser les paramètres par défaut

### 3.4 Activer le PTR dans la zone directe

Lors de la création d'enregistrements A, cocher systématiquement **"Créer un enregistrement PTR associé"**.

### 3.5 Tester la résolution DNS

```cmd
nslookup web.geltram.intra
nslookup 192.168.0.10
```

---

## 4. Configuration du DHCP

### 4.1 Installer le rôle DHCP

1. **Gestionnaire de serveur > Ajouter des rôles > Serveur DHCP**
2. Installer et terminer l'assistant

### 4.2 Créer une étendue DHCP

1. Ouvrir la console **DHCP** (Outils > DHCP)
2. Clic droit sur **IPv4** > **Nouvelle étendue**
3. Paramètres :
   - Nom : `LAN-geltram`
   - Plage IP : `192.168.0.10` à `192.168.0.100`
   - Masque : `255.255.255.0` (/24)
   - Durée du bail : `4 heures`
4. Options d'étendue :
   - **Routeur (003)** : `192.168.0.254`
   - **Serveur DNS (006)** : `192.168.0.1`
   - **Nom de domaine (015)** : `geltram.intra`
5. **Activer l'étendue** à la fin

### 4.3 Vérification

Depuis un client configuré en DHCP, vérifier qu'il obtient une adresse dans la plage définie :

```cmd
ipconfig /all
```

---

## 5. Création des utilisateurs, groupes et UO

### 5.1 Créer des Unités d'Organisation (OU)

1. Ouvrir **Utilisateurs et ordinateurs Active Directory** (ADUC)
2. Clic droit sur le domaine > **Nouveau > Unité d'organisation**

**OU à créer (exemple BTS SIO) :**
- `Professeurs`
- `Etudiants`

**OU à créer (exemple Geltram) :**
- `Services`
  - `Vente`
  - `Atelier`
  - `DSI`
  - `Production`
- `Utilisateurs`

### 5.2 Créer des utilisateurs

1. Clic droit sur l'OU cible > **Nouveau > Utilisateur**
2. Renseigner :
   - Prénom / Nom
   - Nom d'ouverture de session (ex: `eleve1`, `prof1`)
   - Mot de passe (ex: `Btssio2017!`)
3. Options recommandées :
   - Décocher **L'utilisateur doit changer son mot de passe**
   - Cocher **Le mot de passe n'expire jamais** (en environnement de test)

### 5.3 Créer des groupes de sécurité

1. Clic droit sur l'OU > **Nouveau > Groupe**
2. Type : **Sécurité**, Étendue : **Globale**

**Groupes à créer :**
- `etudiants` (dans OU Etudiants)
- `professeurs` (dans OU Professeurs)
- `vendeurs`, `operateurs`, `dsi` (selon les services)

### 5.4 Ajouter des membres aux groupes

1. Double-clic sur le groupe
2. Onglet **Membres** > **Ajouter**
3. Rechercher et ajouter les utilisateurs concernés

### 5.5 Copier un compte utilisateur

Pour créer `eleve2` à partir de `eleve1` :
1. Clic droit sur `eleve1` > **Copier**
2. Modifier le nom et l'identifiant
3. Vérifier que les chemins de profil ont bien été mis à jour (eleve1 → eleve2)

---

## 6. Gestion des profils et dossiers personnels

### 6.1 Créer les dossiers partagés sur le serveur

```cmd
mkdir D:\donnees\profils
mkdir D:\donnees\fichiers_utilisateurs
```

Partager ces dossiers avec les droits appropriés (voir section 7).

### 6.2 Configurer les profils itinérants

1. Dans ADUC, double-clic sur un utilisateur
2. Onglet **Profil**
3. Chemin du profil : `\\AD-GELTRAM\profils\%username%`

> Le dossier est créé automatiquement à la première connexion.

### 6.3 Configurer le dossier personnel (home directory)

Dans l'onglet **Profil** du compte utilisateur :
- Sélectionner **Connecter** la lettre `Z:`
- Chemin : `\\AD-GELTRAM\utilisateurs\%username%`

> ⚠️ Retirer l'héritage du dossier parent `fichiers_utilisateurs` et donner **contrôle total** uniquement au créateur propriétaire.

---

## 7. Partage de disques et permissions NTFS

### 7.1 Préparer le stockage (RAID 0 optionnel)

Si demandé, créer un volume RAID 0 via le Gestionnaire de disques :
1. Clic droit sur l'espace non alloué > **Nouveau volume agrégé par bandes**
2. Ajouter les deux disques
3. Formater en **NTFS**, lettre `E:`

### 7.2 Créer l'arborescence

```
E:\Data
└── Geltram
    ├── Public
    ├── Utilisateurs
    └── Services
        ├── Vente
        ├── Atelier
        └── DSI
```

Via PowerShell :
```powershell
$base = "E:\Data\Geltram"
New-Item -ItemType Directory -Path "$base\Public","$base\Utilisateurs","$base\Services\Vente","$base\Services\Atelier","$base\Services\DSI" -Force
```

### 7.3 Créer les partages Windows (SMB)

**Via le Gestionnaire de serveur :**
1. Gestionnaire de serveur > **Services de fichiers > Partages**
2. Clic droit dans la zone de partages > **Nouveau partage**
3. Profil : **Partage SMB – Rapide**
4. Indiquer le chemin (ex: `E:\geltram\public`)
5. Activer **l'énumération basée sur l'accès**

### 7.4 Tableau des permissions par dossier

| Dossier | Utilisateurs | Droits |
|---|---|---|
| `Public` | Tous les utilisateurs du domaine | Contrôle total |
| `Utilisateurs` | Créateur propriétaire | Contrôle total |
| `Vente` | Groupe `vendeurs` | Lecture/Écriture |
| `Vente` | Chef de service Vente | Contrôle total |
| `Atelier` | Groupe `operateurs` | Lecture/Écriture |
| `Atelier` | Chef de service Atelier | Contrôle total |
| `DSI` | Groupe `dsi` | Lecture/Écriture |
| `DSI` | Administrateurs | Contrôle total |
| `CommunEleves` | Groupe `etudiants` | Lecture seule |
| `CommunEleves` | Groupe `professeurs` | Lecture/Écriture |
| `CommunProfs` | Groupe `professeurs` | Contrôle total |
| `CommunProfs` | Groupe `etudiants` | Aucun accès |

### 7.5 Configurer les permissions NTFS avancées

1. Clic droit sur le dossier > **Propriétés > Sécurité > Avancé**
2. **Désactiver l'héritage** si nécessaire (Convertir ou Supprimer)
3. Ajouter les entrées selon le tableau ci-dessus
4. Cliquer sur **Appliquer** puis **OK**

---

## 8. Scripts de connexion réseau

### 8.1 Emplacement des scripts

Les scripts doivent être placés dans :
```
C:\Windows\SYSVOL\domain\scripts\
```

### 8.2 Exemples de scripts

**vendeurs.bat** :
```bat
@echo off
net use U: \\AD-GELTRAM\Vente /persistent:yes
net use W: \\AD-GELTRAM\Public /persistent:yes
```

**operateurs.bat** :
```bat
@echo off
net use U: \\AD-GELTRAM\Atelier /persistent:yes
net use W: \\AD-GELTRAM\Public /persistent:yes
```

**dsi.bat** :
```bat
@echo off
net use U: \\AD-GELTRAM\DSI /persistent:yes
net use W: \\AD-GELTRAM\Public /persistent:yes
```

**scryptEleves.bat** :
```bat
@echo off
net use U: \\serveur-geltram\CommunEleves /persistent:yes
```

**scryptProfs.bat** :
```bat
@echo off
net use U: \\serveur-geltram\CommunEleves /persistent:yes
net use V: \\serveur-geltram\CommunProfs /persistent:yes
```

### 8.3 Associer le script à un utilisateur

1. ADUC > Double-clic sur l'utilisateur
2. Onglet **Profil**
3. Champ **Script d'ouverture de session** : `vendeurs.bat` (juste le nom du fichier)

---

## 9. Stratégies de groupe (GPO)

### 9.1 Ouvrir la console GPMC

Démarrer > **Gestion des stratégies de groupe**

### 9.2 GPO – Fond d'écran (exemple OU Etudiants)

1. Clic droit sur l'OU `Etudiants` > **Créer un objet GPO et le lier ici**
2. Nom : `GPO fond d'écran`
3. Clic droit > **Modifier**

**Configuration Ordinateur :**
- Préférences > Paramètres Windows > Fichiers : copier l'image de fond d'écran vers un chemin réseau partagé

**Configuration Utilisateur :**
- Stratégies > Modèles d'administration > Bureau > Bureau
  - **Papier peint du Bureau** : Activé
  - Chemin : `C:\Windows\WinSxS\amd64_microsoft-windows-shell-wallpapertheme1_...\img3.jpg`
- Stratégies > Modèles d'administration > Panneau de configuration > Personnalisation
  - **Empêcher de modifier l'arrière-plan du Bureau** : Activé

**Filtrage de sécurité :**
- Appliquer à : `Utilisateurs authentifiés`

### 9.3 GPO – Limiter la taille des profils

1. Créer une GPO sur l'OU `Professeurs` : `GPO Professeurs`
2. Modifier > Configuration utilisateur > Stratégies > Modèles d'administration > Système > **Profil des utilisateurs**
3. **Limiter la taille des profils** : Activé, valeur = `102400 Ko` (100 Mo)
4. Répéter pour l'OU `Etudiants`

### 9.4 GPO – Exclure des dossiers des profils itinérants

1. Dans la GPO concernée > Configuration utilisateur > Modèles d'administration > Système > Profil des utilisateurs
2. **Exclure les répertoires dans la liste de profils itinérants** : Activé
3. Ajouter : `Pictures` (ou `Images`)

### 9.5 GPO – Redirection de dossiers

1. Modifier la GPO > Configuration utilisateur > Stratégies > Paramètres Windows > **Redirection de dossiers**
2. Clic droit sur **Bureau** > Propriétés
3. Paramètre : **De base – Rediriger les dossiers de tout le monde vers le même emplacement**
4. Chemin : `\\192.168.0.1\fichiers_utilisateurs\%USERNAME%\Bureau`
5. Répéter pour **Documents** et **Téléchargements**

### 9.6 Appliquer et tester les GPO

Sur le poste client :
```cmd
gpupdate /force
gpresult /R
```

---

## 10. Installation et configuration d'IPAM

> ⚠️ IPAM doit être installé sur un **serveur applicatif dédié**, pas sur le DC ni le serveur DHCP.

### 10.1 Installation d'IPAM

1. Gestionnaire de serveur > **Ajouter des rôles et fonctionnalités**
2. Fonctionnalités > cocher **Serveur de gestion des adresses IP (IPAM)**
3. Valider l'ajout des outils de gestion associés
4. Installer et fermer

### 10.2 Configuration initiale d'IPAM

1. Dans le Gestionnaire de serveur, cliquer sur **IPAM** (menu gauche)
2. Cliquer sur **Configurer le serveur IPAM**
3. Base de données : **Base de données interne Windows (WID)**
4. Approvisionnement : **Basée sur une stratégie de groupe**
5. Préfixe GPO : `IPAM` (crée automatiquement IPAM_DHCP, IPAM_DNS, IPAM_DC_NPS)

### 10.3 Créer les GPO IPAM manuellement si nécessaire

Si les GPO ne se créent pas automatiquement :
```powershell
Invoke-IpamGpoProvisioning -Domain geltram.intra -GpoPrefixName IPAM -IpamServerFqdn votre-serveur-ipam.geltram.intra
```

Puis appliquer les GPO sur les serveurs gérés :
```cmd
gpupdate /force
```

### 10.4 Découverte des serveurs

1. Dans la console IPAM > **Configurer la découverte de serveurs**
2. Cliquer sur **Obtenir les forêts**, attendre la tâche de fond
3. Ajouter la forêt détectée, cocher tous les rôles
4. **Démarrer la découverte de serveurs**

### 10.5 Gérer les serveurs découverts

1. **Sélectionner ou ajouter des serveurs à gérer**
2. Clic droit sur chaque serveur > **Modifier le serveur**
3. Passer l'état à **Géré**
4. Clic droit > **Récupérer toutes les données serveurs**
5. Sur le serveur géré, forcer : `gpupdate /force` puis redémarrer

### 10.6 Utilisation d'IPAM

- **Réservation DHCP** : Étendues DHCP > clic droit sur l'étendue > Créer une réservation
- **Plage IP statique** : ESPACE D'ADRESSAGE IP > Blocs d'adresses IP > Ajouter une plage
- **Rechercher une IP disponible** : Blocs d'adresses IP > Rechercher et attribuer une adresse IP disponible

---

## 11. Intégration GLPI avec Active Directory (LDAP)

> Prérequis : GLPI installé et accessible, module PHP-LDAP installé sur le serveur GLPI.

### 11.1 Installer le module LDAP pour PHP

Sur le serveur Debian hébergeant GLPI :
```bash
apt install php-ldap
systemctl restart apache2  # ou nginx
```

### 11.2 Configurer l'annuaire LDAP dans GLPI

1. Se connecter à GLPI en **super-admin**
2. Menu **Configuration > Authentification > Annuaire LDAP**
3. Cliquer sur **+ Ajouter**
4. Cliquer sur **Active Directory** (ligne Préconfiguration)

**Remplir les champs :**

| Champ | Valeur exemple |
|---|---|
| Nom | `geltram.intra` |
| Serveur par défaut | Oui |
| Actif | Oui |
| Serveur | `192.168.0.1` (IP du DC) |
| Port | `389` (LDAP standard) |
| BaseDN | `DC=geltram,DC=intra` ou `OU=Services,DC=geltram,DC=intra` |
| DN du compte | `administrateur@geltram.intra` |
| Mot de passe | (mot de passe admin) |

5. Cliquer sur **Ajouter** – un test de connexion est automatiquement effectué

### 11.3 Importer les utilisateurs AD dans GLPI

1. Menu **Administration > Utilisateurs**
2. Cliquer sur **Liaison annuaire LDAP**
3. Cliquer sur **Importation de nouveaux utilisateurs**
4. Cliquer sur **Rechercher** (sans critère pour tout importer)
5. Cocher les utilisateurs > menu **Actions > Importer > Envoyer**

### 11.4 Importer les groupes AD

1. Menu **Administration > Groupes**
2. Cliquer sur **Liaison annuaire LDAP**
3. **Importation de nouveaux groupes** > Rechercher > Sélectionner > Importer

### 11.5 Connexion utilisateur AD à GLPI

Sur la page de connexion GLPI :
- Identifiant : nom d'utilisateur AD (ex: `j.dupont`)
- Mot de passe : mot de passe AD
- Source de connexion : sélectionner le domaine (ex: `geltram.intra`)

> Les utilisateurs importés obtiennent par défaut le profil **Self-Service** (création et suivi de tickets).

---

## 12. Intégration d'un poste client dans le domaine

### 12.1 Configurer le réseau du poste client

- IP : dynamique (DHCP) ou statique dans la plage
- DNS : `192.168.0.1` (adresse du contrôleur de domaine)

### 12.2 Intégrer le poste au domaine

1. Clic droit sur **Ce PC > Propriétés > Paramètres système avancés**
2. Onglet **Nom de l'ordinateur** > **Modifier**
3. Sélectionner **Domaine** : `geltram.intra`
4. Authentification : `Administrateur` / `Rootsio2017` (ou le mot de passe admin du domaine)
5. Redémarrer le poste

### 12.3 Vérifier la connexion au domaine

Se connecter depuis le poste avec un compte du domaine :
```
Identifiant : GELTRAM\eleve1
Mot de passe : Btssio2017
```

### 12.4 Vérifier les lecteurs réseau

Après connexion, vérifier que les lecteurs réseau sont bien montés (via les scripts de connexion) et que les droits d'accès sont respectés.

---

## 13. Commandes utiles et vérifications

### Commandes PowerShell AD

```powershell
# Lister tous les utilisateurs du domaine
Get-ADUser -Filter *

# Lister les groupes
Get-ADGroup -Filter *

# Vérifier les membres d'un groupe
Get-ADGroupMember -Identity "etudiants"

# Créer un utilisateur en PowerShell
New-ADUser -Name "Jean Dupont" -SamAccountName "j.dupont" -AccountPassword (ConvertTo-SecureString "Mdp2024!" -AsPlainText -Force) -Enabled $true -Path "OU=Etudiants,DC=geltram,DC=intra"

# Ajouter un utilisateur à un groupe
Add-ADGroupMember -Identity "etudiants" -Members "j.dupont"
```

### Commandes DNS

```cmd
# Tester la résolution
nslookup web.geltram.intra
nslookup 192.168.0.10

# Vider le cache DNS du serveur
dnscmd /clearcache

# Vider le cache DNS client
ipconfig /flushdns
```

### Commandes GPO

```cmd
# Forcer l'application des GPO
gpupdate /force

# Vérifier les GPO appliquées
gpresult /R

# Rapport GPO détaillé en HTML
gpresult /H C:\rapport_gpo.html
```

### Vérifications générales

```cmd
# Vérifier la connectivité au DC
ping 192.168.0.1

# Vérifier l'appartenance au domaine
whoami /all

# Tester l'authentification Kerberos
klist

# Vérifier les partages réseau
net use

# Tester un partage
net use Z: \\AD-GELTRAM\Public
```

### Diagnostics AD

```cmd
# Vérifier la santé du contrôleur de domaine
dcdiag /test:services
dcdiag /test:replications

# Répliquer AD entre DC
repadmin /syncall
```

---

## 14. Checklist finale de validation

### Infrastructure de base
- [ ] Serveur Windows Server 2022 installé et nommé
- [ ] Adresse IP statique configurée
- [ ] Rôle AD DS installé
- [ ] Domaine créé et serveur promu contrôleur de domaine
- [ ] Serveur DNS opérationnel
- [ ] Zone de recherche directe créée
- [ ] Zone de recherche inversée créée
- [ ] Enregistrement(s) A créé(s)
- [ ] Serveur DHCP installé et étendue configurée

### Utilisateurs et groupes
- [ ] Unités d'organisation créées
- [ ] Utilisateurs créés dans les bonnes OU
- [ ] Groupes de sécurité créés
- [ ] Utilisateurs ajoutés aux bons groupes

### Partages et permissions
- [ ] Arborescence de dossiers créée
- [ ] Partages SMB configurés
- [ ] Permissions NTFS appliquées correctement
- [ ] Héritage désactivé là où nécessaire
- [ ] Test d'accès réalisé avec différents comptes

### Profils et scripts
- [ ] Dossier de profils partagé
- [ ] Chemin de profil itinérant configuré dans les comptes
- [ ] Dossier personnel (home) configuré
- [ ] Scripts .bat créés dans SYSVOL\domain\scripts
- [ ] Scripts associés aux comptes utilisateurs

### GPO
- [ ] GPO fond d'écran créée et liée à l'OU
- [ ] GPO limite taille profil configurée
- [ ] GPO exclusion dossiers profils itinérants configurée
- [ ] GPO redirection dossiers configurée (optionnel)
- [ ] Test GPO avec `gpupdate /force` et `gpresult /R`

### Client
- [ ] Poste client intégré au domaine
- [ ] Connexion avec comptes du domaine testée
- [ ] Lecteurs réseau montés et vérifiés
- [ ] Droits d'accès vérifiés pour chaque profil

### IPAM (si requis)
- [ ] IPAM installé sur serveur dédié
- [ ] GPO IPAM créées
- [ ] Serveurs découverts et gérés
- [ ] Données récupérées

### GLPI (si requis)
- [ ] Module php-ldap installé
- [ ] Annuaire LDAP configuré dans GLPI
- [ ] Test de connexion AD réussi
- [ ] Utilisateurs importés
- [ ] Groupes importés

---

## 💡 Conseils supplémentaires pour le devoir

### Erreurs courantes à éviter

1. **Mot de passe trop simple lors de la création du domaine** → utiliser un mot de passe complexe dès le départ
2. **DNS mal configuré** → toujours vérifier que le serveur pointe sur lui-même comme DNS avant la promotion AD
3. **GPO non appliquées** → toujours tester avec `gpupdate /force` et vérifier avec `gpresult /R`
4. **IPAM sur le DHCP** → IPAM doit être sur un serveur séparé
5. **Oublier le redémarrage** → après la promotion en DC, le redémarrage est obligatoire
6. **Partages SMB vs NTFS** → les deux types de permissions s'additionnent ; c'est la plus restrictive qui s'applique

### Ordre logique de réalisation

```
1. Installer Windows Server → Nommer le serveur → IP statique
2. Installer AD DS → Promouvoir en DC → Redémarrer
3. Configurer DNS (zones + enregistrements)
4. Installer et configurer DHCP
5. Créer OU → Groupes → Utilisateurs
6. Créer les dossiers et partages réseau
7. Configurer les permissions NTFS
8. Créer les scripts de connexion
9. Associer scripts et profils aux comptes
10. Créer et tester les GPO
11. Intégrer les postes clients au domaine
12. Tester avec différents comptes
```

### Informations de contexte utiles à noter pendant le devoir

- Nom du domaine : `_____________________`
- Nom NETBIOS : `_____________________`
- IP du contrôleur de domaine : `_____________________`
- IP du serveur DHCP : `_____________________`
- Plage DHCP : `_____________________` à `_____________________`
- Mot de passe admin domaine : `_____________________` *(à garder confidentiel)*
- Mot de passe DSRM : `_____________________`
- Chemin partages : `_____________________`

---

*Documentation réalisée à partir des ressources : IT-Connect (IPAM, AD), BTS SIO Bloc 3 TP1, Neptunet.fr (GLPI-AD), BTS SIO Bloc B2 (Partage de disques)*
