# Hardware Inventory

## Proxmox Nodes

### Node 1 (Primary)
- **Type**: MiniPC with 2 NICs
- **CPU**: [Specify model/cores]
- **RAM**: [Specify amount and speed]
- **Storage**: [Specify drives, capacity, type]
- **NICs**: 
  - NIC 1 (primary): [Model/speed]
  - NIC 2 (secondary): [Model/speed]
- **Proxmox Role**: Cluster member + OPNsense host
- **Notes**: Runs OPNsense VM for routing/firewall

### Node 2 (Secondary)
- **Type**: MiniPC with 1 NIC
- **CPU**: [Specify model/cores]
- **RAM**: [Specify amount and speed]
- **Storage**: [Specify drives, capacity, type]
- **NIC**: [Model/speed]
- **Proxmox Role**: Cluster member
- **Notes**: 

## Storage & Quorum

### Raspberry Pi NAS (qdevice)
- **Model**: [Pi 4 / Pi 5 / etc]
- **RAM**: [Specify]
- **Storage**: [External USB / SSD details]
- **OS**: Proxmox VE on Pi (or Debian running qdevice daemon)
- **Purpose**: Cluster quorum tie-breaker
- **Network**: Connected via wired Ethernet

### Proxmox Backup Server
- **Deployment**: [VM on Node 1/2 or Dedicated hardware?]
- **Storage Target**: [Local path or NAS share?]
- **Status**: Running
- **Notes**: 

## Network Hardware

### Switch
- **Model**: TP-Link TL-SG108E
- **Type**: 8-port Easy Smart managed switch
- **VLAN Support**: Yes (802.1Q)
- **PoE**: No
- **Uplink**: Connected to travel router
- **Notes**: All Proxmox nodes, Pi-NAS, and DMZ endpoints connect here

### Travel Router (Uplink)
- **Model**: [Specify]
- **Purpose**: Bridged connection to landlord's WiFi
- **Network**: Provides WAN connectivity
- **Notes**: Renting environment — cannot modify upstream

### Planned: OPNsense Router
- **Deployment**: VM on Node 1 (uses both NICs in bridge mode)
- **Purpose**: Internal routing, firewall, VLAN routing
- **Status**: Planned (Network Foundation phase)

## Network Uplink Path
```
Landlord's WiFi 
  ↓
Travel Router (WAN bridge)
  ↓
TP-Link Switch (VLAN trunk)
  ↓
Proxmox Nodes + OPNsense VM
```

## Summary Table
| Component | Model | Specs | Status |
|-----------|-------|-------|--------|
| Node 1 | MiniPC (2 NIC) | [CPU/RAM/Storage] | Active |
| Node 2 | MiniPC (1 NIC) | [CPU/RAM/Storage] | Active |
| Pi NAS | Raspberry Pi | [Model/Storage] | Active |
| PBS | [Where?] | [Storage] | Active |
| Switch | TP-Link TL-SG108E | 8-port managed | Active |
| Router | Travel Router | [Model] | Active |
| OPNsense | Planned VM | 2 NICs passthrough | Planned |

---
**Last Updated**: 2026-06-18  
**Next Step**: Fill in bracketed specs and add network interface MAC addresses if needed for MAC-based reservations.
