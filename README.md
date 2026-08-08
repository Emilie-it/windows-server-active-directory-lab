# Windows Server 2025 & Active Directory Lab

Laboratoire personnel consacré à l'apprentissage de l'administration de Windows Server,
d'Active Directory et des fondamentaux réseau.

Ce dépôt documente les configurations réalisées, les tests effectués,
les incidents rencontrés et ma démarche de diagnostic.

## Objectifs

- Installer et configurer Windows Server 2025
- Déployer les rôles AD DS et DNS
- Créer et administrer un domaine Active Directory
- Gérer les utilisateurs et les unités d'organisation
- Intégrer un poste Windows 11 au domaine
- Créer et tester des stratégies de groupe (GPO)
- Configurer la communication réseau entre le serveur et le poste client
- Diagnostiquer et documenter les incidents rencontrés

## Environnement utilisé

- VirtualBox
- Windows Server 2025
- Windows 11 Pro
- Active Directory Domain Services
- DNS
- IPv4

## Architecture du lab

| Machine | Rôle | OS | Adresse IP | DNS |
|---|---|---|---|---|
| SRV-ASSO-DC1 | Contrôleur de domaine / DNS | Windows Server 2025 | 192.168.10.10/24 | 192.168.10.10 |
| PC-TS1 | Poste client | Windows 11 Pro | 192.168.10.20/24 | 192.168.10.10 |

**Domaine :** `asso.lan`

## Lab 01 — Première infrastructure Active Directory

Travaux réalisés :

- installation de Windows Server 2025 ;
- configuration réseau du serveur et du client ;
- installation des rôles AD DS et DNS ;
- promotion du serveur en contrôleur de domaine ;
- création d'unités d'organisation et de comptes utilisateurs ;
- intégration du poste Windows 11 au domaine ;
- création et application de stratégies de groupe (GPO) ;
- tests de communication entre les machines ;
- diagnostic d'un blocage ICMP lié au pare-feu Windows.

## Démarche de diagnostic

Pour les incidents rencontrés, j'applique progressivement la démarche suivante :

**Symptôme → Hypothèses → Tests → Résultats → Correction → Documentation**

Le diagnostic d'un problème de communication ICMP entre les deux machines
sera notamment documenté dans ce dépôt.
