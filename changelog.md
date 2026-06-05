# Build Changelog

All significant changes to the lab are logged here with dates and context.

---

## Format

```
[Phase X] YYYY-MM-DD — Description
 What was done
 Issues encountered and resolved
 Current status
```

---

## [Phase 1] 2026-05-15 — OPNsense Firewall Installation

### What was done
- Installed OPNsense 24.x bare metal on Dell OptiPlex 7040 SFF
- Verified aftermarket 4-port Intel NIC detected (5 NICs total)
- ZFS stripe configured on 128GB NVMe boot drive
- Initial hardening applied: HTTPS only, HSTS enabled, management restricted to LAN
- Created non-root admin user with TOTP 2FA, root web login disabled
- Firmware updated to latest available release
- WAN interface assigned to onboard NIC, LAN to PCIe card port 1
- DMZ, IoT, and OOB interfaces assigned to remaining PCIe card ports

### Issues encountered
- PCIe card initially required reseating — not detected on first boot
- Identified M.2 slot as NVMe-capable via Linux live USB verification before install

### Status
Phase 1 complete. Firewall operational with basic WAN/LAN configuration.

---

## [Phase 2] 2026-05-16 — VLAN Configuration

### What was done
- Created 5 VLAN interfaces on LAN trunk: ADMIN (10), MGMT (20), STOR (30), FAM (60), GUEST (99)
- Confirmed DMZ (40) and IoT (50) remain on dedicated physical NICs — not created as VLANs
- DHCP configured for all 7 zones (static assignments for infrastructure, dynamic for IoT/FAM/GUEST)
- OPNsense interface mapping documented and saved offline

### Issues encountered
- Initial DHCP conflict on VLAN 10 — resolved by removing temporary static configuration

### Status
Phase 2 complete. All 7 network zones active with DHCP.

---

## [Phase 3] 2026-05-20 — Managed Switch Configuration

### What was done
- TP-Link TL-SG2008P management IP moved to ADMIN VLAN
- All 7 VLANs created on switch
- 802.1Q trunk configured on port 1 (OPNsense LAN) and port 5 (pve01)
- UAP-AC-LITE trunk configured on port 4 with VLANs 10, 50, 60, 99
- NAS access port on port 7 (untagged STOR)
- PoE enabled on port 4 for wireless AP
- Switch config saved and backed up

### Issues encountered
- VLAN 40 tagging on switch port 5 required to support DMZ-zone VMs hosted on pve01
- Resolved by including VLAN 40 on the hypervisor trunk port
- Switch management IP initially unreachable after moving to ADMIN VLAN, workstation VLAN not updated - resolved by setting workstation to ADMIN VLAN subnet temporarily

### Status
Phase 3 complete. All VLAN traffic correctly tagged and segmented.

---

## [Phase 4] 2026-05-24 — Proxmox VE Hypervisor

### What was done
- Proxmox VE 8.x installed bare metal on HP EliteDesk 800 G4 Mini
- Enterprise repo removed, no-subscription repo added
- VLAN-aware bridge (vmbr0) configured with bridge-vids 2-4094
- 2TB NVMe added as vm-pool ZFS storage pool
- LXC containers created: unifi-controller, headscale, uptime-kuma, gitea, vaultwarden
- Backup schedule configured targeting TrueNAS NFS (after Phase 5)
- SSH hardened: key-based auth only, root login disabled after admin user created

### Issues encountered
- Default Proxmox network config used non-VLAN-aware bridge — edited /etc/network/interfaces manually and applied with ifreload
- Subscription nag removed

### Status
Phase 4 complete. Hypervisor running, core LXC containers operational.

---

## [Phase 5] 2026-05-26 — TrueNAS SCALE NAS

### What was done
- TrueNAS SCALE 24.x installed on HP EliteDesk 800 G5 Mini
- Boot drive: 256GB SATA SSD (correctly installed to SATA bay, not NVMe slots)
- ZFS mirror pool "tank" created on both 4TB NVMe drives with AES-256-GCM encryption
- Datasets created: proxmox-backups, logs, configs, media
- NFS export configured for pve01 backup target
- SMB share configured for workstation access
- ZFS snapshot policy: every 6 hours, 14-day retention
- Proxmox NFS storage added and verified functional

### Issues encountered
- Confirmed G5 Mini second NVMe slot is PCIe 3.0 x2 (half-speed) — both drives still show tran=nvme, ZFS mirror healthy

### Status
Phase 5 complete. NAS operational, backups flowing from Proxmox.

---

## [Phase 6] 2026-05-29 — UniFi Wireless Configuration

### What was done
- UniFi Network Application installed in unifi-controller LXC (ADMIN VLAN)
- UAP-AC-LITE adopted and firmware updated
- 4 SSIDs configured with VLAN mapping:
  - ARC-Admin → VLAN 10 (hidden, WPA3, PMF required)
  - ARC-Family → VLAN 60 (WPA3/WPA2 transition)
  - ARC-IoT → VLAN 50 (WPA2, client isolation enabled)
  - ARC-Guest → VLAN 99 (WPA2, client isolation enabled)
- Band steering enabled, legacy 802.11b rates disabled
- SSID passphrases set and stored in Vaultwarden

### Issues encountered
- Initial AP adoption failed — UniFi controller not reachable from AP's default IP range
- Resolved: SSH to AP and manually set inform URL to UniFi controller IP

### Status
Phase 6 complete. Wireless operational across all 4 SSIDs.

---

## [Phase 7] 2026-06-01 — Firewall Rules and Inter-VLAN Policy

### What was done
- Created all named aliases: RFC1918, Admin_Hosts, Internal_DNS, Hypervisors, Storage_Servers, RedTeam_Host, Test_Targets, NTP_Servers
- Implemented per-zone rule sets with default deny baseline
- ADMIN: Full admin access to all zones + internet
- MGMT: NFS to STOR, updates only, no other internal access
- STOR: Internet for updates/threat feeds, no internal initiation
- DMZ: Blocked from all RFC1918, internet only
- IoT: Blocked from all RFC1918, internet only
- FAM: Internet only, blocked from all internal
- GUEST: Internet + red team authorized attack rules only
- Logging enabled on all block rules for SIEM ingestion
- Syslog forwarding target pre-configured to planned Wazuh manager IP — will activate automatically when Phase 11 deployment completes. No active SIEM ingestion until Phase 11.

### Issues encountered
- DNS forwarding NAT redirect rule initially positioned before the allow rule for the internal DNS resolver —  caused DNS failures on STOR VLAN immediately after applying rules. Resolved by reordering: internal DNS allow rule precedes both the port 53 block rule and the NAT redirect on all VLAN interfaces.
- MGMT zone initially had broad internet access — tightened to explicit allow on ports 80, 443, and 123 only after noticing the default allow is too permissive for a management-only zone.

### Status
Phase 7 complete. Zero Trust inter-VLAN policy fully operational and validated.

---

## [Phase 8] 2026-06-04 — VPN (In Progress)

### What was done
- WireGuard plugin installed on OPNsense
- Server and first peer (Omen laptop) configured
- Firewall rules for WireGuard interface added
- Headscale LXC running, initial configuration in place

### Remaining
- Enroll all lab devices in Headscale mesh
- Test remote access from external network (mobile data)
- Document client enrollment process

### Status
In Progress — core components installed, testing underway.

---

## Upcoming

| Phase | Target Date | Description |
|---|---|---|
| Phase 9 | TBD | Pi-hole + Unbound + DoT DNS security |
| Phase 10 | TBD | Suricata IDS/IPS inline deployment |
| Phase 11 | TBD | Wazuh SIEM all-in-one deployment |
| Phase 12 | TBD | Velociraptor DFIR server |
| Phase 13 | TBD | DMZ services + T-Pot honeypot |
| Phase 14 | TBD | Kali red team + Atomic Red Team |
| Phase 15 | TBD | Centralized logging pipeline + 3-2-1 backups |

---

*Maintained by Z.Angell — Last update June 05 2026*
