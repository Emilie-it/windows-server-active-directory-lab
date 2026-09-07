# Bug 01 : Compte expiré puis vérouillé

### Contexte
Poste client PC14 (Windows11), joint au domaine asso.lab.
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

## Bug 02 : Problème de DNS

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
Re
