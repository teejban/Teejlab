# 0006. Tailscale for remote and cross-VLAN access

- **Status**: Accepted
- **Date**: 2026-06-20

## Context

Once management moved onto MGMT VLAN 10, the flat-network workstation could no longer reach it
(flat → VLAN can't initiate, since the flat net is on OPNsense's WAN side). Some path onto the
VLANs was needed — both to manage the lab day-to-day and as a prerequisite for deprecating the
flat network. Shared housing rules out a conventional inbound VPN: no public IP, no port
forwarding through the landlord's network.

## Decision

Run **Tailscale on OPNsense as a subnet router**, advertising the lab subnets (starting with
`10.0.10.0/24`) into a private tailnet. Reach the lab from enrolled personal devices over that.

## Consequences

- Full management access (web + SSH) to `10.0.10.x` from anywhere, with **no port forwarding** —
  the same outbound-only model that makes Cloudflare Tunnel work for inbound here.
- Decouples lab management from the flat network, unblocking the VLAN 1 deprecation and
  de-risking the corosync cutover (a way in survives flat-side hiccups).
- Adds a dependency on Tailscale's coordination service for connection setup (not for carrying
  traffic, which is peer-to-peer). Acceptable for a homelab.
- Subnet-routed hosts use **normal SSH/credentials**; Tailscale's keyless SSH feature is not
  used (it only applies to hosts running the client directly).

## Alternatives considered

- **Traditional hub-and-spoke VPN (OpenVPN/WireGuard server)**: needs inbound reachability
  (public IP / port forward) the housing situation can't provide.
- **Move the admin workstation onto a VLAN** (via an AP or wired access port): needs hardware
  not currently on hand, and doesn't give remote access. Tailscale does both with none.

## Notes

- Personal identity only; the work/Zscaler device is deliberately not enrolled.
- Key expiry disabled on the OPNsense subnet-router node so it doesn't drop off the tailnet.
- See `docs/runbooks/tailscale-subnet-router.md` for the setup procedure.
