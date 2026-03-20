# BTS SIO – Bloc B1
## B1_TP5 – Restauration du système sous Windows *(1h30)*

---

## Introduction

Dans ces travaux pratiques, vous allez **créer un point de restauration** et l'utiliser pour **restaurer votre ordinateur**.

---

## Équipements recommandés

- Un ordinateur équipé de **Windows 10** ou la **VM Windows 10 Light**

---

## Étape 1 : Créer un point de restauration

**a.** Cliquez sur **Panneau de configuration > Récupération**.

**b.** Sélectionnez **Configurer la Restauration du système** dans la fenêtre *Récupération*.

**c.** Cliquez sur l'onglet **Protection du système** dans la fenêtre *Propriétés système*, puis cliquez sur **Créer**.

**d.** Dans le champ de description *Créer un point de restauration*, tapez :
```
Application installée
```
Cliquez sur **Créer**.

**e.** Une fenêtre *Protection système* confirme la création du point de restauration. Cliquez sur **Fermer**.

**f.** Cliquez sur **OK** pour fermer la fenêtre *Propriétés système*.

---

## Étape 2 : Utiliser l'utilitaire Restauration du système

**a.** Dans la fenêtre *Récupération*, cliquez sur **Ouvrir la Restauration du système**.

**b.** La fenêtre *Restauration du système* s'affiche. Cliquez sur **Suivant**.

**c.** La liste des points de restauration s'affiche.

> ❓ **Question :** Quel type de point de restauration avez-vous créé à l'étape 1 ?
>
> **Réponse :** _Le type est **Manuelle** (point créé manuellement par l'utilisateur)._

**d.** Fermez toutes les fenêtres ouvertes.

---

## Étape 5 : Créer un nouveau document dans le dossier Documents

1. Ouvrez le **Bloc-notes** : `Démarrer > taper "Bloc-notes" > Entrée`
2. Tapez le texte suivant :
```
Ceci est un test de point de restauration
```
3. Cliquez sur **Fichier > Enregistrer sous**
4. Dans la fenêtre *Enregistrer sous*, naviguez vers **Documents** et entrez le nom de fichier :
```
Fichier de test du point de restauration
```
5. Cliquez sur **Enregistrer**, puis fermez le Bloc-notes.

---

## Étape 6 : Installer un nouveau logiciel

1. Téléchargez et installez un logiciel, par exemple **7-Zip** ou **Paint.NET**
2. Vérifiez son bon fonctionnement
3. Créez un nouveau point de restauration nommé :
```
save2
```

---

## Étape 7 : Restaurer l'ordinateur au point de restauration de l'étape 1

1. Allez dans **Panneau de configuration > Récupération > Ouvrir la Restauration du système**
2. Sélectionnez le point de restauration **Application installée** (créé à l'étape 1)
3. Cliquez sur **Suivant**
4. Dans la fenêtre *Confirmer le point de restauration*, cliquez sur **Terminer**
5. Un avertissement s'affiche indiquant que le processus ne peut pas être interrompu. Cliquez sur **Oui** pour continuer.

> ⚠️ **Remarque :** Windows redémarre automatiquement pour terminer la restauration.

---

## Étape 8 : Vérifier que la restauration s'est bien déroulée

1. Ouvrez une session sur l'ordinateur si nécessaire
2. La fenêtre *Restauration du système* s'ouvre et confirme le succès de l'opération. Cliquez sur **Fermer**.

> 💡 **Remarque :** Sous Windows 10, il peut être nécessaire de cliquer sur la vignette **Bureau** après le redémarrage pour voir le message de confirmation.

### Questions de vérification

> ❓ **Les nouveaux logiciels installés sont-ils présents et fonctionnels ?**
>
> **Réponse :** _Non, ils ont été supprimés car la restauration ramène le système à un état antérieur à leur installation._

> ❓ **Le fichier `Fichier de test du point de restauration.txt` se trouve-t-il dans le dossier Documents ? Son contenu est-il identique ? Pourquoi ?**
>
> **Réponse :** _Oui, le fichier est toujours présent et son contenu est identique. La restauration du système **n'affecte pas les documents personnels**, elle ne restaure que les fichiers système, les paramètres et les applications._

---

## Remarques générales

> ❓ **Quand est-il judicieux de créer un point de restauration manuel ? Pourquoi ?**

Il est recommandé de créer un point de restauration manuel dans les situations suivantes :

- **Avant d'installer un nouveau logiciel** : en cas de dysfonctionnement ou d'incompatibilité, on peut revenir à l'état précédent.
- **Avant de modifier le registre Windows** : les erreurs dans le registre peuvent rendre le système instable.
- **Avant une mise à jour importante** : pilotes, mises à jour système, etc.
- **Avant une modification de configuration système** : changement de paramètres critiques.

Cela permet de **revenir rapidement à un état stable** sans avoir à réinstaller le système d'exploitation.

---

*Document basé sur le TP B1_TP5 – Restauration du système sous Windows – BTS SIO Bloc B1*
