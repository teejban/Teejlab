# Runbook: Migrate corosync onto MGMT VLAN 10

## Summary

The 2-node cluster's corosync ring was built on the flat network (`ring0_addr` =
`192.168.8.x` per node, QDevice at `192.168.8.230`). The goal is to move cluster
communication onto **MGMT VLAN 10** so the flat network can eventually be retired.

This was done in two stages. **Stage 1**: add VLAN 10 as a *second* corosync link (`ring1`)
alongside the flat `ring0` — additive and redundant, so it can't split the cluster.
**Stage 2**: cut over to a single VLAN 10 link, remove the flat `ring0`, and move the QDevice
to `10.0.10.4`. Both stages are complete; the cluster now has no flat-net dependency.

Prerequisite: all nodes dual-homed on VLAN 10 and able to reach each other on it (see the
per-node management-VLAN runbooks).

## Why add a link instead of swapping the address

`ring0` (flat) stays untouched, so if the new `ring1` address is wrong it simply shows "down"
and the cluster keeps running on `ring0` — no quorum loss, no split-brain. Corosync (knet)
supports up to 8 links; `link_mode: passive` (already set) uses the highest-priority available
one. This makes Stage 1 about as safe as a cluster-comms change gets.

## The danger to respect

`/etc/pve/corosync.conf` lives on **pmxcfs**, which is only writable while the cluster is
quorate. A change that breaks the ring drops quorum and makes `/etc/pve` **read-only** — you
then can't fix it via `/etc/pve`. Recovery is the *local* copy (`/etc/corosync/corosync.conf`),
which is a plain file, plus a corosync restart. Back both up on every node before starting.

## Stage 1 procedure (additive `ring1`) — as performed

### 1. Verify the VLAN 10 mesh between nodes

```sh
# from teejhost1: ping -c3 10.0.10.3
# from teejhost2: ping -c3 10.0.10.2
```

Both must succeed before proceeding.

### 2. Back up on BOTH nodes (cluster copy + local recovery copy)

```sh
cp /etc/pve/corosync.conf /root/corosync.conf.bak
cp /etc/corosync/corosync.conf /root/corosync.conf.local.bak
```

### 3. Edit `/etc/pve/corosync.conf` on ONE node (pmxcfs propagates it)

Three changes:

- add `ring1_addr` (the VLAN 10 IP) to each node in `nodelist`
- add a second `interface { linknumber: 1 }` block in `totem`
- **increment `config_version`** (3 → 4) — corosync ignores the change otherwise

Result:

```
nodelist {
  node { name: teejhost1; nodeid: 1; quorum_votes: 1; ring0_addr: 192.168.8.119; ring1_addr: 10.0.10.2 }
  node { name: teejhost2; nodeid: 2; quorum_votes: 1; ring0_addr: 192.168.8.138; ring1_addr: 10.0.10.3 }
}
...
totem {
  cluster_name: teejlab-cluster
  config_version: 4
  interface { linknumber: 0 }
  interface { linknumber: 1 }
  ...
}
```

The QDevice block (`host: 192.168.8.230`) is left unchanged — it's a qnetd TCP connection, not
a ring, and migrates in Stage 2.

Apply via a staged file + diff to avoid a mid-write read:

```sh
# stage outside pmxcfs, diff, then copy in
diff /etc/pve/corosync.conf /root/corosync.conf.new
cp /root/corosync.conf.new /etc/pve/corosync.conf
```

### 4. Verify both links are up (on each node)

```sh
corosync-cfgtool -s     # LINK ID 0 and LINK ID 1 both present and connected
pvecm status
journalctl -u corosync -n 20 --no-pager
```

The check that matters: **LINK ID 1 shows `connected`** to the other node — a configured but
down link still leaves `pvecm status` quorate (via `ring0`), so it would hide a non-working
VLAN 10 ring.

## Recovery

```sh
# on each node, if pmxcfs went read-only:
cp /root/corosync.conf.local.bak /etc/corosync/corosync.conf
systemctl restart corosync
```

## Stage 2 (done) — cut over to a single VLAN 10 link

What was actually done:

1. Rewrote `corosync.conf` so `ring0_addr` = the VLAN 10 IPs (`10.0.10.2`/`10.0.10.3`),
   removed both `ring1_addr` lines and the `linknumber: 1` interface, bumped `config_version`
   4 → 5. (Collapsed to a single link, VLAN 10 as `ring0`.)
2. `corosync-cfgtool -R` **failed** with `CS_ERR_INVALID_PARAM` — changing a link's bind
   address / removing a link is not a live-reloadable operation.
3. Restarted corosync on **both nodes back-to-back** (`ssh teejhost1 systemctl restart
   corosync && systemctl restart corosync`). A node on the new config and one on the old can't
   talk (mismatched link addresses), so they only reconverge once both restart. Brief quorum
   blip, VMs unaffected.
4. Moved the QDevice: `pvecm qdevice remove` then `pvecm qdevice setup 10.0.10.4 -f` (run on a
   node — the Pi has no `pvecm`; OMV uses an admin user, not root, for the SSH step).

After this the cluster runs entirely on VLAN 10 with the QDevice at `10.0.10.4` — no flat-net
dependency. The flat IPs were then removed from the hosts (see the per-host runbooks), and the
CIFS storage repointed to `10.0.10.4`. Remaining before VLAN 1 can be fully retired: migrate
the PBS VM (`192.168.8.233`) and scrub VLAN 1 off the switch.

## Lessons learned

- **Additive `ring1` is the right call.** Both links came up `connected` on the first apply,
  with zero disruption — `pvecm status` never flinched because `ring0` carried everything
  throughout.
- **`corosync-cfgtool -s` is the real verification**, not `pvecm status`. Confirming LINK ID 1
  shows `connected` (`10.0.10.3` → nodeid 1 connected) is what proves VLAN 10 is actually
  passing corosync traffic.
- **`config_version` bump is mandatory.** Easy to forget; without it the edit is silently
  ignored.
