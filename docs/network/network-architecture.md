# Network Plan

## Domain & DNS Resolution
- **Internal Domain**: `.internal` or `home.arpa` (TBD — choose before deploying DNS)
- **Internal DNS**: [Planned — AdGuard Home / Dnsmasq / OPNsense DNS? Choose one]
- **External DNS**: Cloudflare / quad9 (fallback for external lookups)
- **Split-horizon DNS**: Planned (internal IPs for .internal, external for everything else)

## VLAN Scheme

| VLAN ID | Name | Subnet | Purpose | Gateway |
|---------|------|--------|---------|---------|
| 10 | Management | 10.0.10.0/24 | Proxmox, PBS, Pi-NAS mgmt | 10.0.10.1 |
| 20 | Servers | 10.0.20.0/24 | Production VMs, containers | 10.0.20.1 |
| 30 | Lab | 10.0.30.0/24 | Development, testing | 10.0.30.1 |
| 40 | Trusted | 10.0.40.0/24 | Personal devices, laptops | 10.0.40.1 |
| 50 | DMZ | 10.0.50.0/24 | Public-facing services (resume site, etc) | 10.0.50.1 |
| 99 | IoT | 10.0.99.0/24 | Smart devices, untrusted | 10.0.99.1 |

## IP Address Reservations (VLAN 10: Management)

| Host | IP | MAC | Purpose |
|------|----|----|---------|
| OPNsense (mgmt NIC) | 10.0.10.2 | [MAC] | Firewall/router management |
| Node 1 (out-of-band) | 10.0.10.10 | [MAC] | Proxmox node access |
| Node 2 (out-of-band) | 10.0.10.11 | [MAC] | Proxmox node access |
| Pi-NAS (qdevice) | 10.0.10.20 | [MAC] | Cluster quorum device |
| PBS (Backup) | 10.0.10.30 | [MAC] | Backup server |
| DHCP pool | 10.0.10.100–10.0.10.254 | — | Guest/ephemeral access |

## Physical Network Topology

```
┌─────────────────────────────────────────────┐
│   Landlord's WiFi (WAN)                     │
└────────────────┬────────────────────────────┘
                 │
        ┌────────▼──────────┐
        │  Travel Router    │
        │  (WAN bridge)     │
        └────────┬──────────┘
                 │
        ┌────────▼──────────────────────────────┐
        │  TP-Link TL-SG108E (Managed Switch)   │
        │  Port 1: OPNsense NIC1 (tagged)       │
        │  Port 2: Node 1 primary (VLAN 10)     │
        │  Port 3: Node 2 primary (VLAN 10)     │
        │  Port 4: Pi-NAS (VLAN 10)             │
        │  Port 5: [Reserved]                   │
        │  Port 6: [Reserved]                   │
        │  Port 7: [Reserved]                   │
        │  Port 8: Uplink to Travel Router      │
        └────────┬──────────────────────────────┘
                 │
        ┌────────▼─────────────────────────────┐
        │    OPNsense Router VM (Node 1)       │
        │  NIC1: Uplink (VLAN trunk)           │
        │  NIC2: Internal routing              │
        └────────┬──────────────────────────────┘
                 │
        ┌────────▼────────────────────────────┐
        │  Proxmox Cluster                    │
        │  Node 1 + Node 2 (+ Pi qdevice)     │
        │  ↓ Hosts Containers/VMs             │
        └─────────────────────────────────────┘
```

## OPNsense Interface Plan

### WAN Side (Uplink)
- **Interface**: NIC1 (Node 1 PCI passthrough)
- **Mode**: VLAN trunk (802.1Q) or flat bridge to travel router
- **VLAN Handling**: Receive untagged + handle optional VLAN traffic from upstream
- **Default Route**: Via travel router

### LAN Side (Internal)
- **Interface**: NIC2 or bridge on Node 1
- **Mode**: Internal routing to Proxmox
- **Firewall Rules**: (TBD in Phase 1)

## Routing & Firewall Firewall Policies (Draft)

- **Inbound (WAN → LAN)**: Block all except DMZ exceptions
- **DMZ (50) → WAN**: Allow (for public services)
- **Trusted (40) → All**: Allow (admin access)
- **Lab (30) → Servers (20)**: Allow (testing, with logging)
- **IoT (99) → LAN**: Block (isolated, no egress)
- **Servers (20) → Management (10)**: Rate-limit or block
- **Inter-VLAN**: All others deny by default

## Current Network State
- **Status**: Flat network (all traffic on one segment)
- **Next Step**: VLAN migration (Phase 1)

## Post-Migration Verification Checklist
- [ ] OPNsense boots and reaches management VLAN
- [ ] Node 1 & 2 still reach Proxmox via VLAN 10
- [ ] DNS resolution working (internal + external)
- [ ] VLAN isolation confirmed (cross-VLAN pings fail as expected)
- [ ] Firewall logs show expected rule hits
- [ ] All VMs can access their assigned VLAN gateway

---
**Last Updated**: 2026-06-18  
**Current Phase**: Pre-VLAN migration  
**Blockers**: None identified
