# LAB-PME-PROXMOX

Virtualized enterprise infrastructure simulating a small business, deployed on Proxmox VE (nested virtualization). The lab covers Active Directory, centralized management through GPOs, a mixed Windows/Linux environment with cross-authentication, and network segmentation behind an application firewall.

Project carried out as part of preparing an Ausbildung application for *Fachinformatiker für Systemintegration*.

---

## Table of Contents

- [Objective](#objective)
- [Architecture](#architecture)
- [Technical Environment](#technical-environment)
- [Lab Components](#lab-components)
  - [1. Active Directory Domain Controller](#1-active-directory-domain-controller)
  - [2. Windows 10 Client + GPO](#2-windows-10-client--gpo)
  - [3. Samba File Server Integrated with AD](#3-samba-file-server-integrated-with-ad)
  - [4. Network Segmentation with OPNsense](#4-network-segmentation-with-opnsense)
- [Challenges Encountered and Resolutions](#challenges-encountered-and-resolutions)
- [Limitations and Improvement Areas](#limitations-and-improvement-areas)
- [Resources](#resources)

---

## Objective

This lab was built to address a specific gap in the portfolio: the absence of demonstrated experience with **Windows Server / Active Directory**, and to showcase a skill that is uncommon among junior candidates: **cross-authentication between Windows and Linux** (Kerberos + LDAP + Samba).

Four concrete objectives:

- Deploy an AD domain controller and demonstrate centralized management through GPO
- Integrate a Linux server into the domain with native authentication for AD accounts (no duplicated local accounts)
- Implement real network segmentation with an application firewall (not just routing)
- Document the process, including errors and their resolutions — debugging is part of the demonstrated skill set

---

## Architecture

```
RHEL host (Cockpit / libvirt-KVM)
  └── virbr0 NAT network: 192.168.122.0/24
        └── Proxmox VE VM (nested virtualization, 6 vCPU, LVM-thin storage ~192 GB)
              │
              ├── vmbr0 (WAN) ──── OPNsense ──── vmbr1 (LAN): 10.10.10.0/24
              │                                        │
              │                  ┌─────────────────────┼─────────────────────┐
              │                  │                      │                     │
              │        Windows Server 2022      Windows 10 Client       Ubuntu Server
              │        AD DC / DNS                Client + GPO          Samba / SSSD
              │        10.10.10.10                10.10.10.20           10.10.10.30
```

**Active Directory domain**: `labpme.local` (NetBIOS `LABPME`)

| Machine | Role | IP |
|---|---|---|
| Windows Server 2022 Standard | Domain controller, DNS | 10.10.10.10 |
| Windows 10 | Domain-joined client workstation | 10.10.10.20 |
| Ubuntu Server | Samba file server (cross LDAP) | 10.10.10.30 |
| OPNsense | Firewall / LAN-WAN gateway | 10.10.10.1 (LAN) |

---

## Technical Environment

- **Host hypervisor**: RHEL, Cockpit / libvirt-KVM (nested virtualization)
- **Lab hypervisor**: Proxmox VE 8.x, LVM-thin storage
- **Directory service**: Windows Server 2022 Standard (Desktop Experience), Active Directory Domain Services
- **Client**: Windows 10
- **File server**: Ubuntu Server, Samba 4, SSSD, realmd, Kerberos
- **Network security**: OPNsense (FreeBSD-based)
- **Drivers**: VirtIO (storage and network) for all VMs

---

## Lab Components

### 1. Active Directory Domain Controller

Promotion of a Windows Server 2022 machine to domain controller (`Install-ADDSForest`), carried out entirely in PowerShell after an initial installation was mistakenly done in Server Core mode.

- Domain `labpme.local`, integrated DNS, NTDS / KDC / Netlogon services operational
- Organizational unit structure: `OU_Direction`, `OU_IT`, `OU_Comptabilite`, `OU_Ordinateurs`
- Test user account (`test.user`) created in `OU_IT`

### 2. Windows 10 Client + GPO

Domain-joined client workstation, used to demonstrate centralized management through Group Policies applied to a standard (non-admin) user account.

| GPO | Effect |
|---|---|
| `GPO_FondEcran1` | Enforced wallpaper, not editable by the user |
| `GPO_CMDrestriction` | Command Prompt (`cmd.exe`) disabled |
| `GPO_ConPannel_restriction` | Access to Control Panel blocked |
| `GPO_inactivity` | Automatic session lock after inactivity |
| `GPO_MAPPAGE` | Network drive `Z:` automatically mapped to the Samba share |

Each GPO was validated through real testing (`gpupdate /force`, sign-out/sign-in, `gpresult /r`), not only configured.

### 3. Samba File Server Integrated with AD

Ubuntu server joined to the domain via `realmd`/`adcli`, with Samba configured in `security = ads` to authenticate SMB connections directly against the directory service — no separate local Samba accounts.

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

Authentication chain: **realmd + adcli** (domain discovery and join) → **SSSD** (AD identity resolution on Linux) → **Winbind** (password validation on the Samba side). The `test.user` account authenticates with the same credentials as the Windows session.

The share is automatically mounted as drive `Z:` on the client workstation via GPO — unified access, without double authentication.

### 4. Network Segmentation with OPNsense

The three VMs were migrated from a flat network to an isolated LAN (`10.10.10.0/24`) behind OPNsense's LAN interface, while its WAN interface provides Internet access through the host's NAT.

A firewall rule demonstrates targeted filtering: blocking outbound port 443 (HTTPS) for the Samba VM only, without affecting the rest of the network.

```
Firewall > Rules > LAN
  Action: Block | Source: 10.10.10.30/32 | Destination: any | Port: HTTPS (443)
```

**Verified result**:
- From the Windows client: `ping 8.8.8.8` → OK
- From Samba: `curl https://google.com` → blocked (timeout), `curl http://google.com` → still active

---

## Challenges Encountered and Resolutions

Debugging represented a significant part of the work. A few meaningful examples, documented in detail in the [screenshots](./screenshots):

- **Missing VirtIO drivers** during Windows Server/10 installation → disk and network not detected, resolved by manually injecting drivers (`vioscsi`, `NetKVM`) from the `virtio-win` ISO
- **AD DS promotion failed** (`TCP/IP networking protocol must be properly configured`) → caused by DHCP IP instead of a static self-referenced DNS IP
- **Winbind / SSSD conflict** after joining the domain twice (`realm join` then `net ads join`) → rebuilt cleanly with a single consistent join mechanism
- **Emulated `e1000` network card not working** under nested virtualization after migration to the new LAN → replaced with the VirtIO model already used on the other interfaces
- **Home router MAC filtering** blocking direct network bridging → worked around by keeping the Proxmox VM behind the host's `virbr0` NAT instead of physical bridge mode

---

## Limitations and Improvement Areas

Intentionally accepted to keep the lab within a reasonable timeframe:

- **No automated backup (PBS)**: insufficient disk space on the LVM-thin pool during initial sizing
- **Samba share DNS resolution not working** (`\\samba.labpme.local`): IP-based access kept; likely root cause (Kerberos SPN mismatch) identified but not fully investigated
- **PowerShell not restricted** for the standard user: intentionally left open to keep diagnostic flexibility during lab development
- **Monitoring VM (Grafana/Prometheus)**: not implemented in this iteration, deprioritized in favor of the four core building blocks

---

## Resources

- [`/screenshots`](./screenshots) — screenshots of each step (installation, configuration, validation testing)
- [PowerPoint Presentation](./LABPME-presentation.pptx) — visual summary of the project

---

**Author**: [Naaka01](https://github.com/Naaka01)
