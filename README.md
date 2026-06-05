# Archangel Home Network Security Lab

**Owner:** Z.Angell
**GitHub:** [@ZAngell-Cloud](https://github.com/ZAngell-Cloud)
**Last Updated:** June 05 2026
**Status:** Phase 7 of 15 Complete — Actively Expanding

> A production-grade, Zero Trust home network security lab built on enterprise open-source tooling. Designed to demonstrate hands-on proficiency in network security architecture, firewall administration, SIEM operations, incident response, and threat detection — closely aligned with enterprise and DoD security environments.

---

## Table of Contents

- [Overview](#overview)
- [Zero Trust Architecture](#zero-trust-architecture)
- [Hardware](#hardware)
- [Build Progress](#build-progress)
- [Enterprise Technology Mapping](#enterprise-technology-mapping)
- [Skills Demonstrated](#skills-demonstrated)
- [Repository Structure](#repository-structure)
- [Security Notice](#security-notice)

---

## Overview

This lab implements a **multi-zone, physically and logically segmented network** using exclusively open-source software deployed on enterprise-grade refurbished hardware. Every design decision attempts to mirror practices used in production enterprise and DoD environments, including:

- **Zero Trust network architecture** — no implicit trust between zones, explicit allow-only firewall policy
- **Physical network isolation** for the highest-risk zones (DMZ and IoT) using dedicated NIC interfaces
- **7-VLAN micro-segmentation** enforcing least-privilege inter-zone communication
- **Defense-in-depth** — firewall → IDS/IPS → SIEM → EDR/DFIR layered across all traffic paths
- **Full logging pipeline** feeding centralized SIEM with multi-source correlation
- **Adversary emulation** using Atomic Red Team and Caldera mapped to MITRE ATT&CK

The lab serves as a live skills demonstration environment and a managed security services demo platform.

---

## Zero Trust Architecture

This lab is designed around the **NIST SP 800-207 Zero Trust Architecture** principles:

| ZTA Principle | Implementation |
|---|---|
| **Never trust, always verify** | Default deny; all inter-VLAN traffic requires explicit firewall allow rules |
| **Least privilege access** | Each VLAN has minimum necessary access only; admin hosts restricted by IP alias |
| **Assume breach** | Dedicated red team node (Kali) in isolated VLAN allowed to attack test zones; blue team detection validated against real attacks |
| **Micro-segmentation** | 7 VLANs enforce zone boundaries; DMZ and IoT physically isolated via dedicated NICs |
| **Continuous verification** | Suricata IDS/IPS inline on all traffic; Wazuh agents on every endpoint |
| **Policy Enforcement Point (PEP)** | OPNsense firewall enforces all access decisions at zone boundaries |
| **Policy Decision Point (PDP)** | Firewall rules + Wazuh SIEM correlation drive access decisions |

### Network Security Zones

```
Internet
    │
    │ WAN (physical NIC — ISP modem bridge)
    │
┌───▼──────────────────────────────────────────────────┐
│              OPNsense Firewall (fw01)                │
│         5-NIC — Zero Trust Policy Enforcer           │
│	 (creates VLANs 10/20/30/40/50/60/99)	       │
│  Suricata IPS │ WireGuard VPN │ Unbound DNS+DoT      │
└──┬────────┬──────────┬──────────────┬────────────────┘
   │        │          │              │
   │    [Physical   [Physical  [LAN Trunk — Tagged VLANs]
   │    DMZ NIC]   IoT NIC]           │
   │        │          │         ┌────▼──────────────┐
   │     VLAN 40    VLAN 50      │ TP-Link TL-SG2008P│
   │     (DMZ)      (IoT)        │   Managed Switch  │
   │                             └──────┬────────────┘
   │                                    │
   │              ┌─────────────────────┤
   │		  │			│
   │	    [Port-A trunk]	  [Port-B trunk]
   │              │                     │
   │         ┌────▼──────┐    ┌─────────▼────────────┐
   │         │  pve01    │    │  UAP-AC-LITE         │
   │         │ Proxmox   │    │  WPA3 SSIDs          │
   │         │  G4 Mini  │    │  VLAN-mapped         │
   │         └────┬──────┘    └─────────┬────────────┘
   │              │			│
   │		  │			└────────────┐
   │    ┌─────────┴──────────┐		   ┌─────────┴──────────┐
   │    │                    │		   │                    │
   │ VLANs: 10(ADMIN) 20(MGMT) 30(STOR)     VLANs: 10(ADMIN) 50(IoT)
   │ 40(DMZ) 60(FAMILY) 99(GUEST)		     60(FAMILY) 99(GUEST)
   │
   │
   └── OOB Management (NIC5 — direct workstation cable)
```

---

## Hardware

> **Security Notice:** Specific IP addresses, hostnames, and configuration details are not published in this repository. All network references use zone names and roles. See [ARCHITECTURE.md](ARCHITECTURE.md) for full zone design.

### Phase 1 — Active Inventory

| Device | Specs | Role | OS | Zone |
|---|---|---|---|---|
| **Dell OptiPlex 7040 SFF** | Intel i7-6700 · 32GB RAM · 5 NICs (4-port aftermarket PCIe + onboard) | Firewall · IDS/IPS · VPN Gateway | OPNsense 24.x (bare metal, ZFS) | Edge; between WAN and all internal zones |
| **HP EliteDesk 800 G4 Mini** | Intel i7-8700T · 64GB RAM · 2× NVMe + SATA | Hypervisor (single-node) | Proxmox VE 8.x (bare metal) | MGMT VLAN (VLAN 20) |
| **HP EliteDesk 800 G5 Mini** | Intel i5-9500T · 16GB RAM · 2× NVMe + SATA | Network-Attached Storage | TrueNAS SCALE 24.x (bare metal) | STOR VLAN (VLAN 30) |
| **Dell Inspiron 3501** | Intel i5-1135G7 · 16GB RAM · 256GB NVMe | Red Team / Adversary Emulation | Kali Linux 2024.x (FDE) | GUEST VLAN 99 (isolated) |
| **HP Omen 16** | (Primary workstation) | Admin Workstation | Existing OS | ADMIN VLAN 10 via VPN |
| **TP-Link TL-SG2008P** | 8-port PoE managed switch | Layer 2; VLAN-aware switching | TL-SG2008P firmware | Core switching |
| **Ubiquiti UAP-AC-LITE** | 802.11ac dual-band AP | Wireless access · VLAN-tagged SSIDs | UniFi OS | Managed via UniFi controller (pve01 LXC) |

### Phase 2 — Planned Additions

| Device | Role | Status |
|---|---|---|
| **HP ProDesk 400 G4 Mini** (i5-8500T · 16GB) | Proxmox Backup Server (PBS) — dedicated backup tier | Shelved for Phase 2 |

**Why Phase 1 runs a single-node:** Economics (all available NVMe drives currently allocated), reduced Phase 1 complexity, and room to scale/improve as money allows: single-node → dedicated backup server → full HA cluster.

---

## Build Progress

| Phase | Description | Status | Notes |
|---|---|---|---|
| **Phase 1** | OPNsense Firewall Installation | Complete | ZFS, 5-NIC, hardened |
| **Phase 2** | VLAN Configuration | Complete | 7 VLANs, physical isolation on DMZ & IoT |
| **Phase 3** | Managed Switch Configuration | Complete | 802.1Q trunking, PoE for AP |
| **Phase 4** | Proxmox Hypervisor | Complete | Single-node, VLAN-aware bridge, ZFS |
| **Phase 5** | TrueNAS SCALE NAS | Complete | ZFS mirror, encrypted, NFS/SMB |
| **Phase 6** | UniFi Wireless | Complete | WPA3, 4 SSIDs VLAN-mapped |
| **Phase 7** | Firewall Rules & Inter-VLAN Policy | Complete | Default deny, least privilege, DNS redirect |
| **Phase 8** | VPN (WireGuard + Headscale) | In Progress | Self-hosted Tailscale coordinator |
| **Phase 9** | DNS Security (Pi-hole + Unbound + DoT) | Planned | Recursive resolver, DNSSEC, DoT upstream |
| **Phase 10** | IDS/IPS (Suricata) | Planned | Inline IPS, ET Open + URLhaus rulesets |
| **Phase 11** | SIEM (Wazuh) | Planned | All-in-one, agents on all endpoints |
| **Phase 12** | DFIR (Velociraptor) | Planned | Hunt server + endpoint agents |
| **Phase 13** | DMZ Services + Reverse Proxy | Planned | Nginx + T-Pot honeypot |
| **Phase 14** | Red Team (Kali + Atomic Red Team) | Planned | MITRE ATT&CK emulation |
| **Phase 15** | Logging, Backups & Monitoring | Planned | Centralized pipeline, 3-2-1 backup strategy |

---

## Enterprise Technology Mapping

This lab uses open source equivalents to commercial enterprise and DoD deployed tools. My goal is to exhibit skills that are fully transferable.

| Home Lab (Open Source) | Commercial / Enterprise Equivalent | Domain |
|---|---|---|
| **OPNsense** | Palo Alto NGFW · Fortinet FortiGate · Check Point | Firewall / NGFW |
| **Suricata** | Palo Alto Threat Prevention · Snort Enterprise · Darktrace | IDS/IPS |
| **Wazuh** | Splunk SIEM · IBM QRadar · Microsoft Sentinel · Elastic SIEM | SIEM / XDR |
| **Velociraptor** | CrowdStrike Falcon · Carbon Black · SentinelOne | EDR / DFIR |
| **Pi-hole + Unbound** | Cisco Umbrella · Infoblox · BlueCat DNS | DNS Security |
| **WireGuard + Headscale** | Cisco AnyConnect · Palo Alto GlobalProtect · ZScaler ZPA | VPN / ZTNA |
| **Proxmox VE** | VMware vSphere · Microsoft Hyper-V · Nutanix | Virtualization |
| **TrueNAS SCALE** | NetApp ONTAP · Dell EMC Isilon | Enterprise NAS |
| **Nginx Proxy Manager** | F5 BIG-IP · Citrix ADC · AWS ALB | Reverse Proxy / Load Balancer |
| **T-Pot Honeypot** | Attivo Networks · Illusive Networks · TrapX | Deception Technology |
| **Atomic Red Team + Caldera** | SafeBreach · AttackIQ · Cymulate | Adversary Emulation |
| **Gitea** | GitHub Enterprise · GitLab Self-Hosted · Bitbucket Server | Source Control |
| **Vaultwarden** | CyberArk · HashiCorp Vault · Delinea Secret Server | Secrets Management |

---

## Skills Demonstrated

| Skill Domain | Evidence in This Lab |
|---|---|
| **Zero Trust Architecture** | NIST SP 800-207 implementation, PEP/PDP model, micro segmentation |
| **Network Security Engineering** | 7 VLAN design, 5 NIC physical isolation, 802.1Q trunking |
| **Firewall Administration** | OPNsense complex rule sets, aliased objects, inter-VLAN policy, NAT |
| **IDS/IPS Deployment** | Suricata inline IPS, custom rulesets, false positive reduction |
| **SIEM Operations** | Wazuh multi-source ingestion, custom detection rules, MITRE ATT&CK mapping |
| **EDR / DFIR** | Velociraptor hunt queries, artifact collection, IR playbook execution |
| **Threat Intelligence** | URLhaus, Emerging Threats Open, T-Pot threat data correlation |
| **VPN Architecture** | WireGuard site VPN + Headscale self-hosted mesh coordinator |
| **DNS Security** | Pi-hole sinkholing, Unbound recursive resolver, DNS-over-TLS, DNSSEC |
| **Virtualization** | Proxmox single-node (eventual cluster expansion), LXC + VM, VLAN aware bridge |
| **Storage Engineering** | ZFS mirror, dataset encryption, snapshot policy, 3-2-1 backup strategy |
| **Adversary Emulation** | Atomic Red Team MITRE ATT&CK techniques, Caldera server, detection validation |
| **Incident Response** | Written IR playbooks: ransomware, phishing, data exfil, insider threat |
| **Logging Pipeline** | Multi-source syslog aggregation, SIEM correlation, archive to NAS |
| **Secrets Management** | Vaultwarden (Bitwarden-compatible), SSH key only auth, TOTP 2FA |
| **Documentation Practice** | Full build documentation, changelog, sanitized config references |

---

## Repository Structure

```
archangel-homelab/
├── README.md                        ← This file; overview and skills mapping
├── ARCHITECTURE.md                  ← Network topology, VLAN design, Zero Trust model
├── hardware/
│   ├── inventory.md                 ← Detailed hardware specs and role assignments
│   └── storage-allocation.md        ← Drive inventory, ZFS pool design, allocation rationale
├── setup/
│   ├── 01-firewall.md               ← Phase 1-2: OPNsense installation and VLAN configuration
│   ├── 02-switch.md                 ← Phase 3: TP-Link managed switch setup
│   ├── 03-hypervisor.md             ← Phase 4: Proxmox VE single-node deployment
│   ├── 04-nas.md                    ← Phase 5: TrueNAS SCALE configuration
│   ├── 05-wireless.md               ← Phase 6: UniFi AP adoption and SSID configuration
│   ├── 06-firewall-rules.md         ← Phase 7: Inter-VLAN policy and firewall rules
│   ├── 07-vpn.md                    ← Phase 8: WireGuard + Headscale
│   ├── 08-dns.md                    ← Phase 9: Pi-hole + Unbound + DoT
│   ├── 09-ids-ips.md                ← Phase 10: Suricata IDS/IPS
│   ├── 10-siem.md                   ← Phase 11: Wazuh SIEM deployment
│   ├── 11-dfir.md                   ← Phase 12: Velociraptor DFIR
│   ├── 12-dmz.md                    ← Phase 13: DMZ services and reverse proxy
│   ├── 13-red-team.md               ← Phase 14: Kali + adversary emulation
│   └── 14-logging-backups.md        ← Phase 15: Centralized logging and backup strategy
├── playbooks/
│   ├── ir-ransomware.md             ← Incident response: ransomware
│   ├── ir-phishing.md               ← Incident response: phishing
│   ├── ir-data-exfil.md             ← Incident response: data exfiltration
│   └── ir-insider-threat.md         ← Incident response: insider threat
└── progress/
    └── changelog.md                 ← Dated build log
```

---

## Security Notice

> **This is a public repository.** The following information is intentionally omitted:
> - Actual IP addressing scheme (RFC 1918 private — not published)
> - Firewall configuration exports
> - Credential material of any kind
> - VPN keys or certificates
> - Internal DNS zone data
>
> Configuration examples throughout this documentation use generic placeholders clearly marked as `[EXAMPLE]` or zone-name references (e.g., `<WAZUH-IP>`, `<ADMIN-GW>`). Adapt all addressing to your own environment.

---

*Maintained by Z.Angell*

*Certifications: CompTIA Security+ SY0-701 · CompTIA Hybrid Server Pro: Core · Google IT Automation with Python Professional Certificate*
