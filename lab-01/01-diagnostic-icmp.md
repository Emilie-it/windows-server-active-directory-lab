# Diagnostic 01 — ARP OK, ping KO entre deux VM

## Contexte

L'incident apparaît avant la jonction du poste client au domaine Active Directory.

Les deux machines sont connectées au même réseau interne VirtualBox :

- `SRV-ASSO-DC1` : `192.168.10.10/24`
- `PC-TS1` : `192.168.10.20/24`

Le DHCP est désactivé et les adresses IPv4 sont configurées manuellement.

## Symptôme

Le ping entre les deux machines échoue avec 100 % de perte.

La commande :

`arp -a`

permet cependant de constater la présence de la machine distante dans la table ARP.

Cela indique que les machines communiquent sur le réseau local, mais que le trafic ICMP n'aboutit pas.

## Diagnostic — PC-TS1 vers SRV-ASSO-DC1

La recherche de la cause a été réalisée par élimination des différents profils du pare-feu Windows.

### Test 1 — Profil Domaine

Désactivation temporaire du pare-feu pour le profil Domaine.

**Résultat :** ping toujours KO.

Le profil Domaine n'était pas actif sur cette interface.

### Test 2 — Profil Privé

Désactivation temporaire du pare-feu pour le profil Privé.

**Résultat :** ping toujours KO.

Le profil Privé n'était pas actif.

### Test 3 — Profil Public

Désactivation temporaire du pare-feu pour le profil Public.

**Résultat :** ping OK.

Le filtrage du profil Public est donc identifié comme la cause du blocage ICMP entrant.

## Correction

La désactivation du pare-feu a uniquement servi de test de diagnostic.
Elle n'a pas été conservée comme solution.

Le pare-feu Windows a été maintenu actif et la règle entrante prédéfinie :

`Demande d'écho — ICMPv4 entrant`

a été activée.

La règle a été vérifiée pour les profils :

- Domaine
- Privé
- Public

**Résultat :**

`PC-TS1 → SRV-ASSO-DC1 : ping OK`

## Vérification dans le sens inverse

Le test a ensuite été effectué depuis le serveur vers le client :

`SRV-ASSO-DC1 → PC-TS1`

**Résultat initial :** ping KO.

Le client bloquait lui aussi les requêtes ICMP entrantes.

Après activation de la règle ICMPv4 entrante sur le client :

**Résultat :** ping OK.

## État final

- PC-TS1 → SRV-ASSO-DC1 : OK
- SRV-ASSO-DC1 → PC-TS1 : OK
- Pare-feu Windows serveur : actif
- Pare-feu Windows client : actif

La communication réseau bidirectionnelle est validée avant la poursuite du lab.

## Limites de ma démarche

Lors de ce premier diagnostic, j'ai identifié le profil concerné en testant successivement les différents profils du pare-feu.

Avec le recul, une méthode plus efficace aurait été de commencer par identifier directement le profil réseau actif avec :

`Get-NetConnectionProfile`

J'aurais ainsi pu cibler immédiatement le bon profil avant de poursuivre le diagnostic.

De même, le poste n'étant pas encore joint au domaine, l'hypothèse du profil Domaine pouvait être écartée plus tôt.

## Ce que j'en retiens

La désactivation d'un pare-feu peut être utilisée ponctuellement dans un environnement de laboratoire pour isoler une cause, mais elle ne constitue pas une correction.

La démarche à privilégier est :

1. identifier le profil réseau actif ;
2. identifier précisément le trafic bloqué ;
3. appliquer la règle la plus ciblée possible ;
4. maintenir les protections actives ;
5. tester la communication dans les deux sens.

### ARP et ICMP

La présence d'une entrée ARP ne garantit pas qu'un ping fonctionnera.

ARP permet ici de confirmer que les machines peuvent se découvrir sur le réseau local.

Le ping utilise ICMP et peut être filtré indépendamment par le pare-feu Windows.

**À retenir : ARP fonctionnel ≠ ICMP autorisé.**
