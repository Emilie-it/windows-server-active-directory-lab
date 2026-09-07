# Lab 01 — Infrastructure Windows Server & Active Directory

## Objectif

Mettre en place un premier environnement Active Directory virtualisé afin de pratiquer :

- la configuration réseau d'un serveur et d'un poste client ;
- l'installation d'Active Directory Domain Services et DNS ;
- la création d'un domaine ;
- la gestion des utilisateurs et des unités d'organisation ;
- l'intégration d'un poste Windows au domaine ;
- la création et la vérification de stratégies de groupe (GPO) ;
- le diagnostic d'incidents réseau et Active Directory.

## Architecture

| Machine | Rôle | Système | Adresse IPv4 | DNS |
|---|---|---|---|---|
| SRV-ASSO-DC1 | Contrôleur de domaine / DNS | Windows Server 2025 | 192.168.10.10/24 | 192.168.10.10 |
| PC-TS1 | Poste client | Windows 11 Pro | 192.168.10.20/24 | 192.168.10.10 |

**Hyperviseur :** VirtualBox  
**Réseau :** réseau interne `Asso-lan`  
**Domaine Active Directory :** `asso.lan`  
**Nom NetBIOS :** `ASSO`  
**DHCP :** désactivé

## Mise en place du contrôleur de domaine

- Installation du rôle AD DS
- Installation et configuration du DNS
- Promotion de `SRV-ASSO-DC1` en contrôleur de domaine
- Création d'une nouvelle forêt `asso.lan`

Avant la promotion, `Get-ADDomain` ne retournait aucun domaine actif.

Après la promotion, la commande permettait notamment de vérifier :

- `DNSRoot : asso.lan`
- `NetBIOSName : ASSO`
- `Forest : asso.lan`
- `DomainMode : Windows2025Domain`
- contrôleur de domaine : `SRV-ASSO-DC1.asso.lan`

## Organisation Active Directory

Création d'une OU parente :

`Champigny`

Puis de quatre OU correspondant aux différents services :

- Direction
- IT
- Secrétariat
- TS

Le poste `PC-TS1` a ensuite été déplacé dans l'OU `TS`.

Des comptes utilisateurs de test ont également été créés puis contrôlés dans Active Directory.

## Jonction du poste client

Avant la jonction, la commande :

`ipconfig /all`

a permis de vérifier que `PC-TS1` utilisait uniquement `192.168.10.10` comme serveur DNS.

Le poste a ensuite été joint au domaine `asso.lan`.

Après redémarrage :

- le suffixe DNS principal était `asso.lan` ;
- l'ouverture de session avec un compte du domaine fonctionnait ;
- `whoami` permettait de confirmer que l'authentification était effectuée par le domaine.

## Stratégie de groupe

Création d'une GPO utilisateur :

`Suppression_corbeille`

Objectif : supprimer l'icône Corbeille du bureau pour les utilisateurs concernés par l'OU `TS`.

La stratégie n'a pas été appliquée immédiatement.

Après diagnostic, les commandes et actions suivantes ont permis de vérifier son application :

`gpresult /r`

`gpupdate /force`

puis déconnexion / reconnexion de la session.

Résultat : l'icône Corbeille n'était plus présente sur le bureau.

## Incident réseau documenté

Lors de la préparation du réseau avant la jonction au domaine :

- ARP fonctionnait entre les deux machines ;
- le ping échouait ;
- le problème a été isolé au niveau du pare-feu Windows ;
- le pare-feu a été conservé actif ;
- une règle ICMPv4 entrante adaptée a permis de rétablir la communication.

Voir : `diagnostic-icmp.md`

## Démarche

Je documente mes incidents avec la logique suivante :

**Symptôme → Hypothèses → Tests → Résultats → Cause → Correction → Enseignement**
