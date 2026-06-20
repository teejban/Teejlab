# Runbook: Migrate teejhost2 management onto MGMT VLAN 10

## Summary

teejhost2 is dual-homed onto **MGMT VLAN 10** the same way as the other nodes: keep the flat
IP (`192.168.8.138`) on `vmbr0` and add a tagged VLAN 10 address (`10.0.10.3`) on `vmbr0.10`.

This is the **smallest** of the three migrations but the **highest-stakes**, because teejhost2
hosts the OPNsense VM (VMID 100). `vmbr0` is already VLAN-aware (`bridge-vids 2-4094`) and
switch Port 5 already tags VLAN 10 — so there's **no bridge-mode change and no switch
reconfiguration**, just one added stanza. But if `vmbr0` is broken, you lose the router and
with it DHCP/DNS/routing/internet for the entire lab, not just one node.

Done without a physical console (deliberate choice), so the safety net is a **dead-man's-switch
auto-rollback** plus the fact that the change is purely additive and never modifies the working
management path.

## Why this is low-risk despite the blast radius

- The edit only **appends** a `vmbr0.10` stanza. The existing `vmbr0` (with `192.168.8.138`
  and the gateway), `vmbr1` (WAN), and both OPNsense taps are untouched.
- The host's VLAN-10 traffic to `10.0.10.1` bridges locally within `vmbr0` to the OPNsense LAN
  tap — it doesn't even depend on the switch.
- Validation happens before apply; an auto-rollback heals the box if connectivity drops.

## Target state

- `vmbr0`: unchanged — `192.168.8.138/24`, gateway `192.168.8.1`, VLAN-aware, vids 2–4094.
- New `vmbr0.10`: `10.0.10.3/24`, **no gateway** (default route stays on `vmbr0`).
- `vmbr1` (WAN) and OPNsense VM: untouched. No switch change.

## Procedure

### Step 0 — Back up

```sh
cp /etc/network/interfaces /etc/network/interfaces.bak
```

### Step 1 — Append the VLAN interface (do not rewrite the file)

```sh
cat >> /etc/network/interfaces <<'EOF'

auto vmbr0.10
iface vmbr0.10 inet static
        address 10.0.10.3/24
EOF
```

### Step 2 — Validate before applying

```sh
diff /etc/network/interfaces.bak /etc/network/interfaces   # only the new stanza
ifquery -a                                                 # parses cleanly
ifreload -a --dry-run                                       # plans to add vmbr0.10, nothing else
```

### Step 3 — Arm the dead-man's switch

```sh
# auto-revert in 5 min unless cancelled (survives an SSH drop)
echo 'cp /etc/network/interfaces.bak /etc/network/interfaces && ifreload -a' | at now + 5 minutes
```

(If `at` is missing: `apt install at`.)

### Step 4 — Apply and verify

```sh
ifreload -a
ip -br a               # vmbr0.10 → 10.0.10.3; vmbr0 still 192.168.8.138
ping -c3 10.0.10.1     # OPNsense gateway on VLAN 10
```

### Step 5 — Cancel the rollback (only once verified)

```sh
atq                    # find the job number
atrm <job#>
```

### Step 6 — Confirm nothing else moved

```sh
pvecm status           # cluster quorate
```

Also sanity-check the lab still routes (a client still has internet / reaches services) — it
should, since OPNsense was never touched.

## Rollback

If you're still connected: `cp /etc/network/interfaces.bak /etc/network/interfaces && ifreload -a`.
If you got cut off, the dead-man's switch does it for you within 5 minutes. Worst case (apply
broke `vmbr0` and the rollback also failed): physical power-cycle of teejhost2 — the lab is down
until then. This is the residual risk accepted by skipping the console.

## Lessons learned

- **It really was the smallest change of the three.** `vmbr0` already VLAN-aware +
  Port 5 already tagging VLAN 10 meant just one appended stanza — no bridge-mode change, no
  switch reconfiguration. The ping to `10.0.10.1` worked immediately via the local bridge path.
- **The dead-man's switch was skipped** this run (step 3 not armed). It worked out, but that
  means the apply ran without a safety net on the router host — the additive nature of the
  change (untouched `vmbr0`/management IP) is what carried it. Arm it next time on a
  higher-risk edit; the cost is one command.
- **`pvecm status` was unchanged** — corosync untouched by this step (it's a separate change),
  and OPNsense never noticed.

### What I'd do differently

- Arm the auto-rollback even when "it's just an append." On the box that runs the router, the
  downside of skipping it is the whole lab; the downside of using it is ~nothing.
