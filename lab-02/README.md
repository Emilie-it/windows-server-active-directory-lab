# lab-02/README.md

## Lab 02 — Nouveau contrôleur de domaine et diagnostics réseau/DNS

### Objectif
Reconstruire un environnement Active Directory sur une nouvelle VM, indépendante du Lab 01, et approfondir le diagnostic DNS et la résolution de problèmes de jonction au domaine.

### Architecture

| Machine   | Rôle                        | Système           | Adresse IPv4    | DNS        |
|-----------|-----------------------------|--------------------|------------------|------------|
| SRV-DC01  | Contrôleur de domaine / DNS | Windows Server 2025| 10.0.2.10/24     | 10.0.2.10  |
| PC1       | Poste client (Direction)    | Windows 11 25H2    | 10.0.2.20/24     | 10.0.2.10  |

**Hyperviseur :** VirtualBox
**Réseau :** interne, `intnet`
**Domaine Active Directory :** `asso.lab`
**Nom NetBIOS :** `ASSO`

### Mise en place du contrôleur de domaine

Installation des rôles AD DS, DNS et Services de fichiers et de stockage sur `SRV-DC01`, puis promotion en contrôleur de domaine (nouvelle forêt `asso.lab`).

Vérification post-promotion :

- `Get-ADDomainController` : `SRV-DC01`, catalogue global `True`
- `Get-ADDomain` : `DNSRoot = asso.lab`, `NetbiosName = ASSO`
- `dcdiag` : échec initial du test `Connectivity`
- `dcdiag /test:dns` : réussi

Voir : `diagnostic-dc-promotion.md`

### Jonction du poste client (PC1)

Avant la jonction, blocage de l'installation de Windows 11 par mémoire insuffisante (2048 Mo, minimum 4 Go requis) — corrigé par une allocation de 4646 Mo.

Écran noir constaté après ce correctif, résolu par une augmentation du nombre de vCPU — cause exacte non confirmée, à investiguer.

Jonction ensuite bloquée malgré une configuration IP/DNS correcte : IPv6 activé et prioritaire sur IPv4, alors que l'infrastructure AD répond uniquement en IPv4. Après correction de la priorité de protocole, jonction réussie et vérifiée (`whoami`, ping bidirectionnel).

### Incidents documentés

- Résolution DNS erronée (loopback au lieu de l'IP statique) et échecs `dcdiag` initiaux : voir `diagnostic-dc-promotion.md`
