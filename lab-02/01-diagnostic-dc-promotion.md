# lab-02/diagnostic-dc-promotion.md

## Diagnostic 02 — Échecs dcdiag après promotion du contrôleur de domaine

### Contexte
`SRV-DC01` vient d'être promu en contrôleur de domaine (forêt `asso.lab`). Une série de vérifications post-promotion est effectuée avant la jonction du premier poste client.

### Symptôme
`Get-DnsClientServerAddress -AddressFamily IPv4` renvoie `127.0.0.1` comme serveur DNS, alors que l'IP statique configurée est `10.0.2.10`.

`dcdiag` échoue sur le test `Connectivity`, avec l'erreur : l'adresse IP de l'hôte `<GUID>._msdcs.asso.lab` n'a pas pu être résolue.

### Diagnostic
Comparaison entre la configuration attendue et la configuration réelle via `ipconfig /all` : confirmation que le DNS de la carte réseau pointait sur `127.0.0.1` au lieu de `10.0.2.10`.

Correction : reconfiguration manuelle du DNS de la carte réseau sur `10.0.2.10`.

Le test `Connectivity` échouant encore après cette correction, la chaîne de résolution DNS a été vérifiée maillon par maillon :

1. `nslookup -type=CNAME <GUID>._msdcs.asso.lab` → renvoie `srv-dc01.asso.lab` : l'alias CNAME existe et pointe correctement.
2. `nslookup srv-dc01.asso.lab` → renvoie `10.0.2.10` : la résolution nom → IP fonctionne.
3. `Get-Service Netlogon` → `Running` : le service responsable de l'enregistrement DNS automatique du contrôleur fonctionne normalement.

Hypothèse retenue : l'échec initial de `dcdiag` était lié à la fraîcheur du contrôleur de domaine, qui n'avait pas encore terminé de propager l'ensemble de ses enregistrements DNS au moment du premier test.

### Vérification
Relance de `dcdiag` : le test `Connectivity` réussit. L'hypothèse est confirmée, pas supposée.

Le test `SystemLog` échoue toujours, pour deux causes distinctes identifiées dans les journaux d'événements :

- Échecs de mise à jour Secure Boot (DBX, SBAT) : liés à l'absence de Secure Boot sur la VM VirtualBox — limitation connue de l'environnement de virtualisation, sans impact sur le fonctionnement du lab.
- Avertissement d'extinction inattendue de la VM : correction comportementale à appliquer (arrêt propre du système invité avant fermeture de VirtualBox).

Le test `DFSREvent` réussit mais signale des avertissements récents sur le partage SYSVOL, à surveiller lors des prochains contrôles.

### État final
- `dcdiag` — `Connectivity` : OK
- `dcdiag` — `SystemLog` : échec (causes identifiées, l'une acceptée comme limite du lab, l'autre corrigée par changement d'habitude)
- `dcdiag` — `DFSREvent` : OK, sous surveillance

### Ce que j'en retiens
Un échec `dcdiag` au tout début de la vie d'un contrôleur de domaine n'est pas nécessairement un vrai problème — certains services ont besoin de temps pour terminer leur propagation. Mais un échec non expliqué ne doit jamais être requalifié en "normal" sans être revérifié avec une preuve concrète.
