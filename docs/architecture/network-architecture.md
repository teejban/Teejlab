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

All three nodes are now fully on **MGMT VLAN 10** (`10.0.10.2/3/4`), each via a tagged sub-interface on a single NIC over a trunked switch port; the flat IPs have been removed. Corosync and the QDevice also run on VLAN 10, so the cluster has no flat-net dependency. VLAN 1 is retained on the switch only as a break-glass recovery network.

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

`teejlab.dev` is owned, with DNS hosted on Cloudflare (chosen over Route 53 because the inbound plan, Cloudflare Tunnel, requires the zone there, and the same account provides the ACME DNS-01 API token). Internal services use `<service>.teejlab.dev`, resolved by OPNsense Unbound as split-horizon — see `docs/runbooks/internal-dns.md`.

TLS is **live** via Let's Encrypt **DNS-01** (Cloudflare). The four infra UIs — `teejhost1`/`teejhost2` (Proxmox), `pbs` (PBS), `edge` (OPNsense) — each hold real, auto-renewing certs via their native ACME clients. `.dev` is HSTS-preloaded (browsers force HTTPS and refuse the self-signed bypass), so valid certs are mandatory here, not cosmetic. Services spun up going forward get TLS centrally from a reverse proxy holding a `*.teejlab.dev` wildcard (see `docs/runbooks/tls-letsencrypt-dns01.md` and ADR-0009). The NAS UI (no native ACME) will be fronted by that proxy; the switch is HTTP-only hardware and not a TLS candidate.

## Constraints and tradeoffs

**Shared housing**: no control over upstream WiFi, no static IP, no port forwarding through the landlord's router. Inbound publishing uses Cloudflare Tunnel as a workaround.

**1-NIC asymmetry**: teejhost1 has a single NIC, teejhost2 has 4. OPNsense lives on teejhost2 to use dedicated NICs for WAN (`enp2s0f1`) and LAN trunk (`enp2s0f0`). teejhost1 carries its management (and any VM VLANs) tagged over its one trunk port via `vmbr0.10`.

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
- [x] Corosync second link (`ring1`) on VLAN 10 added and verified — redundant over flat + VLAN 10
- [x] Corosync cutover: single link on VLAN 10 (`ring0` = `10.0.10.x`), flat ring removed (needed a rolling corosync restart — link address change isn't live-reloadable)
- [x] QDevice moved to `10.0.10.4` (VLAN 10)
- [x] Flat IPs removed from all hosts — teejhost1, teejhost2, pi-nas now VLAN-10-only; default gateways on the `.10` interfaces
- [x] CIFS storage (`teejlab-share`) repointed from `192.168.8.230` to `10.0.10.4`
- [x] PBS rebuilt VLAN-10-native (`pbs`, `10.0.10.6`), datastore on the Pi via NFS — closed the last flat-net dependency
- [x] Flat network retired from the lab — nothing in the cluster uses VLAN 1 anymore
- [~] VLAN 1 intentionally kept on the switch as a break-glass recovery network (the Easy Smart switch's own management IP lives there); not deprecated by design
- [x] Tailscale deployed on OPNsense for remote/cross-VLAN access (subnet router for `10.0.10.0/24`)
- [x] Internal DNS — Unbound split-horizon for `*.teejlab.dev`; every host points at `10.0.10.1`
- [x] TLS — real Let's Encrypt certs (DNS-01 / Cloudflare) on all four infra UIs, auto-renewing
- [ ] Reverse proxy + `*.teejlab.dev` wildcard for automatic TLS on spun-up services
- [ ] Firewall hardening pass (now safe — management fully on VLAN 10 + Tailscale)

## Future considerations

- First Docker host + reverse proxy (Caddy/Traefik) holding a `*.teejlab.dev` wildcard — automatic TLS for any service spun up
- Firewall hardening pass (default-deny inter-VLAN)
- Migrate the switch's management onto VLAN 10 (firmware-permitting) and fully retire VLAN 1
- WiFi access points broadcasting per-VLAN SSIDs (post-housing-change)

_(Done since first draft — see Migration status: internal DNS via Unbound, Let's Encrypt DNS-01 certs on the infra UIs, Tailscale remote access.)_
