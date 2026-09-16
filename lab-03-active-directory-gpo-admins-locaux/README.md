# Gestion centralisée des administrateurs locaux par GPO

## Objectif

Mettre en place une stratégie de groupe afin de gérer de manière centralisée les droits d'administrateur local sur les postes clients placés dans l'OU `asso1` et ses sous-OU.

Les utilisateurs autorisés sont regroupés dans un groupe de sécurité Active Directory nommé `GG_AdminsLocauxPC`.

La GPO doit permettre d'ajouter ce groupe Active Directory au groupe local `Administrateurs` des postes concernés.

---

## Environnement

- Domaine DNS : `asso.lab`
- Nom NetBIOS du domaine : `ASSO`
- Contrôleur de domaine : `SRV-DC01`
- Poste client : `PC1`
- OU principale : `asso1`
- OU des utilisateurs : `IT`
- OU des postes clients : `PC Clients`
- Groupe de sécurité : `GG_AdminsLocauxPC`
- Étendue finale du groupe : `Domaine local`

---

## Architecture

```text
Domaine : asso.lab
│
└── OU asso1
    │
    ├── OU IT
    │   ├── Chantal Carbone
    │   ├── Daniel Date
    │   ├── Fanny Fanon
    │   └── Gregory Grougo
    │
    └── OU PC Clients
        └── PC1
```

Les quatre utilisateurs sont membres du groupe Active Directory `GG_AdminsLocauxPC`.

```text
Chantal ──┐
Daniel  ──┤
Fanny   ──┼──► GG_AdminsLocauxPC
Gregory ──┘
```

La GPO `Administrateurs locaux des postes clients` est liée à l'OU `asso1`.

Son rôle est de configurer les postes concernés afin que le groupe Active Directory `ASSO\GG_AdminsLocauxPC` soit membre de leur groupe local `Administrateurs`.

```text
GPO liée à asso1
       │
       ▼
Postes concernés
       │
       ▼
Groupe local Administrateurs
       │
       └── ASSO\GG_AdminsLocauxPC
                    │
                    └── utilisateurs autorisés
```

---

## Mise en œuvre initiale

### 1. Création du groupe de sécurité

Création d'un groupe de sécurité Active Directory nommé `GG_AdminsLocauxPC`.

Ce groupe permet de regrouper les utilisateurs auxquels doivent être attribués des droits d'administrateur local sur les postes concernés.

L'étendue du groupe a été corrigée au cours du dépannage pour devenir **Domaine local**.

![Groupe de sécurité Active Directory](images/01-groupe-ad-domaine-local.png)

---

### 2. Création et ajout des utilisateurs

Création d'une sous-OU nommée `IT` dans l'OU `asso1`.

Quatre comptes utilisateurs ont été créés dans cette OU :

- Chantal Carbone
- Daniel Date
- Fanny Fanon
- Gregory Grougo

Ces utilisateurs ont ensuite été ajoutés comme membres du groupe `GG_AdminsLocauxPC`.

---

### 3. Création et liaison de la GPO

Dans la console **Gestion de stratégie de groupe**, création de la GPO :

`Administrateurs locaux des postes clients`

La GPO est ensuite liée à l'OU `asso1`.

Le lien avec `asso1` permet aux objets ordinateurs présents dans cette OU et ses sous-OU de se trouver dans le périmètre où cette GPO peut s'appliquer, sous réserve de l'héritage et du filtrage.

---

## Premier test : échec

J'ai effectué une connexion sur `PC1` avec le compte utilisateur de Fanny Fanon.

Dans les paramètres du compte Windows, le statut d'administrateur n'était pas indiqué.

Pour vérifier directement les membres du groupe local `Administrateurs`, j'ai exécuté :

```cmd
net localgroup Administrateurs
```

Le groupe `ASSO\GG_AdminsLocauxPC` n'apparaissait pas parmi les membres du groupe local `Administrateurs`.

Le résultat attendu n'était donc pas obtenu sur `PC1`.

![Échec initial de la vérification](images/02-echec-groupe-administrateurs.png)

---

## Diagnostic et résolution

### 1. Configuration incomplète de la GPO

Je suis retournée dans la gestion des stratégies de groupe afin de vérifier la configuration.

Je me suis également appuyée sur une ressource IT-Connect afin de comparer ma procédure avec une configuration fonctionnelle.

J'ai constaté que la GPO avait été créée et liée, mais que la préférence permettant de modifier le groupe local `Administrateurs` n'avait pas été configurée.

J'ai donc configuré la préférence suivante :

```text
Configuration ordinateur
        │
        ▼
Préférences
        │
        ▼
Paramètres du Panneau de configuration
        │
        ▼
Utilisateurs et groupes locaux
        │
        ▼
Groupe local : Administrateurs
```

Paramètres utilisés :

- Action : `Mettre à jour`
- Groupe local : `Administrateurs`
- Membre ajouté : `ASSO\GG_AdminsLocauxPC`

La GPO peut ainsi modifier l'appartenance au groupe local `Administrateurs` sur les ordinateurs auxquels elle s'applique.

---

### 2. PC1 situé hors du périmètre de la GPO

J'ai également constaté que `PC1` se trouvait initialement dans le conteneur Active Directory par défaut `Computers`.

`Computers` est un **conteneur Active Directory** et non une OU située sous `asso1`.

La situation initiale était donc :

```text
asso.lab
│
├── Computers
│   └── PC1
│
└── asso1
    └── GPO liée ici
```

`PC1` ne se trouvait donc pas dans la branche Active Directory où ma GPO était liée.

J'ai créé l'OU `PC Clients` sous `asso1`, puis déplacé l'objet ordinateur `PC1` dans cette OU.

```text
asso.lab
│
└── asso1
    │
    ├── GPO liée ici
    │
    └── PC Clients
        └── PC1
```

`PC1` se trouvait désormais dans le périmètre où il pouvait hériter de la GPO.

---

### 3. Correction de l'étendue du groupe et SID obsolète

Le groupe `GG_AdminsLocauxPC` avait initialement été créé avec une étendue **Global**.

Après révision de la configuration, je l'ai supprimé puis recréé avec une étendue **Domaine local**.

Après cette recréation, une entrée sous forme de SID non résolu est apparue dans le filtrage de sécurité de la GPO.

![SID obsolète dans la GPO](images/03-sid-obsolete-gpo.png)

Pour identifier l'origine de cette entrée, j'ai vérifié le SID du groupe actuellement présent dans Active Directory avec PowerShell :

```powershell
Get-ADGroup "GG_AdminsLocauxPC" -Properties SID | Select-Object Name, SID
```

Le SID du groupe recréé était différent de celui encore référencé dans la GPO.

![Vérification du SID du groupe](images/04-verification-sid-groupe.png)

La suppression puis la recréation d'un objet Active Directory avec le même nom ne recrée donc pas le même objet de sécurité : un nouveau SID lui est attribué.

La référence correspondant à l'ancien SID a été supprimée de la configuration de la GPO.

---

## Application et vérifications

Après correction de la configuration, j'ai actualisé les stratégies de groupe sur `PC1` avec :

```cmd
gpupdate /force
```

Puis j'ai vérifié à nouveau les membres du groupe local `Administrateurs`.

Une première vérification graphique dans la gestion de l'ordinateur montre désormais le groupe :

`ASSO\GG_AdminsLocauxPC`

comme membre du groupe local `Administrateurs`.

![Vérification graphique du groupe Administrateurs](images/05-groupe-local-administrateurs.png)

J'ai ensuite confirmé le résultat en ligne de commande :

```cmd
net localgroup Administrateurs
```

Le groupe `ASSO\GG_AdminsLocauxPC` apparaît désormais parmi les membres.

![Vérification finale avec net localgroup](images/06-verification-finale-net-localgroup.png)

Le résultat attendu est donc obtenu.

---

## Résultat

La GPO permet désormais d'ajouter le groupe de sécurité Active Directory `ASSO\GG_AdminsLocauxPC` au groupe local `Administrateurs` des postes concernés situés sous `asso1`.

La gestion des droits peut ainsi être centralisée :

```text
Administrateur AD
       │
       ▼
modifie les membres de
GG_AdminsLocauxPC
       │
       ▼
GPO
       │
       ▼
Administrateurs local des postes
       │
       ▼
droits administrateur local
```

Il n'est donc plus nécessaire d'ajouter individuellement chaque utilisateur au groupe local `Administrateurs` de chaque poste.

---

## Ce que j'ai appris

- J'ai appris que l'organisation des OU et des groupes est déterminante pour gérer efficacement les utilisateurs et les ordinateurs.
- J'ai compris qu'une GPO contient des paramètres de configuration et non des utilisateurs ou des membres.
- Lors du dépannage, j'ai découvert que la configuration des préférences est distincte du filtrage de sécurité : les préférences définissent l'action à réaliser, tandis que le filtrage participe à déterminer quels objets peuvent appliquer la GPO.
- J'ai découvert qu'un objet Active Directory supprimé puis recréé avec le même nom possède un nouveau SID.
- Je dois encore approfondir les notions de groupes Active Directory, de GPO, de préférences et de périmètre d'application des stratégies.

---

## Ressources

- IT-Connect — GPO : définir un utilisateur administrateur local sur les postes Windows  
  https://www.it-connect.fr/gpo-definir-un-utilisateur-administrateur-local-de-tous-les-pcs/
