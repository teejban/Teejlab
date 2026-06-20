# 0005. Single-NIC management via VLAN dual-homing

- **Status**: Accepted
- **Date**: 2026-06-20

## Context

Management is moving off the transitional flat network onto a dedicated MGMT VLAN (10).
But teejhost1 has only one NIC, and the cluster nodes' corosync rings currently ride their
flat IPs — so a node's management IP can't simply be swapped to a new subnet without
disturbing the cluster.

## Decision

**Dual-home** each node over its single NIC: keep the existing untagged flat IP on the bridge
and add a tagged VLAN 10 sub-interface (`vmbr0.10` on Proxmox, `eth0.10` on the OMV Pi) for
the new management IP. Make the switch port a trunk (untagged VLAN 1 PVID + tagged VLAN 10).
Leave **corosync on the flat net** for now; migrate the ring cluster-wide as a separate,
coordinated change once all nodes are dual-homed.

## Consequences

- Management moves to VLAN 10 with **no risk of lockout** — the flat IP is retained as a
  fallback and continues to carry corosync.
- No new hardware needed; one cable carries both networks via 802.1Q tagging.
- The flat network can't be fully deprecated until the corosync migration happens (it's a
  prerequisite, tracked separately).
- Only one default gateway is set across the two interfaces; flat-net peers remain reachable as
  a directly-connected route.

## Alternatives considered

- **Add a second NIC** to teejhost1: defeats the point — single-NIC trunking is the more
  portable, more interview-worthy solution, and plenty of real hosts run this way.
- **Swap `ring0_addr` to VLAN 10 per node, one at a time**: would split the corosync ring,
  since all members must share a low-latency network — hence the deferral to a coordinated move.
- **WiFi/AP onto the VLAN for clients**: separate concern; the travel router's WiFi sits
  upstream of OPNsense and can't place clients on a downstream VLAN without a dedicated AP.
