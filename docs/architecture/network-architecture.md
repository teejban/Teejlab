# Network Architecture

## Overview

The homelab runs on a 2-node Proxmox cluster behind an OPNsense router/firewall, with Layer-2 segmentation provided by a TP-Link managed switch. The design is constrained by shared housing — upstream internet comes from the landlord's WiFi through a GL-iNet travel router, which limits inbound connectivity but doesn't restrict the internal lab design.

OPNsense acts as the lab's actual router: handling DHCP, DNS, inter-VLAN routing, and firewalling. The travel router is treated as an opaque ISP-equivalent — effectively the lab's "modem."

## Hardware

| Role        | Device                          | Notes                                              |
|-------------|---------------------------------|----------------------------------------------------|
| Hypervisor  | Lenovo ThinkCentre M920q ×2     | `teejhost1` (1 NIC), `teejhost2` (4 NICs)          |
| Storage     | Raspberry Pi 5 + 4-bay SATA HAT | `teejlab-pi-nas`, ZFS pool `teejlab-tank` (~1.3 TB)|
| Switch      | TP-Link TL-SG108E (Easy Smart)  | 8-port, 802.1Q VLAN capable                        |
| Upstream    | GL-iNet GL-A1300 (Slate Plus)   | Travel router, WiFi-as-WAN to landlord's network   |

## Physical topology

```
+---------------------+
|  Landlord's WiFi    |
|  (uncontrolled)     |
+----------+----------+
           | WiFi-as-WAN
+----------v----------+
|   GL-A1300 router   |
|   192.168.8.1       |
+--+---------------+--+
   | LAN port      | LAN port
   |               | (direct cable, not switched)
   v               v
[Switch port 4]  [teejhost2 enp2s0f1 -> vmbr1 -> OPNsense WAN]

+------------------------------------------------------+
| TP-Link TL-SG108E managed switch                     |
| Port 2: teejlab-pi-nas (trunk: untagged 1, tagged 10)|
| Port 3: teejhost1 (trunk: untagged 1, tagged 10)     |
| Port 4: Travel router uplink (legacy flat network)   |
| Port 5: teejhost2 enp2s0f0 (trunk to OPNsense LAN)   |
| Port 6: VLAN 30 access port (test/lab clients)       |
| Ports 1, 7, 8: free                                  |
+------------------------------------------------------+

[teejhost1: 1 NIC]    [teejhost2: 4 NICs, hosts OPNsense]    [teejlab-pi-nas]
                                                              (QDevice + SMB target)
```

Both the travel router uplink to the switch *and* the direct WAN cable to OPNsense terminate on **teejhost2**. teejhost1 is a pure compute node (also hosting the PBS VM) connected only to the switch.

teejhost1 and teejlab-pi-nas are now **dual-homed**: each keeps its flat IP for cluster/QDevice traffic and carries a tagged VLAN 10 management address (`10.0.10.2` and `10.0.10.4`) over the same single NIC, via a trunked switch port. teejhost2's management migration and the corosync ring move are still pending.

## VLAN scheme

| VLAN | Name    | Subnet         | Gateway     | Purpose                          |
|------|---------|----------------|-------------|----------------------------------|
| 1    | Default | 192.168.8.0/24 | 192.168.8.1 | Legacy flat net (transitional)   |
| 10   | MGMT    | 10.0.10.0/24   | 10.0.10.1   | Infrastructure management        |
| 20   | SERVERS | 10.0.20.0/24   | 10.0.20.1   | Production services              |
| 30   | LAB     | 10.0.30.0/24   | 10.0.30.1   | Ephemeral test workloads         |
| 40   | TRUSTED | 10.0.40.0/24   | 10.0.40.1   | Personal admin devices           |
| 50   | DMZ     | 10.0.50.0/24   | 10.0.50.1   | Internet-facing services         |
| 99   | IOT     | 10.0.99.0/24   | 10.0.99.1   | Defined in OPNsense, DHCP not enabled yet |

OPNsense holds `.1` on each subnet and acts as gateway, DHCP server, and DNS resolver.

## Switch port plan

- **Port 5 (trunk to OPNsense)**: untagged VLAN 1 (legacy), tagged VLANs 10, 20, 30, 40, 50
- **Port 3 (teejhost1)** and **Port 2 (teejlab-pi-nas)**: trunks — untagged VLAN 1 (PVID 1, for flat/cluster traffic), tagged VLAN 10 (management)
- **Port 6 (VLAN 30 access)**: untagged VLAN 30, PVID 30
- Additional access ports will be configured per VLAN as devices migrate from the flat network

## Traffic flow

A device on Port 6 (VLAN 30 access) sending traffic:

```
client -> switch tags as VLAN 30
       -> trunk port 5
       -> teejhost2 enp2s0f0
       -> Proxmox vmbr0 (VLAN-aware)
       -> tap interface (per-VM trunks=10;20;30;40;50)
       -> OPNsense vtnet0 + VLAN 30 sub-interface (LAB)
       -> Kea DHCP / firewall / routing
       -> OPNsense WAN (enp2s0f1 -> vmbr1, DHCP lease from travel router)
       -> travel router (NAT)
       -> landlord's WiFi (NAT)
       -> internet
```

The result is triple NAT (lab -> OPNsense -> travel router -> landlord). Acceptable for outbound. Inbound publishing for public services will use Cloudflare Tunnel rather than port forwarding.

## DHCP

OPNsense runs Kea DHCPv4. Each VLAN has its own subnet definition with the gateway, DNS, and domain auto-collected from the corresponding OPNsense interface. Pool sizes vary by expected client density:

| VLAN    | Subnet         | Pool                          | Rationale                            |
|---------|----------------|-------------------------------|--------------------------------------|
| MGMT    | 10.0.10.0/24   | 10.0.10.10 - 10.0.10.30       | Small; most infra is static          |
| SERVERS | 10.0.20.0/24   | 10.0.20.10 - 10.0.20.30       | Small; services use static IPs       |
| LAB     | 10.0.30.0/24   | 10.0.30.10 - 10.0.30.100      | Larger; ephemeral test workloads     |
| TRUSTED | 10.0.40.0/24   | 10.0.40.10 - 10.0.40.20       | Small; few personal devices          |
| DMZ     | 10.0.50.0/24   | 10.0.50.10 - 10.0.50.15       | Tiny; public-facing services static  |

IOT subnet is reserved but not yet served — added when IoT devices arrive.

## Firewall

Initial state (buildout phase): each VLAN has a permissive "pass any" rule to allow internet egress and inter-VLAN reachability while the architecture is validated.

Planned hardening pass:

- Default-deny inter-VLAN, with explicit allows per flow
- DMZ -> RFC1918 blocked (DMZ cannot initiate to private networks)
- TRUSTED -> any allowed (admin access)
- Specific service ports allowed between VLANs as needed

An `RFC1918` alias is already defined to support these future rules.

## Domain and TLS

`teejlab.dev` is owned, with DNS hosted on Cloudflare. Cloudflare was chosen over Route 53 because the inbound plan (Cloudflare Tunnel) requires the zone to live on Cloudflare, and the same account provides the API token for ACME DNS-01. Internal services will use the same domain (`<service>.teejlab.dev`). Let's Encrypt via DNS-01 challenge will issue valid certificates for internal hostnames — no browser warnings on lab-internal HTTPS services. `.dev` is HSTS-preloaded, so browsers force HTTPS on these hostnames by default.

## Constraints and tradeoffs

**Shared housing**: no control over upstream WiFi, no static IP, no port forwarding through the landlord's router. Inbound publishing uses Cloudflare Tunnel as a workaround.

**1-NIC asymmetry**: teejhost1 has a single NIC, teejhost2 has 4. OPNsense lives on teejhost2 to use dedicated NICs for WAN (`enp2s0f1`) and LAN trunk (`enp2s0f0`). teejhost1 will eventually carry all VLANs over a single trunk port.

**Triple NAT**: a consequence of the housing situation, not a design choice. Outbound is unaffected. Inbound is sidestepped entirely via Cloudflare Tunnel.

**Proxmox VLAN-aware bridges require per-VM trunks config**: a VLAN-aware bridge alone isn't enough. Each VM NIC's tap defaults to VLAN 1 only, so tagged frames get dropped before reaching the guest. The VM NIC needs `trunks=10;20;30;40;50` (or equivalent) in its config to be a member of the relevant VLANs. See `docs/runbooks/proxmox-vlan-trunks.md` for the full debugging writeup.

## Migration status

- [x] OPNsense VM built, WAN routing functional
- [x] VLANs defined on switch and OPNsense
- [x] DHCP serving all five active VLANs
- [x] Proxmox vmbr0 VLAN-aware on teejhost2
- [x] Per-VM trunks config on OPNsense VM NIC
- [x] End-to-end VLAN validation: client on Port 6 -> 10.0.30.X DHCP lease via OPNsense
- [x] teejhost1 management dual-homed onto MGMT VLAN 10 (`10.0.10.2`, tagged `vmbr0.10`)
- [x] teejlab-pi-nas management dual-homed onto MGMT VLAN 10 (`10.0.10.4`, tagged `eth0.10` via OMV)
- [x] teejhost2 management dual-homed onto MGMT VLAN 10 (`10.0.10.3`, tagged `vmbr0.10`)
- [x] Corosync second link (`ring1`) on VLAN 10 added and verified — redundant over flat + VLAN 10, both links connected
- [ ] Corosync cutover: promote VLAN 10 to primary, remove flat `ring0`, move QDevice off `192.168.8.230`
- [ ] Legacy flat network (VLAN 1) deprecated (blocked on the corosync cutover)
- [ ] Remaining VLAN access ports configured on switch; switch management onto VLAN 10 (firmware-permitting)
- [x] Tailscale deployed on OPNsense for remote/cross-VLAN access (subnet router for `10.0.10.0/24`)
- [ ] Firewall hardening pass (after VLAN migration completes)

## Future considerations

- WiFi access points broadcasting per-VLAN SSIDs (post-housing-change)
- Internal DNS records via OPNsense Unbound for lab hostnames under `teejlab.dev`
- Public Let's Encrypt certs for internal services via DNS-01
- Site-to-site or VPN access for cross-subnet connectivity from remote locations
