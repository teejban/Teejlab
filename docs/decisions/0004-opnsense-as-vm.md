# 0004. OPNsense as a VM, not a hardware appliance

- **Status**: Accepted
- **Date**: 2026-06-18

## Context

The lab needs a firewall/router for VLAN routing and inter-VLAN policy. Options ranged from a
dedicated hardware appliance to a VM to lower-level Linux firewalling.

## Decision

Run **OPNsense as a VM** on teejhost2, using dedicated NICs (WAN on `enp2s0f1` → vmbr1, LAN
trunk on `enp2s0f0` → vmbr0).

## Consequences

- Config is versionable and reproducible (the OPNsense `config.xml` can be backed up, edited,
  and restored), which fits the IaC/disaster-recovery goals.
- Exercises the full routing/firewall stack you'd meet in real work, without dedicated hardware.
- Couples the router's availability to teejhost2 — if that node is down, routing is down.
  Mitigated by console access always being available and PBS backups.
- Management changes to teejhost2 must be done carefully, since fumbling them can take down the
  router itself (see [ADR-0005](0005-single-nic-vlan-dual-homing.md) and the migration runbooks).

## Alternatives considered

- **pfSense/OPNsense on dedicated hardware**: not reproducible-as-code, and spends hardware on a
  job a VM does fine.
- **Linux VM with nftables/iptables**: more control, more operational burden, less of a
  recognizable production-style stack.
- **Proxmox's built-in firewall**: limited routing capability for this design.
