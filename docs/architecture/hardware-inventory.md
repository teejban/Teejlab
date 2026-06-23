# Hardware Inventory

The lab is a 2-node Proxmox cluster with a Raspberry Pi acting as both NAS and
cluster QDevice. One node (`teejhost2`) carries the routing/firewall workload as
an OPNsense VM and has the extra NICs to do it; the other (`teejhost1`) has a
single NIC but is the larger node (more RAM + a second disk) and hosts the PBS VM.
Both run **Proxmox VE 8.4.19**. Cluster totals: **12 threads** (2× Intel i5-8400T,
6c/6t each), **~93 GiB RAM** (62 + 31), **~1.4 TB raw disk**.

## Proxmox Nodes

### teejhost1 (compute)
- **Type**: Lenovo ThinkCentre M920q (mini PC)
- **CPU**: Intel Core i5-8400T @ 1.70 GHz (6 cores / 6 threads)
- **RAM**: 62 GiB
- **Storage**: 2 × ~512 GB — Samsung 850 PRO 512 GB (SATA, `sda`) + KIOXIA 512 GB (NVMe, `nvme0n1`)
- **NICs**: 1 × onboard NIC (`eno1`)
- **Management IP**: `10.0.10.2` (MGMT VLAN 10, on tagged sub-interface `vmbr0.10`;
  default gateway `10.0.10.1`). Flat IP removed — `vmbr0` is now `manual` (no host IP),
  still bridging the PBS VM and the VLAN trunk.
- **Switch port**: Port 3 (trunk: untagged VLAN 1 + tagged VLAN 10)
- **Proxmox Role**: Cluster member (pure compute node); also hosts the PBS VM
- **Notes**: Single NIC, so management rides a tagged VLAN over the one trunk.
  Fully on VLAN 10 — flat IP removed after corosync moved off it. Runs the Proxmox
  Backup Server as VMID 101 (`pbs`, now VLAN-10-native at `10.0.10.6`).

### teejhost2 (compute + OPNsense host)
- **Type**: Lenovo ThinkCentre M920q (mini PC)
- **CPU**: Intel Core i5-8400T @ 1.70 GHz (6 cores / 6 threads)
- **RAM**: 31 GiB
- **Storage**: 1 × ~500 GB — TEAM TM8FPK500G (NVMe, `nvme0n1`)
- **NICs**: 4 × NIC (onboard + added 4-port card)
  - `enp2s0f0` → vmbr0 (VLAN-aware) → OPNsense LAN trunk → switch Port 5
  - `enp2s0f1` → vmbr1 → OPNsense WAN (direct cable to travel router LAN)
  - `enp2s0f2`, `enp2s0f3`, `eno1` → spare (down)
- **Management IP**: `10.0.10.3` (MGMT VLAN 10, on tagged sub-interface `vmbr0.10`;
  default gateway `10.0.10.1`). Flat IP removed — `vmbr0` is now `manual`, still
  carrying the OPNsense LAN trunk. `vmbr0` was already VLAN-aware, so no switch change.
- **Switch port**: Port 5 (trunk to OPNsense; already carries VLAN 10 tagged)
- **Proxmox Role**: Cluster member + OPNsense host
- **Notes**: Hosts the OPNsense VM (VMID 100, named `edge` — the lab's actual
  router/firewall). Both the switch trunk and the direct WAN cable terminate here.
  Highest-stakes node for management changes, since breaking `vmbr0` takes down the router.

## Storage & Quorum

### teejlab-pi-nas (NAS + QDevice)
- **Model**: Raspberry Pi 5 (8 GB)
- **RAM**: 8 GB
- **Storage**: 4 × SSD on a SATA HAT, ZFS pool `teejlab-tank` (~1.32 TiB)
- **OS**: OpenMediaVault 7
- **NIC**: `eth0`
- **IP**: `10.0.10.4` (MGMT VLAN 10, on tagged sub-interface `eth0.10`; default
  gateway `10.0.10.1`). Flat IP removed via the OMV UI (eth0 IPv4 set to Disabled).
- **Switch port**: Port 2 (trunk: untagged VLAN 1 + tagged VLAN 10)
- **Purpose**: SMB target for Proxmox + other clients **and** cluster QDevice
  (quorum tie-breaker for the 2-node cluster). The Proxmox `teejlab-share` CIFS
  storage now points at `10.0.10.4`; the QDevice runs on `10.0.10.4` too.
- **Network**: Wired Ethernet to switch. VLAN interface added via the OMV UI
  (OMV owns the network config, so it is not hand-edited).

### Proxmox Backup Server
- **Deployment**: VM (VMID 101, name `pbs`) on teejhost1; hostname `pbs` → `pbs.teejlab.dev`
- **IP**: `10.0.10.6` (MGMT VLAN 10, NIC tagged `tag=10`) — rebuilt VLAN-10-native
- **Datastore**: `teejlab-backups` at `/mnt/pbs-backups`, backed by NFS from the Pi
  (`10.0.10.4:/export/teejlab-backups`, export needs `no_root_squash`, folder owned uid 34)
- **Cluster storage**: added as `pbs` (server `10.0.10.6`, user `root@pam`, fingerprint-trusted)
- **Status**: Running; test backup of OPNsense verified
- **Notes**: Rebuilt from scratch on VLAN 10 rather than migrated (see
  `docs/runbooks/pbs-rebuild.md`). This closed the last flat-net dependency in the lab.

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
- **Deployment**: VM (VMID 100, display name `edge`) on teejhost2, dedicated NICs
  (WAN `enp2s0f1` → vmbr1, LAN trunk `enp2s0f0` → vmbr0)
- **OS hostname**: `opnsense`; managed at **`edge.teejlab.dev`** (`10.0.10.1`), real
  Let's Encrypt cert via the `os-acme-client` plugin (DNS-01)
- **Purpose**: Internal routing, firewall, inter-VLAN routing, DHCP (Kea), DNS
  (Unbound), Tailscale subnet router
- **Status**: **Active** — routing, DHCP on five VLANs, internal DNS, and Tailscale all live
- **Notes**: Holds `.1` on every VLAN subnet — the lab's real router. The `edge`
  management record is a host override → `10.0.10.1`; the OS hostname is deliberately
  *not* `edge` (a multi-homed firewall self-registers to all its interface IPs). The
  two WAN-side `192.168.8.x` addresses are the OPNsense WAN (DHCP lease from the travel
  router) — _(confirm whether the second is a VIP/leftover)_.

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
| teejhost1       | Lenovo M920q (1 NIC)         | Compute + PBS host, 10.0.10.2 (VLAN 10)       | Active  |
| teejhost2       | Lenovo M920q (4 NICs)        | Compute + OPNsense host, 10.0.10.3 (VLAN 10)  | Active  |
| teejlab-pi-nas  | Raspberry Pi 5 (8 GB)        | ZFS ~1.32 TiB, SMB + QDevice, 10.0.10.4       | Active  |
| PBS (`pbs`)     | VM 101 on teejhost1          | Backup server, 10.0.10.6, datastore on Pi/NFS | Active  |
| Switch          | TP-Link TL-SG108E            | 8-port 802.1Q managed                         | Active  |
| Travel router   | GL-iNet GL-A1300 (Slate Plus)| OpenWRT, WiFi-as-WAN, .1                       | Active  |
| OPNsense (`edge`) | VM 100 on teejhost2        | Router/FW/DNS/Tailscale, `edge.teejlab.dev`   | Active  |

**Cluster totals**: 12 threads (2× i5-8400T 6c/6t) · ~93 GiB RAM (62 + 31) · ~1.4 TB raw disk · Proxmox VE 8.4.19

---
**Last Updated**: 2026-06-23
**Next Step**: Network foundation essentially complete — VLAN 10 migration, internal DNS, and
TLS (real Let's Encrypt certs on all four infra UIs) are done. Next: first **Docker host +
reverse proxy** (Caddy/Traefik) for automatic TLS on services, then the firewall hardening
pass, then the lab mail server. TLS exceptions: the **NAS UI** (no native ACME — front it via
the reverse proxy later) and the **switch** (HTTP-only hardware — not a TLS candidate).
