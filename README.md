# LAB-PME-PROXMOX

Infrastructure d'entreprise virtualisée simulant une petite PME, déployée sur Proxmox VE (nested virtualization). Le lab couvre l'annuaire Active Directory, la gestion centralisée par GPO, un environnement mixte Windows/Linux avec authentification croisée, et la segmentation réseau derrière un firewall applicatif.

Projet réalisé dans le cadre de la préparation d'une candidature Ausbildung *Fachinformatiker für Systemintegration*.

---

## Sommaire

- [Objectif](#objectif)
- [Architecture](#architecture)
- [Environnement technique](#environnement-technique)
- [Composants du lab](#composants-du-lab)
  - [1. Contrôleur de domaine Active Directory](#1-contrôleur-de-domaine-active-directory)
  - [2. Poste client Windows 10 + GPO](#2-poste-client-windows-10--gpo)
  - [3. Serveur de fichiers Samba intégré à AD](#3-serveur-de-fichiers-samba-intégré-à-ad)
  - [4. Segmentation réseau avec OPNsense](#4-segmentation-réseau-avec-opnsense)
- [Difficultés rencontrées et résolutions](#difficultés-rencontrées-et-résolutions)
- [Limites et axes d'amélioration](#limites-et-axes-damélioration)
- [Ressources](#ressources)

---

## Objectif

Ce lab a été construit pour combler un point spécifique du portfolio : l'absence d'expérience démontrée sur **Windows Server / Active Directory**, et pour illustrer une compétence peu fréquente chez les candidats juniors : l'**authentification croisée Windows/Linux** (Kerberos + LDAP + Samba).

Quatre objectifs concrets :

- Déployer un contrôleur de domaine AD et démontrer la gestion centralisée via GPO
- Intégrer un serveur Linux au domaine avec authentification native des comptes AD (pas de comptes locaux dupliqués)
- Mettre en place une segmentation réseau réelle avec un firewall applicatif (pas seulement du routage)
- Documenter le processus, y compris les erreurs et leur résolution — le débogage fait partie de la compétence démontrée

---

## Architecture

```
Hôte RHEL (Cockpit / libvirt-KVM)
  └── réseau NAT virbr0 : 192.168.122.0/24
        └── VM Proxmox VE (nested virtualization, 6 vCPU, stockage LVM-thin ~192 Go)
              │
              ├── vmbr0 (WAN) ──── OPNsense ──── vmbr1 (LAN) : 10.10.10.0/24
              │                                        │
              │                  ┌─────────────────────┼─────────────────────┐
              │                  │                      │                     │
              │        Windows Server 2022      Windows 10 Client       Ubuntu Server
              │        AD DC / DNS                Client + GPO          Samba / SSSD
              │        10.10.10.10                10.10.10.20           10.10.10.30
```

**Domaine Active Directory** : `labpme.local` (NetBIOS `LABPME`)

| Machine | Rôle | IP |
|---|---|---|
| Windows Server 2022 Standard | Contrôleur de domaine, DNS | 10.10.10.10 |
| Windows 10 | Poste client joint au domaine | 10.10.10.20 |
| Ubuntu Server | Serveur de fichiers Samba (LDAP croisé) | 10.10.10.30 |
| OPNsense | Firewall / passerelle LAN-WAN | 10.10.10.1 (LAN) |

---

## Environnement technique

- **Hyperviseur hôte** : RHEL, Cockpit / libvirt-KVM (virtualisation imbriquée)
- **Hyperviseur du lab** : Proxmox VE 8.x, stockage LVM-thin
- **Annuaire** : Windows Server 2022 Standard (Desktop Experience), Active Directory Domain Services
- **Client** : Windows 10
- **Serveur de fichiers** : Ubuntu Server, Samba 4, SSSD, realmd, Kerberos
- **Sécurité réseau** : OPNsense (dérivé FreeBSD)
- **Pilotes** : VirtIO (stockage et réseau) pour l'ensemble des VMs

---

## Composants du lab

### 1. Contrôleur de domaine Active Directory

Promotion d'un Windows Server 2022 en contrôleur de domaine (`Install-ADDSForest`), entièrement réalisée en PowerShell suite à une installation initiale par erreur en mode Server Core.

- Domaine `labpme.local`, DNS intégré, services NTDS / KDC / Netlogon opérationnels
- Structure d'unités organisationnelles : `OU_Direction`, `OU_IT`, `OU_Comptabilite`, `OU_Ordinateurs`
- Compte utilisateur de test (`test.user`) créé dans `OU_IT`

### 2. Poste client Windows 10 + GPO

Poste client joint au domaine, utilisé pour démontrer la gestion centralisée par stratégies de groupe appliquées à un compte utilisateur standard (non-administrateur).

| GPO | Effet |
|---|---|
| `GPO_FondEcran1` | Fond d'écran imposé, non modifiable par l'utilisateur |
| `GPO_CMDrestriction` | Invite de commandes (`cmd.exe`) désactivée |
| `GPO_ConPannel_restriction` | Accès au Panneau de configuration bloqué |
| `GPO_inactivity` | Verrouillage automatique de session après inactivité |
| `GPO_MAPPAGE` | Lecteur réseau `Z:` monté automatiquement vers le partage Samba |

Chaque GPO a été validée par test réel (`gpupdate /force`, déconnexion/reconnexion, `gpresult /r`), pas seulement configurée.

### 3. Serveur de fichiers Samba intégré à AD

Serveur Ubuntu joint au domaine via `realmd`/`adcli`, avec Samba configuré en `security = ads` pour authentifier les connexions SMB directement contre l'annuaire — aucun compte Samba local séparé.

```ini
[global]
   workgroup = LABPME
   realm = LABPME.LOCAL
   security = ads

[Shared-IT]
   path = /srv/shared-IT
   valid users = @"LABPME\Domain Users"
   read only = no
   browsable = yes
   create mask = 0660
   directory mask = 2770
```

Chaîne d'authentification : **realmd + adcli** (découverte et jonction du domaine) → **SSSD** (résolution des identités AD côté Linux) → **Winbind** (validation des mots de passe côté Samba). Le compte `test.user` s'authentifie avec les mêmes identifiants que sa session Windows.

Le partage est monté automatiquement en lecteur `Z:` sur le poste client via GPO — accès unifié, sans double authentification.

### 4. Segmentation réseau avec OPNsense

Les trois VMs ont été migrées d'un réseau plat vers un LAN isolé (`10.10.10.0/24`) derrière l'interface LAN d'OPNsense, dont l'interface WAN assure la sortie Internet via le NAT de l'hôte.

Une règle de pare-feu démontre un filtrage ciblé : blocage du port 443 (HTTPS) en sortie pour la VM Samba uniquement, sans affecter le reste du réseau.

```
Firewall > Rules > LAN
  Action: Block | Source: 10.10.10.30/32 | Destination: any | Port: HTTPS (443)
```

**Résultat vérifié** :
- Depuis le client Windows : `ping 8.8.8.8` → OK
- Depuis Samba : `curl https://google.com` → bloqué (timeout), `curl http://google.com` → toujours actif

---

## Difficultés rencontrées et résolutions

Le débogage a représenté une part importante du travail. Quelques exemples significatifs, documentés en détail dans les [captures d'écran](./screenshots) :

- **Drivers VirtIO manquants** à l'installation de Windows Server/10 → disque et réseau non détectés, résolu par injection manuelle des pilotes (`vioscsi`, `NetKVM`) depuis l'ISO `virtio-win`
- **Promotion AD DS échouée** (`TCP/IP networking protocol must be properly configured`) → causée par une IP en DHCP au lieu d'une IP statique auto-référencée en DNS
- **Conflit Winbind / SSSD** après une double jonction du domaine (`realm join` puis `net ads join`) → repris proprement avec un seul mécanisme de jonction cohérent
- **Carte réseau émulée en `e1000` non fonctionnelle** sous virtualisation imbriquée après migration vers le nouveau LAN → remplacée par le modèle VirtIO déjà piloté sur les autres interfaces
- **Filtrage MAC du routeur domestique** empêchant le bridge réseau direct → contourné en gardant la VM Proxmox derrière le NAT `virbr0` de l'hôte plutôt qu'en bridge physique

---

## Limites et axes d'amélioration

Assumés sciemment pour tenir le lab dans un temps raisonnable :

- **Pas de sauvegarde automatisée (PBS)** : espace disque insuffisant sur le pool LVM-thin au moment du dimensionnement initial
- **Résolution DNS du partage Samba non fonctionnelle** (`\\samba.labpme.local`) : accès via IP retenu, cause probable (désaccord de SPN Kerberos) identifiée mais non creusée
- **PowerShell non restreint** pour l'utilisateur standard : laissé ouvert volontairement pour conserver la flexibilité de diagnostic pendant le développement du lab
- **VM de supervision (Grafana/Prometheus)** : non implémentée dans cette itération, dépriorisée au profit des quatre briques cœur

---

## Ressources

- [`/screenshots`](./screenshots) — captures d'écran de chaque étape (installation, configuration, tests de validation)
- [Présentation PowerPoint](./LABPME-presentation.pptx) — synthèse visuelle du projet

---

**Auteur** : [Naaka01](https://github.com/Naaka01)
