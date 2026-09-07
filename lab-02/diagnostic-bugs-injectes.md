# Bug 01 : Compte expiré puis verrouillé

### Contexte
Poste client PC1 (Windows11), joint au domaine asso.lab.
Utilisateur test (noms fictifs): Amine Adad (OU Direction).
Tentative de connexion avec identifiant et mot de passe corrects.

### Symptôme 1
Message d'erreur à l'ouverture de session : "votre compte est expiré ».

### Diagnostic 1
Vérification dans Utilisateurs et ordinateurs Active Directory → propriétés de l'utilisateur → onglet Compte.
Constat : la date d'expiration du compte est fixée à aujourd'hui.

### Correction 1
Date d'expiration repositionnée sur « jamais ».
Nouvelle tentative de connexion avec le même identifiant et mot de passe.

### Symptôme 2
Message d'erreur : « ce compte est verrouillé ».

### Diagnostic 2
Retour dans les propriétés du compte, onglet Compte : la case « Le compte est verrouillé » est cochée.

### Correction 2
Case décochée pour déverrouiller le compte.
Connexion testée à nouveau et réussie.

### Ce que j'en retiens
Les deux symptômes ont été découverts l'un après l'autre, ce qui a doublé le temps de diagnostic. Une seule commande aurait affiché les deux états en même temps :
Get-ADUser -Identity aadad -Properties LockedOut, AccountExpirationDate, Enabled | Select LockedOut, AccountExpirationDate, Enabled

Autre point à retenir, indépendant de la méthode de diagnostic : positionner l'expiration sur « jamais » résout le lab, mais ce n'est pas une bonne pratique en production — un compte sans date d'expiration reste une surface d'attaque ouverte indéfiniment. Dans un vrai contexte, la bonne réponse aurait été de fixer une nouvelle date d'expiration cohérente avec une politique de renouvellement, pas de supprimer la contrainte.

Enfin, une politique de notification avant expiration (ou au minimum un contrôle régulier des dates à venir) aurait évité l'incident plutôt que de le corriger après coup.



# Bug 02 : Problème de DNS

### Contexte
Poste client PC1 (Windows 11), joint au domaine asso.lab.
Page d'authentification de l'utilisateur2 : identifiant et mot de passe corrects renseignés.

### Symptôme
Message d'erreur : il semble que l'utilisateur ne soit pas connecté au domaine.

### Diagnostic
**Hypothèse 1** — Vérifier si le blocage est spécifique à utilisateur2 ou général à PC1 : connexion testée avec utilisateur1 (compte de domaine, déjà connecté une fois sur PC1 — voir Bug 01) et avec le compte admin local. Les deux réussissent.
Hypothèse écartée à tort dans un premier temps : le compte admin local ne passe jamais par le domaine, ce test ne prouve rien sur l'état du réseau. La réussite d'utilisateur1 s'explique par les **identifiants de domaine mis en cache** — Windows conserve localement la dernière authentification réussie d'un utilisateur sur un poste, ce qui permet une reconnexion même si le contrôleur de domaine est injoignable. Utilisateur2, jamais connecté sur PC1 auparavant, ne disposait d'aucun cache et devait obligatoirement joindre le contrôleur — ce qui a révélé le vrai problème.

**Hypothèse 2** — Vérifier si utilisateur2 est correctement renseigné et actif dans l'AD : propriétés du compte (onglets Compte et Sessions) contrôlées, rien d'anormal.

**Hypothèse 3** — Vérifier si PC1 est autorisé à authentifier différents utilisateurs : propriétés de l'objet PC1 dans le dossier Computers (onglets « Membre de » et « Délégation ») contrôlées, rien d'anormal.

**Hypothèse 4** — Redescendre à la couche réseau : ping bidirectionnel PC1 ↔ AD depuis le poste d'utilisateur1, puis `ipconfig /all` depuis PC1.
Constat : DNS configuré sur `10.1.1.1` au lieu de `10.0.2.10`.

### Correction
DNS de PC1 reconfiguré sur `10.0.2.10`.
Vérification : ping bidirectionnel OK, `ipconfig /all` conforme.
Connexion testée avec succès depuis la page d'authentification d'utilisateur2, puis `ipconfig /all` confirmé depuis son poste.

### Ce que j'en retiens
Un test de connexion réussi avec un autre compte ne prouve pas l'absence de problème réseau si ce compte s'est déjà authentifié avant sur le même poste — les identifiants mis en cache masquent un DNS cassé. Pour un test fiable d'accès réseau, utiliser un compte qui ne s'est jamais connecté sur ce poste, ou vérifier directement la couche réseau (ping, `ipconfig /all`) avant de conclure à partir d'un test de connexion.
Réflexe général : penser à la couche réseau avant la couche applicative face à un problème d'authentification.



# Bug 03 : Relation d'approbation rompue

### Contexte
Poste client PC1 (Windows 11), joint au domaine asso.lab.
Page d'authentification de l'utilisateur3 : identifiant et mot de passe corrects renseignés.

### Symptôme
Blocage de la connexion avec le message : « La relation d'approbation entre cette station de travail et le domaine principal a échoué. »

### Diagnostic
**Hypothèse 1** — Vérifier une éventuelle relation de confiance rompue entre domaines/forêts via « Domaines et approbations Active Directory ». Rien de significatif trouvé.
Piste écartée à raison, mais pour la mauvaise interprétation au départ : cet outil gère les approbations **entre domaines/forêts**, pas le lien de confiance entre **un poste et son domaine** (appelé canal sécurisé). Le message d'erreur, malgré son vocabulaire proche, concernait ce second mécanisme, hors du périmètre de cet outil.

**Hypothèse 2** — Vérification des propriétés du compte utilisateur3. Rien d'anormal trouvé.

**Recherche documentaire** — Message d'erreur recherché en ligne. Source consultée : [IT-Connect — Comment corriger l'erreur de relation d'approbation](https://www.it-connect.fr/windows-comment-corriger-erreur-de-relation-approbation-voici-plusieurs-methodes/).

### Correction
Connexion avec le compte administrateur local. Poste PC1 sorti du domaine, puis réinséré (Système → Membre d'un groupe de travail, appliqué, puis remis sur le domaine).
Cause : lien de confiance (canal sécurisé) rompu entre PC1 et le contrôleur de domaine.

### Ce que j'en retiens
Distinguer clairement deux notions qui portent le même nom en français : la relation d'approbation **entre domaines/forêts** (outil dédié, sans lien avec ce cas), et le **canal sécurisé** entre un poste et son domaine (la vraie cause ici) — une confusion de vocabulaire qui a coûté du temps de diagnostic.
Méthode alternative plus rapide pour la prochaine fois, sans sortir le poste du domaine : `Test-ComputerSecureChannel -Repair`, qui répare directement ce lien en une commande.
La veille documentaire ciblée (chercher le message d'erreur exact) a permis de débloquer la situation efficacement — un réflexe à garder, pas à percevoir comme un aveu de faiblesse.




# Bug 04 — Problème de chargement du profil utilisateur

### Contexte
Poste client PC1 (Windows 11), joint au domaine asso.lab.
Authentification de l'utilisateur berengere.bertrand (OU TSociaux) réussie (identifiant + mot de passe corrects).

### Symptôme
Message d'erreur au démarrage de la session : « Nous ne pouvons pas nous connecter à votre compte ».

### Diagnostic
Recherche documentaire sur le message d'erreur exact. Source consultée : [Malekal — Nous ne pouvons pas nous connecter à votre compte](https://www.malekal.com/windows-10-nous-ne-pouvons-pas-nous-connecter-a-votre-compte/).

Procédure suivie :
- Vérification de l'ouverture avec un profil temporaire (message d'avertissement habituel sur le bureau) : absent.
- Vérification des profils existants dans `C:\Users`, recherche d'un dossier temporaire : aucun trouvé.
- Vérification du nom exact du dossier de profil : `bertrandberengere.mod` — suffixe anormal identifié.

### Correction
Suppression du suffixe `.mod` sur le nom du dossier de profil.
Redémarrage du poste, authentification réussie, session utilisable sans message d'erreur.
Vérification finale sur `C:\Users` : nom de dossier conforme.

### Ce que j'en retiens
Le renommage du dossier a suffi ici, mais ce n'est pas la garantie générale : Windows associe un profil à un utilisateur via le registre (`ProfileList`, chemin `ProfileImagePath` lié au SID), pas seulement par le nom du dossier. Si un renommage seul ne résout pas un incident similaire à l'avenir, vérifier et corriger cette clé de registre est l'étape suivante.
La documentation technique ciblée (rechercher le message d'erreur exact plutôt que deviner) reste le réflexe le plus efficace face à un symptôme inconnu.
