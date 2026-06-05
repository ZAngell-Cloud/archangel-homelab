# Network Architecture

**Version:** 1.0
**Last Updated:** June 05 2026
**Status:** Phase 7 Complete — Core segmentation and policy enforcement operational

> Specific IP addresses are redacted in this public document. All network references use VLAN names, zone designations, and role-based hostnames. Adapt addressing to your own RFC 1918 scheme.

---

## Table of Contents

- [Design Philosophy](#design-philosophy)
- [Zero Trust Model](#zero-trust-model)
- [Network Zones](#network-zones)
- [VLAN Design](#vlan-design)
- [Physical Topology](#physical-topology)
- [NIC Allocation](#nic-allocation)
- [Inter-Zone Traffic Policy](#inter-zone-traffic-policy)
- [Design Decisions](#design-decisions)

---

## Design Philosophy

This network is designed around three core network security principles:

**1. Explicit Trust**
Nothing is trusted by default. Every device, regardless of physical location, must be authorized to communicate with every other zone. There are no implicit trust relationships. For example, a device on the "Family" VLAN cannot reach the "Management" VLAN just because they share the same physical switch.

**2. Physical Isolation for High Risk Zones**
VLAN hopping attacks, where a misconfiguration with a switch could allow traffic to cross VLAN boundaries, are a known class of vulnerability. This home lab mitigates that risk for the two highest-risk zones (DMZ and IoT) by connecting them to **dedicated physical NIC interfaces** on the firewall rather than relying on VLAN tagging alone.

**3. Defense in Depth**
Security controls are layered. Traffic passes through: physical zone boundary → firewall rule enforcement → IDS/IPS inspection with DNS sinkhole and SIEM logging. No single control is relied upon exclusively.

---

## Zero Trust Model

This implementation is designed to map to **NIST SP 800-207** Zero Trust Architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                       CONTROL PLANE                             │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              POLICY DECISION POINT (PDP)                │    │
│  │  ┌─────────────────────┐  ┌──────────────────────────┐  │    │
│  │  │    Policy Engine    │  │   Policy Administrator   │  │    │
│  │  │  (OPNsense rules +  │  │  (Wazuh SIEM correlation │  │    │
│  │  │   Wazuh detection)  │  │   → dynamic block rules) │  │    │
│  │  └─────────────────────┘  └──────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │ Access decisions
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         DATA PLANE                              │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           POLICY ENFORCEMENT POINT (PEP)                │    │
│  │                OPNsense Firewall (fw01)                 │    │
│  │         Allow / Deny / Inspect / Log blocked            │    │
│  │    (Future: Alerts=Suricata, Auto-quarantine=Wazuh)     │    │
│  └─────────────────────────────────────────────────────────┘    │
│              │              │              │                    │
│      ┌───────┘       ┌──────┘        ┌─────┘                    │
│      ▼               ▼               ▼                          │
│  [ADMIN Zone]   [MGMT Zone]    [STOR Zone]   [DMZ]  [IoT]       │
│  Trust: High    Trust: High    Trust: Med    Low     None       │
└─────────────────────────────────────────────────────────────────┘
```
| **Full flow logging** | Planned Phase 10: Suricata NetFlow plugin |
| **Automated quarantine** | Planned Phase 11: Wazuh Active Response → OPNsense API and/or NAC Quarantine 802.1X on switch → restricted VLAN|


**Trust Levels by Zone:**

| Zone | Trust Level | Rationale |
|---|---|---|
| ADMIN (VLAN 10) | High | Admin workstation; controlled access |
| MGMT (VLAN 20) | High | Hypervisor management; only admins |
| STOR (VLAN 30) | Medium | Security tools/NAS; servers only, no general access |
| DMZ (VLAN 40) | Low | Internet-facing; treat as hostile-adjacent |
| IoT (VLAN 50) | None | Untrusted; blocked from all internal zones |
| FAM (VLAN 60) | Low | Family devices; internet only, no internal access |
| GUEST (VLAN 99) | None | Visitor isolation + red team node |

---

## Network Zones

### VLAN 10 — ADMIN
**Purpose:** Network infrastructure management and privileged administrator access.
**Devices:** Admin workstation (functions as PAW), OPNsense management interface, switch management, UniFi controller. Planned: Dedicated hardened jump server LXC (Phase TBD).
**Trust:** Highest; restricted to named admin hosts only via firewall alias.
**Internet:** Full outbound allowed.
**Cross-zone access:** Admin hosts can reach all other zones for management purposes.

### VLAN 20 — MGMT
**Purpose:** In-band hypervisor management. Isolates Proxmox management traffic from user VLANs. Restricts access to authorized admin hosts only.
**Devices:** Proxmox pve01, PBS (Phase 2).
**Trust:** High; only accessible from ADMIN zone.
**Internet:** Outbound for updates only (ports 80/443/123).
**Cross-zone access:** Hypervisors can reach STOR zone for NFS backup target.

### VLAN 30 — STOR
**Purpose:** Security infrastructure, storage, and internal services.
**Devices:** TrueNAS NAS, Wazuh SIEM, Velociraptor DFIR server, Pi-hole DNS, Headscale VPN coordinator, Gitea, Vaultwarden, Uptime Kuma.
**Trust:** Medium; servers only, no user devices.
**Internet:** Outbound for updates and threat intel feeds only.
**Cross-zone access:** Accepts log/agent traffic from all zones (inbound only); no outbound to other internal zones.

### VLAN 40 — DMZ
**Purpose:** Internet-exposed services and honeypots.
**Devices:** Nginx reverse proxy, T-Pot multi-honeypot.
**Trust:** Low; treat as hostile. Any DMZ servers are considered compromised in threat model.
**Physical isolation:** Connected to dedicated NIC (not VLAN tag) on OPNsense, VLAN hopping impossible.
**Cross-zone access:** **BLOCKED to all internal zones**. DMZ can only reach internet.

### VLAN 50 — IoT
**Purpose:** Smart home devices, cameras, untrusted consumer hardware.
**Devices:** Eufy HomeBase S380, smart appliances.
**Trust:** None; assumed hostile vendor telemetry, unpatched firmware.
**Physical isolation:** Dedicated NIC on OPNsense.
**Cross-zone access:** **BLOCKED to all internal zones**. Internet allowed on ports 80/443/123 only. DNS forced through Pi-hole.

### VLAN 60 — FAM
**Purpose:** Family devices, mobile phones, personal computers.
**Trust:** Low; personal use, not IT-managed.
**Cross-zone access:** **BLOCKED to all internal zones**. Internet only. DNS forced through Pi-hole.

### VLAN 99 — GUEST
**Purpose:** Visitor isolation and red team attack node (Kali).
**Trust:** None.
**Cross-zone access:** **BLOCKED to all internal zones** except specific firewall rules permitting Kali to attack designated test targets (STOR and FAM VMs only, never ADMIN or MGMT).

---

## Management Access
### In-Band Management (VLAN 20)
Proxmox access via the switched network. Isolated from user VLANs, but goes down if switch fails.

### Out-of-Band Management (NIC5 Direct)
Physical point-to-point cable: OPNsense to admin workstation. Bypasses all switching infrastructure.
Emergency console access when network is broken. Not a network zone: no routing, no firewall policy, no other devices.

### Possible OOB Expansion
PiKVM for pve01 and nas01. USB-serial for switch console.

---

## VLAN Design

| VLAN | Name | Subnet (Example) | Gateway | DHCP Range | Purpose |
|---|---|---|---|---|---|
| 10 | ADMIN | `[REDACTED]/24` | `[REDACTED]` | Static assignments | Network device mgmt (planned jump server) |
| 20 | MGMT | `[REDACTED]/24` | `[REDACTED]` | Static assignments | Hypervisor management |
| 30 | STOR | `[REDACTED]/24` | `[REDACTED]` | Static assignments | NAS, SIEM, DFIR, DNS, internal services |
| 40 | DMZ | `[REDACTED]/24` | `[REDACTED]` | Static assignments | Internet exposed services |
| 50 | IoT | `[REDACTED]/24` | `[REDACTED]` | `.100–.199` | Smart home, untrusted devices |
| 60 | FAM | `[REDACTED]/24` | `[REDACTED]` | `.100–.199` | Family devices |
| 99 | GUEST | `[REDACTED]/24` | `[REDACTED]` | `.100–.199` | Visitor isolation, red team node |

> Actual subnets and gateway addresses are redacted. All zones follow a consistent private RFC 1918 addressing scheme. Static DHCP assignments are maintained for all infrastructure devices.

---

## Physical Topology

```
ISP Modem (Bridge Mode)
        │
        │ WAN — Physical NIC 1 (onboard)
        │
┌───────▼──────────────────────────────────────────────────────────┐
│                   Dell OptiPlex 7040 SFF                         │
│                   OPNsense Firewall — fw01                       │
│                                                                  │
│  NIC1 (WAN)   NIC2 (LAN Trunk)   NIC3 (DMZ)    NIC4 (IoT)        │
│  Onboard      PCIe card port 1   PCIe card 2   PCIe card 3       │
│               Tagged: 10,20,30   Untagged 40   Untagged 50       │
│               60,99                                              │
│                                                                  │
│  NIC5 (OOB)                                                      │
│  PCIe card 4 — Direct cable to admin workstation                 │
└───────┬────────────────────────┬─────────────────────────────────┘
        │ LAN Trunk              │ OOB (direct cable)
        │                        │
┌───────▼────────────────┐  ┌────▼────────────────────────────────┐
│ TP-Link TL-SG2008P     │  │ Admin Workstation                   │
│ 8-port PoE Managed     │  │ (Omen 16 — VLAN 10 via OOB)         │
│                        │  └─────────────────────────────────────┘
│ Port 1: OPNsense LAN   │
│ Port 2: OPNsense DMZ   │
│ Port 3: OPNsense IoT   │
│ Port 4: UAP-AC-LITE    │
│ Port 5: G4 Mini (pve01)│
│ Port 6: Reserved (PBS) │
│ Port 7: G5 Mini (NAS)  │
│ Port 8: Workstation/etc│
└────────┬───────────────┘
         │
    ┌────┴──────────────────────────────────────────────┐
    │                                                   │
    ▼                                                   ▼
┌──────────────────────────────────────┐    ┌──────────────────────┐
│ HP EliteDesk 800 G4 Mini — pve01     │    │ UAP-AC-LITE          │
│ Proxmox VE 8.x Hypervisor            │    │ WPA3 Wireless        │
│                                      │    │                      │
│ LXCs:                                │    │ SSIDs:               │
│   pihole-01 (STOR)                   │    │  ARC-Admin  → V10    │
│   unifi-controller (ADMIN)           │    │  ARC-Family → V60    │
│   headscale (STOR)                   │    │  ARC-IoT    → V50    │
│   uptime-kuma (STOR)                 │    │  ARC-Guest  → V99    │
│   gitea (STOR)                       │    └──────────────────────┘
│   vaultwarden (STOR)                 │
│   nginx-proxy-mgr (DMZ)              │    ┌──────────────────────────────┐
│   n8n (STOR, planned)                │    │ HP EliteDesk 800 G5 Mini     │
│                                      │    │ TrueNAS SCALE — nas01        │
│ VMs:                                 │    │                              │
│   wazuh-mgr (STOR, planned)          │    │ ZFS mirror: 2× 4TB NVMe      │
│   velociraptor (STOR, planned)       │    │ Boot: 256GB SATA SSD         │
│   tpot-honeypot (DMZ, planned)       │    │                              │
│   dc01 Win Server (MGMT, planned)    │    │ Services:                    │
│   win10-test (FAM, planned)          │    │   NFS → pve01 backups        │
│   caldera (GUEST, planned)           │    │   SMB → workstation access   │
│                                      │    │   Encrypted off-site sync    │
│ RAM: 64GB — ample headroom           │    └──────────────────────────────┘
└──────────────────────────────────────┘
```

---

## NIC Allocation

The firewall's 5 NIC configuration is a deliberate architecture decision:

| NIC | Physical Source | Role | VLAN Mode | Security Rationale |
|---|---|---|---|---|
| NIC1 | Onboard | WAN | None | Onboard used for WAN; most isolated from the PCIe card |
| NIC2 | PCIe card — port 1 | LAN Trunk | Tagged: 10, 20, 30, 60, 99 | Carries all software defined VLANs |
| NIC3 | PCIe card — port 2 | DMZ | Untagged: 40 | **Physical isolation:** DMZ traffic never on trunk |
| NIC4 | PCIe card — port 3 | IoT | Untagged: 50 | **Physical isolation:** IoT traffic never on trunk |
| NIC5 | PCIe card — port 4 | OOB Management | Static ADMIN | Direct workstation cable; out-of-band firewall access |

**Why physical isolation matters:** VLAN misconfiguration on a trunk port could allow VLAN 40 (DMZ) or VLAN 50 (IoT) traffic to reach internal zones. By placing DMZ and IoT on dedicated physical interfaces, no misconfiguration on the LAN trunk can bridge them to internal VLANs (VLAN 40 and VLAN 50 do not exist on that wire). VLAN 40 is present on the hypervisor trunk to support DMZ VM hosting on pve01, but inter-VLAN traffic on that path still traverses OPNsense for policy enforcement. OPNsense remains the enforcement boundary regardless of switching topology.

---

## Inter-Zone Traffic Policy

All inter-zone communication is **default deny**. The following matrix shows allowed traffic directions:

| Source Zone | ADMIN | MGMT | STOR | DMZ | IoT | FAM | GUEST | Internet Access |
|---|---|---|---|---|---|---|---|---|
| **ADMIN** | ✅ | ✅ | ✅ | ✅ (full via RFC1918 rule) | ✅ (full via RFC1918 rule) | ✅ (full via RFC1918 rule) | ✅ (full via RFC1918 rule) | Full outbound |
| **MGMT** | ❌ | ✅ | ✅ (NFS/DNS/NTP/Wazuh) | ❌ | ❌ | ❌ | ❌ | HTTP/HTTPS/NTP only |
| **STOR** | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | HTTP/HTTPS, email ports |
| **DMZ** | ❌ | ❌ | (Wazuh agent ports only) | ✅ | ❌ | ❌ | ❌ | HTTP/HTTPS only |
| **IoT** | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | HTTP/HTTPS/NTP only |
| **FAM** | ❌ | ❌ | (Wazuh agent ports only) | ❌ | ❌ | ✅ | ❌ | Full outbound |
| **GUEST** | ❌ | ❌ | (Wazuh agent ports only) | ❌ | ❌ | (test targets only) | ✅ | Full outbound (rate-limited) |

> **All zones:** Outbound internet allowed on ports 80/443 minimum. STOR additionally allows outbound email ports (25/465/587) for SIEM alerts. DNS forced through Pi-hole via NAT redirect on port 53. NTP allowed to authorized time servers only.
Note on Phase 11 (not yet implemented): ADMIN zone has broad RFC1918 access via admin rule on IoT/DMZ/FAM/GUEST VLANs access is full, not management-restricted. Consider tightening to specific management ports post-Phase 11.

---

## Design Decisions

### Why OPNsense Over pfSense

OPNsense was selected over pfSense for the following reasons:

1. **Active open-source development:** OPNsense follows a faster release cadence with more frequent security updates
2. **Modern UI:** cleaner interface with better plugin ecosystem (CrowdSec, HAProxy, Suricata)
3. **ZFS support on bare metal:** native ZFS installation option provides snapshot/rollback capability on the firewall itself
4. **FreeBSD base with active security patching:** maintains a close upstream sync with FreeBSD security advisories and applies patches rapidly
5. **Enterprise-comparable:** zone-based firewall concepts, stateful inspection, NAT, VPN, and IDS/IPS administration patterns transfer to FortiOS, pfSense Enterprise, and Check Point. Conceptual familiarity with Palo Alto PAN-OS architecture (App-ID and Security Zone model).

### Why Proxmox Over VMware ESXi

1. **Fully open source:** no licensing cost, full feature access
2. **LXC container support:** lightweight containers for simple services (Pi-hole, Gitea) alongside full VMs for complex workloads (Wazuh, Windows lab)
3. **Ceph and PBS integration:** native backup server and future cluster expansion path
4. **Enterprise-comparable:** skill transfer directly to VMware vSphere administration

### Why Physical Isolation for DMZ and IoT

Standard home lab implementations place all VLANs on a single NIC using 802.1Q tagging. This design uses dedicated physical interfaces for VLAN 40 (DMZ) and VLAN 50 (IoT) because:

1. **Eliminates VLAN hopping attack surface:** no switch misconfiguration can bridge these zones to the internal trunk
2. **Demonstrates enterprise physical segmentation awareness;** many enterprise environments physically separate DMZ switching from corporate LAN
3. **Mirrors DoD network segmentation practices:** controlled interfaces and physical separation of sensitive from non-sensitive traffic

### Why Headscale Over Tailscale Cloud

Headscale is a self-hosted, open-source implementation of the Tailscale control plane (DERP/coordination server):

1. **Zero reliance on third-party SaaS:** no vendor dependency for VPN coordination
2. **Full control over access policies and device enrollment**
3. **Protocol-level VPN understanding:** not just "install Tailscale and click"
4. **Differentiator:** operating a self-hosted mesh VPN coordinator

### Why TrueNAS SCALE Over FreeNAS / TrueNAS CORE

1. **Linux-based (Debian):** broader application support, Docker-based app ecosystem
2. **ZFS native:** same proven filesystem, enterprise-grade data integrity
3. **MinIO integration:** S3-compatible object storage for Wazuh log archiving and application data
4. **Enterprise-comparable:** ZFS administration, dataset management, snapshot policies, NFS/SMB/iSCSI protocols, and replication concepts transfer to enterprise NAS platforms (NetApp ONTAP ZFS clusters)
---

*See [setup/](../setup/) for phase-by-phase implementation guides.*
*See [progress/changelog.md](../progress/changelog.md) for build history.*
