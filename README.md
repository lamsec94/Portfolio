# Homelab Infrastructure Portfolio

A two-node Proxmox-based homelab engineered to mirror enterprise infrastructure patterns across
virtualization, identity, networking, security, and automation. This repository serves as the
primary landing page for all lab documentation, architecture decisions, and operational runbooks.

> **Built by:** Lamar Scott | **GitHub:** [lamsec94](https://github.com/lamsec94) | **Last updated:** May 2026

---

## Architecture

![Homelab Topology](../homelab-network-documentation/images/HOMELAB-NETWORK-ARCHITECTURE.png)



---

## What I Built

Migrated a flat home network into a fully segmented, enterprise-style infrastructure across
5 VLANs with OPNsense handling all routing and firewall policy. Deployed a dual-node Proxmox
cluster running 15 VMs and LXC containers across Windows and Linux workloads. Implemented
dual identity management with Active Directory (Windows Server 2022/2025) and FreeIPA
(AlmaLinux), enterprise PKI with a wildcard certificate across all 14 internal HTTPS services,
a full Splunk SIEM pipeline ingesting from 10+ sources, and an Ansible automation layer
managing 12 hosts across 6 inventory groups. All services are reverse-proxied through Nginx
Proxy Manager with TLS termination via an internal CA.

---

## Hardware

| Node | Device          | RAM   | Role                          |
|------|-----------------|-------|-------------------------------|
| su1  | Lenovo M910t    | 48 GB | Primary Proxmox node          |
| su2  | HP EliteDesk    | 16 GB | Secondary Proxmox node        |
| Pi5  | Raspberry Pi 5  | 8 GB  | AdGuard Home, Tailscale relay |
| —    | Netgear GS308EP | —     | Layer-2 VLAN switching        |
| —    | TP-Link ER7206  | —     | Upstream edge router          |
| —    | TP-Link AX1800  | —     | AP mode — GUEST + IOT SSIDs   |

---

## VLAN Design

| VLAN | Name    | Purpose                           |
|------|---------|-----------------------------------|
| 1    | MGMT    | Hypervisor and network management |
| 10   | LAB     | Servers, VMs, admin workstations  |
| 20   | GUEST   | Guest wireless isolation          |
| 30   | IOT     | IoT device isolation              |
| 100  | TRANSIT | Upstream link to edge router      |

---

## Service Inventory

| Service             | Type   | Internal URL                | Notes                            |
|---------------------|--------|-----------------------------|----------------------------------|
| OPNsense            | VM     | —                           | Firewall, Suricata IDS, DHCP     |
| LAB-DC (WS2022)     | VM     | —                           | Primary DC, DNS, ADCS, PKI       |
| LAB-DC2 (WS2025)    | VM     | —                           | Secondary DC, Splunk UF          |
| FreeIPA             | VM     | ipa01.ipa.homelab.local     | Linux identity, HBAC, sudo       |
| Splunk Enterprise   | LXC    | splunk.homelab.local        | SIEM, 10+ log sources            |
| Ansible Controller  | VM     | —                           | Ubuntu 24.04, 12 managed hosts   |
| Forgejo             | LXC    | forgejo.homelab.local       | Internal Git service             |
| Nextcloud           | LXC    | nextcloud.homelab.local     | File storage, laptop backup      |
| GLPI                | Docker | glpi.homelab.local          | ITSM, LDAP auth, asset discovery |
| Nginx Proxy Manager | Docker | npm.homelab.local           | Reverse proxy, wildcard HTTPS    |
| Immich              | LXC    | immich.homelab.local        | Photo management, Docker         |
| AdGuard Home        | Pi5    | adguard.homelab.local       | DNS, conditional forwarding      |
| Kali Purple         | VM     | —                           | Security lab, Ansible-managed    |
| Win11 Pro           | VM     | —                           | Domain-joined admin workstation  |

---

## Identity & Access

**Windows — Active Directory**
- Domain: `homelab.local` | Primary DC: LAB-DC (WS2022) | Secondary: LAB-DC2 (WS2025)
- OU hierarchy: `Corp-Computers`, `Corp-Users`, `Engineering`, `IT Staff`
- GPOs: screen lock, 7-Zip deployment, WSUS targeting, USB storage blocking
- Enterprise Root CA (`homelab-CA`) via ADCS — wildcard cert covering all 14 internal services
- LDAP service account pattern (`svc-glpi`) for least-privilege third-party integration

**Linux — FreeIPA**
- Host: `ipa01` (AlmaLinux) | Enrolled: `ipa01`, `ubuntuserver`
- HBAC: `allowadmins` active, `allowall` disabled
- Sudo rule requiring password prompt for all admin operations

---

## Security & Monitoring

**Splunk Enterprise SIEM**
- 10+ log sources: OPNsense syslog, Suricata IDS alerts, Windows Event Logs, Linux auth logs
- Universal Forwarders deployed via Ansible to all Linux VMs; Windows via MSI
- Dashboards: Authentication, DNS Intelligence, FreeIPA, Lab Health, Linux Security, Network Traffic, Windows Security

**Suricata IDS** — inline on OPNsense with alert forwarding to Splunk

**Hardening Baseline** (Ansible-enforced on all Linux hosts)
- SSH key-only auth · `ufw` default-deny · `fail2ban` · scheduled patching

**PKI**
- All 14 services served over HTTPS with wildcard cert signed by internal `homelab-CA`
- CA cert distributed to system trust store and Firefox on Fedora admin workstation

---

## Automation

Ansible Controller on Ubuntu 24.04 managing 12 hosts across 6 inventory groups.

| Inventory Group  | Members                               |
|------------------|---------------------------------------|
| proxmox_cluster  | proxmox1, proxmox2                    |
| linux_vms        | ubuntu-server, almalinux, kali-purple |
| windows_vms      | windows-dc, windows11, LAB-DC2        |
| lxc_containers   | forgejo, splunk, nextcloud            |
| docker_hosts     | docker-host                           |
| splunk_servers   | splunk                                |

**Key playbooks:**
- `update-all.yml` — fleet patching across apt/dnf/Docker; runtime reduced from 23 min → 2 min after SSH pipelining and module optimization; 12/12 host success rate
- `deploy-glpi-agent.yml` — cross-platform GLPI asset discovery agent deployment across Linux fleet
- `splunkforwarder` role — UF deployment and log source configuration across all Linux targets

---

## Backup & Recovery

| Scope            | Tool                 | Schedule      | Notes                            |
|------------------|----------------------|---------------|----------------------------------|
| VM snapshots     | Proxmox vzdump + PBS | Weekly Sunday | Both nodes                       |
| Laptop home dir  | Déjà Dup → Nextcloud | Scheduled     | Fedora workstation               |
| System snapshots | Timeshift (rsync)    | Pre-change    | Taken before all major changes   |
| Key VM snapshots | Proxmox manual       | Event-driven  | post-dc-promotion, glpi-baseline |

---

## Documentation

| Repository | Contents |
|---|---|
| [active-directory-lab](../active-directory-lab) | AD domain design, OU structure, GPOs, PKI, LDAP integration |
| [homelab-network-documentation](../homelab-network-documentation) | VLAN layout, OPNsense config, DNS architecture |
| [splunk-siem](../splunk-siem) | SIEM deployment, log sources, dashboards, UF automation |
| [homelab-runbooks](../homelab-runbooks) | Operational procedures, SOPs, change log, incident templates |
| [glpi-itsm-deployment](../Glpi-itsm-deployment) | ITSM platform deployment, LDAP auth, intake forms, asset discovery |

---

## Skills Demonstrated

`Proxmox VE` `Windows Server 2022/2025` `Active Directory` `Group Policy` `ADCS / PKI`
`FreeIPA` `OPNsense` `VLAN segmentation` `Suricata IDS` `Splunk Enterprise`
`Ansible` `Docker` `LXC` `Linux administration` `DNS` `Nginx Proxy Manager`
`GLPI ITSM` `Tailscale` `Backup & Recovery` `Infrastructure documentation`

---

**Email:** [scottlamar05@gmail.com](mailto:scottlamar05@gmail.com)
**LinkedIn:** [linkedin.com/in/lamarscott](https://www.linkedin.com/in/lamarscott/)
**GitHub:** [github.com/lamsec94](https://github.com/lamsec94)
