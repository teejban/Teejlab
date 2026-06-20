# Hardware Inventory

The lab is a 2-node Proxmox cluster with a Raspberry Pi acting as both NAS and
cluster QDevice. One node (`teejhost2`) carries the routing/firewall workload as
an OPNsense VM and has the extra NICs to do it; the other (`teejhost1`) is a
single-NIC compute node. Cluster totals: **12 CPUs, ~94 GiB RAM, ~888 GiB
storage**.

> Per-node CPU/RAM/storage splits below are marked _(not yet recorded)_ where the
> exact figure hasn't been captured from the node directly. The cluster totals
> above are confirmed.

## Proxmox Nodes

### teejhost1 (compute)
- **Type**: Lenovo ThinkCentre M920q (mini PC)
- **CPU**: _(not yet recorded — see cluster total)_
- **RAM**: _(not yet recorded — see cluster total)_
- **Storage**: _(not yet recorded — see cluster total)_
- **NICs**: 1 × onboard NIC
- **Management IP**: `192.168.8.119` (flat network; will move to MGMT VLAN 10)
- **Switch port**: Port 3
- **Proxmox Role**: Cluster member (pure compute node)
- **Notes**: Single NIC means everything will eventually trunk over one port.
  Currently flat-network only.

### teejhost2 (compute + OPNsense host)
- **Type**: Lenovo ThinkCentre M920q (mini PC)
- **CPU**: _(not yet recorded — see cluster total)_
- **RAM**: _(not yet recorded — see cluster total)_
- **Storage**: _(not yet recorded — see cluster total)_
- **NICs**: 4 × NIC (onboard + added 4-port card)
  - `enp2s0f0` → vmbr0 (VLAN-aware) → OPNsense LAN trunk → switch Port 5
  - `enp2s0f1` → vmbr1 → OPNsense WAN (direct cable to travel router LAN)
- **Management IP**: `192.168.8.138` (flat network; will move to MGMT VLAN 10)
- **Switch port**: Port 5 (trunk to OPNsense)
- **Proxmox Role**: Cluster member + OPNsense host
- **Notes**: Hosts the OPNsense VM (the lab's actual router/firewall). Both the
  switch trunk and the direct WAN cable terminate here.

## Storage & Quorum

### teejlab-pi-nas (NAS + QDevice)
- **Model**: Raspberry Pi 5 (8 GB)
- **RAM**: 8 GB
- **Storage**: 4 × SSD on a SATA HAT, ZFS pool `teejlab-tank` (~1.32 TiB)
- **OS**: OpenMediaVault 7
- **IP**: `192.168.8.230` (flat network; will move to MGMT VLAN 10)
- **Purpose**: SMB target for Proxmox + other clients **and** cluster QDevice
  (quorum tie-breaker for the 2-node cluster)
- **Network**: Wired Ethernet to switch

### Proxmox Backup Server
- **Deployment**: _(not yet recorded — confirm whether VM or dedicated)_
- **Storage Target**: _(not yet recorded — likely the Pi NAS SMB/ZFS share)_
- **Status**: Running
- **Notes**: In place as part of current tooling.

## Network Hardware

### Switch
- **Model**: TP-Link TL-SG108E
- **Type**: 8-port Easy Smart managed switch
- **VLAN Support**: Yes (802.1Q)
- **PoE**: No
- **Port assignments**:
  - Port 3 → teejhost1
  - Port 4 → travel router uplink (legacy flat network, kept during transition)
  - Port 5 → teejhost2 (trunk to OPNsense: untagged VLAN 1, tagged 10/20/30/40/50)
  - Port 6 → VLAN 30 access port (lab/test clients, PVID 30)
  - Ports 1, 2, 7, 8 → free
- **Notes**: Easy Smart quirks — 802.1Q config spans three menus, needs an
  explicit Save Config to persist across reboots, and needs at least one port
  member to create a VLAN.

### Travel Router (Upstream)
- **Model**: GL-iNet GL-A1300 (Slate Plus)
- **OS**: OpenWRT
- **IP**: `192.168.8.1`
- **Purpose**: WiFi-as-WAN to the landlord's network; treated as an opaque
  ISP-equivalent ("modem") for the lab
- **Notes**: Shared housing — cannot modify upstream. No static IP, no port
  forwarding through the building network. Inbound publishing handled via
  Cloudflare Tunnel.

### OPNsense Router (VM)
- **Deployment**: VM on teejhost2, using dedicated NICs
  (WAN `enp2s0f1` → vmbr1, LAN trunk `enp2s0f0` → vmbr0)
- **Purpose**: Internal routing, firewall, inter-VLAN routing, DHCP (Kea), DNS
- **Status**: **Active** — WAN routing functional, DHCP serving all five active
  VLANs, end-to-end VLAN validation passed
- **Notes**: Holds `.1` on every VLAN subnet. This is the lab's real router.

## Network Uplink Path
```
Landlord's WiFi (uncontrolled)
  ↓ WiFi-as-WAN
GL-A1300 travel router (192.168.8.1)
  ├─ LAN port → switch Port 4 (legacy flat net)
  └─ LAN port → teejhost2 enp2s0f1 → vmbr1 → OPNsense WAN (direct cable)
        ↓
OPNsense (routing / firewall / DHCP / DNS)
        ↓ enp2s0f0 → vmbr0 (VLAN-aware) → switch Port 5 (trunk)
TP-Link TL-SG108E switch (VLAN trunk/access ports)
  ↓
Clients / Proxmox nodes / Pi NAS
```

## Summary Table
| Component       | Model                        | Specs / Role                                  | Status  |
|-----------------|------------------------------|-----------------------------------------------|---------|
| teejhost1       | Lenovo M920q (1 NIC)         | Compute node, IP .119                         | Active  |
| teejhost2       | Lenovo M920q (4 NICs)        | Compute + OPNsense host, IP .138              | Active  |
| teejlab-pi-nas  | Raspberry Pi 5 (8 GB)        | ZFS `teejlab-tank` ~1.32 TiB, SMB + QDevice   | Active  |
| PBS             | _(deployment TBD)_           | Proxmox Backup Server                         | Active  |
| Switch          | TP-Link TL-SG108E            | 8-port 802.1Q managed                         | Active  |
| Travel router   | GL-iNet GL-A1300 (Slate Plus)| OpenWRT, WiFi-as-WAN, .1                       | Active  |
| OPNsense        | VM on teejhost2              | Router/firewall, dedicated WAN + LAN NICs     | Active  |

**Cluster totals**: 12 CPUs · ~94 GiB RAM · ~888 GiB storage

---
**Last Updated**: 2026-06-20
**Next Step**: Capture per-node CPU model, RAM, and storage from each node
(`pveversion`, `lscpu`, `free -h`, `lsblk`) and confirm PBS deployment/target.
Add NIC MAC addresses if MAC-based DHCP reservations become needed.
