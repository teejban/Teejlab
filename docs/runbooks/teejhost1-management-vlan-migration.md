# Runbook: Migrate teejhost1 management onto MGMT VLAN 10

## Summary

teejhost1 currently lives on the transitional flat network at `192.168.8.119`, reached
through switch Port 3 as an untagged (VLAN 1) access port. The goal is to manage it over
**MGMT VLAN 10 (`10.0.10.0/24`)** instead. The complication is that teejhost1 has a
**single NIC** and is a **Proxmox cluster member**, so two things are entangled with that
one IP: host management (web UI / SSH) and corosync cluster traffic.

This runbook migrates **management access only**. It adds a tagged VLAN 10 interface to
teejhost1 and a trunk to Port 3, leaving the flat IP in place for corosync. Moving
corosync itself to VLAN 10 is a coordinated, all-nodes change and is deliberately **out of
scope here** — see "Why corosync is not touched" below. After this runbook you can reach
teejhost1 on `10.0.10.2`, the cluster is unaffected, and the flat network is still present
(its removal waits on the corosync migration).

Risk level: **medium.** Single NIC means no out-of-band network path — if the NIC config
and switch port disagree, you lose remote access and recover only via the physical console.
Have a monitor and keyboard on teejhost1 before you start.

## Why this node is the right place to start

teejhost1 is a pure compute node — no VMs you can't briefly lose, not the OPNsense host,
not the QDevice. If the procedure goes sideways, nothing critical goes down and you learn
the single-NIC tagging dance on the lowest-stakes box. teejhost2 (OPNsense host) and the Pi
NAS (QDevice) come later, informed by whatever this run teaches.

## The constraint that shapes the design

A single NIC can't have a "spare" port for management, so the management IP has to ride a
**tagged VLAN sub-interface** over the same physical link that carries everything else.
That means Port 3 stops being a plain access port and becomes a **trunk**: untagged VLAN 1
(kept during transition) plus tagged VLAN 10. On the host side, the IP lives on a
`vmbr0.10` VLAN interface rather than directly on the bridge.

### Why corosync is not touched

The cluster was built on the flat network, so each node's corosync `ring0_addr` is its
`192.168.8.x` address (confirm in `/etc/pve/corosync.conf` during pre-flight). Corosync
needs every ring member to reach every other member on the **same low-latency network** —
it is not something you route across VLANs through a firewall. Moving only teejhost1's ring
to `10.0.10.x` while teejhost2 and the QDevice stay on the flat net would split the ring.

So corosync stays on `192.168.8.119`. teejhost1 ends up **dual-homed**: flat IP for cluster
traffic, VLAN 10 IP for management. Because teejhost2 (`.138`) and the QDevice (`.230`) are
on the same flat `/24`, corosync reaches them as a directly-connected route with no gateway
involved — so we are free to point the host's *default* route at VLAN 10 later without
disturbing the ring. The full corosync cutover (and flat-net removal) is a separate
cluster-wide runbook for when all three nodes are ready.

### Management reachability during the transition

You do not lose access to anything by running this. teejhost1 keeps `192.168.8.119`, so it
stays reachable however you reach it today, and `10.0.10.2` is purely additive.

Be aware the flat net sits on OPNsense's **WAN** side, which makes cross-boundary access
asymmetric:

- **VLAN 10 → `192.168.8.x` works** — routed out WAN and NAT'd to OPNsense's WAN IP
  (stateful, so replies return fine). A VLAN 10 admin station can reach teejhost2 (`.138`)
  and the Pi (`.230`) this way.
- **`192.168.8.x` → VLAN 10 does not work** — flat hosts use the travel router as their
  gateway, which has no route into `10.0.10.0/24`, and OPNsense's WAN blocks inbound from
  private ranges. You can't initiate from the flat net into a VLAN.

All of this depends on the current permissive "pass any" inter-VLAN rules. The firewall
**hardening pass must not run until management is fully on the VLANs**, or it will cut the
management path. When it does run, it needs an explicit allow for the admin VLAN to reach
management targets, plus handling for the WAN-side flat-net path.

## Target state

- **Switch Port 3**: trunk — PVID 1 / untagged VLAN 1 (kept during transition), tagged VLAN 10.
- **teejhost1**: `vmbr0` VLAN-aware; new `vmbr0.10` = `10.0.10.2/24` (static, outside the
  DHCP pool `.10–.30`); flat IP `192.168.8.119` retained on `vmbr0` for corosync.
- **Cluster**: unchanged — corosync still on the flat `/24`, quorum intact.

## Pre-flight

**Safety net.** Connect a monitor and keyboard to teejhost1. The M920q has no IPMI, so this
is your only recovery path if the NIC drops. Do not skip this.

**Capture current state** (paste these somewhere you can reference if you need to roll back):

```sh
ip -br a                       # confirm the NIC name and current addresses
ip r                           # current default route (expect via 192.168.8.1)
cat /etc/network/interfaces    # current bridge config
cat /etc/hosts
pvecm status                   # note quorum + that all links are up BEFORE changes
cat /etc/pve/corosync.conf     # confirm ring0_addr for every node = its 192.168.8.x IP
```

Back up the file you are about to edit:

```sh
cp /etc/network/interfaces /etc/network/interfaces.bak
```

Also screenshot the current TP-Link Port 3 / VLAN / PVID config before changing it.

**Confirm VLAN 10 is actually live** so you are not chasing a dead network: from OPNsense,
verify the VLAN 10 interface is up and `10.0.10.1` answers, ideally by confirming a test
client on VLAN 10 pulls a DHCP lease (same check you already passed on VLAN 30).

**Confirm the static IP is free**: `10.0.10.2` is below the DHCP pool start (`.10`), so it
won't collide with leases. A quick `ping 10.0.10.2` from OPNsense should get nothing.

## Procedure

### Step 1 — Switch: turn Port 3 into a trunk

On the TP-Link, add **VLAN 10 tagged** to Port 3 while leaving VLAN 1 as the untagged /
PVID member. This is the three-menu dance (802.1Q VLAN membership, PVID, and per-port), and
remember the switch does **not** auto-save: after Apply, do **System Tools → Save Config**
or a reboot wipes it.

At this point Port 3 carries untagged VLAN 1 (corosync/flat, unchanged) and tagged VLAN 10
(new, not yet used by the host). Nothing has changed for teejhost1 yet.

### Step 2 — Host: add the VLAN 10 management interface

Make `vmbr0` VLAN-aware (if it isn't already) and add a `vmbr0.10` interface with the new
IP. Keep the existing flat IP and gateway on `vmbr0` for now. Edit
`/etc/network/interfaces` so the relevant stanzas look like this (substitute your real NIC
name from the `ip -br a` output — on an M920q it's typically `eno1` or `enp0s31f6`):

```
auto vmbr0
iface vmbr0 inet static
        address 192.168.8.119/24
        gateway 192.168.8.1
        bridge-ports eno1
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 2-4094

auto vmbr0.10
iface vmbr0.10 inet static
        address 10.0.10.2/24
        # No gateway here in this phase — default route stays via vmbr0 (flat).
```

Apply without a reboot:

```sh
ifreload -a
ip -br a            # vmbr0.10 should now show 10.0.10.2
```

### Step 3 — Verify the new path BEFORE removing anything

Both addresses are on directly-connected subnets, so this is a pure L2 check — no routing
needed:

```sh
# from teejhost1:
ping -c3 10.0.10.1          # OPNsense gateway on VLAN 10

# from OPNsense (or any VLAN 10 host):
ping 10.0.10.2             # teejhost1's new management IP
```

Then confirm the web UI answers on the new address: `https://10.0.10.2:8006`.

If either ping fails, the switch tagging and host VLAN config disagree — fix that before
going further. You have lost nothing: the flat IP and the cluster are untouched.

### Step 4 — Confirm the cluster is still healthy

```sh
pvecm status               # quorate, expected votes present, all links up
```

This should look identical to the pre-flight capture, because corosync is still on the flat
net. If quorum changed, stop and investigate — do not proceed.

### Step 5 (optional, can defer) — Move the default route to VLAN 10

Once `10.0.10.2` is solid, you can make the node's outbound traffic egress via the MGMT VLAN
by moving the gateway from `vmbr0` to `vmbr0.10`. Corosync is unaffected because the flat
peers are reached as a connected route, not via the default gateway.

```
# in /etc/network/interfaces: remove `gateway 192.168.8.1` from vmbr0,
# and add `gateway 10.0.10.1` under vmbr0.10
```

```sh
ifreload -a
ip r                       # default now via 10.0.10.1
ping -c3 1.1.1.1           # internet egress still works
pvecm status               # cluster still quorate
```

This step is genuinely optional for "manage it over VLAN 10" — management already works
after Step 3. Defer it if you'd rather make one change at a time.

## Verification checklist

- `10.0.10.2` reachable from OPNsense and from another VLAN 10 host.
- teejhost1 web UI loads at `https://10.0.10.2:8006`.
- `pvecm status` shows the cluster quorate with all links up (unchanged from pre-flight).
- The switch config survived a Save (re-check after a reboot if you want certainty).

## Rollback

If management access is lost, recover from the **physical console** and:

```sh
cp /etc/network/interfaces.bak /etc/network/interfaces
ifreload -a
```

On the switch, revert Port 3 to its previous untagged-only VLAN 1 access config (remove the
tagged VLAN 10 membership) and Save. Because corosync never moved, the cluster needs no
rollback.

## What's still left after this

This completes management *access* on VLAN 10 for teejhost1. The roadmap item
"management interfaces migrated to MGMT VLAN 10" is only fully done once corosync also moves
off the flat net — and that is a coordinated change across teejhost1, teejhost2, and the
QDevice, done together, with its own runbook. Internal DNS (`*.teejlab.dev` host overrides)
waits until those final `10.0.10.x` IPs are settled.

## Lessons learned

- **NIC name was `eno1`.** Confirmed with `ip -br a` before editing — worth doing rather
  than assuming, since M920q onboard NICs can also enumerate as `enp0s31f6`.
- **`ifreload -a` applied cleanly** — no reboot needed. The bridge switched to VLAN-aware
  live, with at most a momentary blip to the PBS VM (which stayed on VLAN 1 as expected).
- **The failure was downstream, and the symptom told us so.** After the host config, the
  ping gave `Destination Host Unreachable` *from `10.0.10.2` itself* — that's ARP failing to
  resolve, i.e. an L2 problem, not an IP one. The host was fine (`bridge vlan show dev eno1
  vid 10` confirmed VLAN 10 on the uplink); the switch port was the holdup.
- **The TP-Link gotcha: Modify, then Save.** Setting Port 3 to tagged VLAN 10 does nothing
  until you hit **Modify** on the 802.1Q VLAN form, and it's RAM-only until **System Tools →
  Save Config**. The whole ping failure was just Port 3 not yet trunking.
- **`pvecm status` was unchanged** after the migration — corosync never noticed, because it's
  still on the flat net (by design). Confirms the dual-homing left the cluster alone.

### What I'd do differently

- **Do the switch trunk *first*, then the host interface.** Going host-first left a confusing
  "interface is up but can't ping" window. Trunk the port first and the host config works on
  the first try.
- **Verify VLAN 10 end-to-end before starting** — the only prior validation was on VLAN 30, so
  VLAN 10 was unproven going in. It worked, but that was luck, not diligence.
