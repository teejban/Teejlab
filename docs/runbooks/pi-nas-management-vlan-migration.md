# Runbook: Migrate teejlab-pi-nas management onto MGMT VLAN 10

## Summary

teejlab-pi-nas (Raspberry Pi 5, OpenMediaVault 7) was reached on the flat network at
`192.168.8.230`. This dual-homes it onto **MGMT VLAN 10**: it keeps the flat IP and gains a
tagged VLAN 10 address (`10.0.10.4`) on `eth0.10`, with its switch port (Port 2) turned into
a trunk. The flat IP stays for cluster/QDevice traffic and as a no-lockout fallback.

This is the same dual-homing pattern as the teejhost1 runbook, but the Pi is **lower risk**
(losing it briefly doesn't break quorum — with the QDevice down you still have 2 of 3 votes
while both nodes are up) and the mechanics differ because **OpenMediaVault owns the network
config**. See `teejhost1-management-vlan-migration.md` for the shared concepts (single-NIC
tagging, the corosync deferral, the reachability asymmetry).

## The key difference: OMV manages networking

Unlike Proxmox, you do **not** hand-edit `/etc/network/interfaces` or the `systemd-networkd`
files on OMV — it regenerates them from its own config database on every apply and will
silently overwrite your edits. All network changes go through the **OMV web UI** (or its RPC),
then a deploy step writes and activates them.

## Target state

- **Switch Port 2**: trunk — PVID 1 / untagged VLAN 1 (kept), tagged VLAN 10.
- **Pi**: `eth0` keeps `192.168.8.230` (flat, holds the default gateway); new `eth0.10` =
  `10.0.10.4/24` (static, no gateway).
- **Cluster**: QDevice unchanged on the flat net; quorum intact.

## Pre-flight

- You stay reachable on `192.168.8.230` throughout (additive change), so the lockout risk is
  low. A monitor + keyboard on the Pi is still a sensible fallback.
- Confirm the static IP is free: `10.0.10.4` is below the DHCP pool (`.10–.30`).

## Procedure

### Step 1 — Switch: trunk Port 2

802.1Q VLAN → VLAN 10 → set **Port 2 = Tagged** → **Modify**. PVID → Port 2 stays **1**.
Then **System Tools → Save Config** (RAM-only otherwise).

### Step 2 — OMV: add the VLAN interface

**Network → Interfaces → Create (+) → VLAN**:

- Parent device: `eth0`
- VLAN id: `10`
- IPv4 method: `Static`, Address `10.0.10.4`, Netmask `255.255.255.0`, **Gateway empty**
- IPv6: disabled; DNS: empty

**Save**, then click **Apply** on the pending-changes banner. Equivalent from the shell (must
be root):

```sh
sudo omv-salt deploy run systemd-networkd
```

### Step 3 — Verify

```sh
ip -br a               # eth0.10 UP with 10.0.10.4; eth0 still 192.168.8.230
ping -c3 10.0.10.1     # OPNsense gateway on VLAN 10
```

From a Proxmox node: `pvecm status` — QDevice still listed, cluster quorate.

## Lessons learned

- **Saving in OMV isn't applying.** The interface showed correctly in Network → Interfaces
  (Static, `10.0.10.4`) but `ip a` showed `eth0.10` **down with no IP** — because the change
  was staged, not deployed. The fix is the **Apply** banner (or `omv-salt deploy run
  systemd-networkd`), not anything to do with the interface config itself.
- **A reboot would not have helped.** OMV only writes live network files on deploy; rebooting
  with the change still pending just brings the interface back up unconfigured. Reboot was the
  wrong instinct here.
- **`omv-salt` must run as root.** Running it as a normal user fails deep in a traceback whose
  real cause is the last line: `PermissionError: .../var/cache/salt/...`. `sudo` fixes it.
- **NIC is `eth0`** (not `end0` as Pi 5 sometimes enumerates) — confirmed with `ip -br a`.

### What I'd do differently

- Click **Apply** immediately after Save and watch it finish before testing — would have
  skipped the "interface created but down" confusion entirely.
