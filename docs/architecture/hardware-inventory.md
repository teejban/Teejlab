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
- **NICs**: 1 × onboard NIC (`eno1`)
- **Management IP**: `192.168.8.119` (flat, on `vmbr0`) **and** `10.0.10.2` (MGMT
  VLAN 10, on tagged sub-interface `vmbr0.10`) — dual-homed
- **Switch port**: Port 3 (trunk: untagged VLAN 1 + tagged VLAN 10)
- **Proxmox Role**: Cluster member (pure compute node); also hosts the PBS VM
- **Notes**: Single NIC, so management rides a tagged VLAN over the one trunk.
  Dual-homed during transition — flat IP retained for corosync until the
  cluster-wide ring migration. Runs the Proxmox Backup Server as VMID 101.

### teejhost2 (compute + OPNsense host)
- **Type**: Lenovo ThinkCentre M920q (mini PC)
- **CPU**: _(not yet recorded — see cluster total)_
- **RAM**: _(not yet recorded — see cluster total)_
- **Storage**: _(not yet recorded — see cluster total)_
- **NICs**: 4 × NIC (onboard + added 4-port card)
  - `enp2s0f0` → vmbr0 (VLAN-aware) → OPNsense LAN trunk → switch Port 5
  - `enp2s0f1` → vmbr1 → OPNsense WAN (direct cable to travel router LAN)
  - `enp2s0f2`, `enp2s0f3`, `eno1` → spare (down)
- **Management IP**: `192.168.8.138` (flat, on `vmbr0`) **and** `10.0.10.3` (MGMT
  VLAN 10, on tagged sub-interface `vmbr0.10`) — dual-homed. `vmbr0` was already
  VLAN-aware (it trunks all VLANs to OPNsense), so this needed no switch change.
- **Switch port**: Port 5 (trunk to OPNsense; already carries VLAN 10 tagged)
- **Proxmox Role**: Cluster member + OPNsense host
- **Notes**: Hosts the OPNsense VM (VMID 100 — the lab's actual router/firewall).
  Both the switch trunk and the direct WAN cable terminate here. Highest-stakes
  node for management changes, since breaking `vmbr0` takes down the router.

## Storage & Quorum

### teejlab-pi-nas (NAS + QDevice)
- **Model**: Raspberry Pi 5 (8 GB)
- **RAM**: 8 GB
- **Storage**: 4 × SSD on a SATA HAT, ZFS pool `teejlab-tank` (~1.32 TiB)
- **OS**: OpenMediaVault 7
- **NIC**: `eth0`
- **IP**: `192.168.8.230` (flat, on `eth0`) **and** `10.0.10.4` (MGMT VLAN 10, on
  tagged sub-interface `eth0.10`) — dual-homed
- **Switch port**: Port 2 (trunk: untagged VLAN 1 + tagged VLAN 10)
- **Purpose**: SMB target for Proxmox + other clients **and** cluster QDevice
  (quorum tie-breaker for the 2-node cluster)
- **Network**: Wired Ethernet to switch. VLAN interface added via the OMV UI
  (OMV owns the network config, so it is not hand-edited).

### Proxmox Backup Server
- **Deployment**: VM (VMID 101) on teejhost1
- **Storage Target**: _(not yet recorded — likely the Pi NAS SMB/ZFS share)_
- **Status**: Running
- **Notes**: Runs with the Proxmox firewall enabled on its VM NIC. Stays on the
  flat network (untagged VLAN 1) after teejhost1's bridge went VLAN-aware.

## Network Hardware

### Switch
- **Model**: TP-Link TL-SG108E
- **Type**: 8-port Easy Smart managed switch
- **VLAN Support**: Yes (802.1Q)
- **PoE**: No
- **Port assignments**:
  - Port 2 → teejlab-pi-nas (trunk: untagged VLAN 1, tagged VLAN 10)
  - Port 3 → teejhost1 (trunk: untagged VLAN 1, tagged VLAN 10)
  - Port 4 → travel router uplink (legacy flat network, kept during transition)
  - Port 5 → teejhost2 (trunk to OPNsense: untagged VLAN 1, tagged 10/20/30/40/50)
  - Port 6 → VLAN 30 access port (lab/test clients, PVID 30)
  - Ports 1, 7, 8 → free
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
| teejhost1       | Lenovo M920q (1 NIC)         | Compute + PBS host, .119 / .10.2 (VLAN 10)    | Active  |
| teejhost2       | Lenovo M920q (4 NICs)        | Compute + OPNsense host, .138 / .10.3 (VLAN 10) | Active  |
| teejlab-pi-nas  | Raspberry Pi 5 (8 GB)        | ZFS ~1.32 TiB, SMB + QDevice, .230 / .10.4    | Active  |
| PBS             | VM 101 on teejhost1          | Proxmox Backup Server                         | Active  |
| Switch          | TP-Link TL-SG108E            | 8-port 802.1Q managed                         | Active  |
| Travel router   | GL-iNet GL-A1300 (Slate Plus)| OpenWRT, WiFi-as-WAN, .1                       | Active  |
| OPNsense        | VM on teejhost2              | Router/firewall, dedicated WAN + LAN NICs     | Active  |

**Cluster totals**: 12 CPUs · ~94 GiB RAM · ~888 GiB storage

---
**Last Updated**: 2026-06-20
**Next Step**: Corosync cutover to VLAN 10 (promote `ring1`, drop the flat `ring0`,
move the QDevice off `192.168.8.230`), then deprecate the flat network. All three
nodes are dual-homed and corosync is redundant over both links. Still to capture:
per-node CPU model, RAM, and storage (`pveversion`, `lscpu`, `free -h`, `lsblk`);
PBS storage target; NIC MACs if MAC-based DHCP reservations become needed.
