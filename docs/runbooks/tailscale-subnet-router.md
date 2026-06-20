# Runbook: Tailscale subnet router on OPNsense

## Summary

OPNsense runs the Tailscale client as a **subnet router**, advertising the MGMT VLAN
(`10.0.10.0/24`) into a private tailnet. Any enrolled device then reaches `10.0.10.x` — web
UIs and SSH alike — from anywhere, with no port forwarding through the landlord's network.

This is what decouples *managing the lab* from the flat network: previously a flat-net
workstation was the only way in (flat → VLAN can't initiate, since the flat net sits on
OPNsense's WAN side). Tailscale removes that dependency and is therefore a prerequisite for
deprecating VLAN 1.

## Why Tailscale (vs a traditional VPN)

Shared housing means no public IP and no inbound port forwarding — a conventional
hub-and-spoke VPN *into* the lab is a non-starter. Tailscale (WireGuard under the hood) is
outbound-only: devices reach a coordination server, then connect peer-to-peer through NAT.
Same constraint-fit as Cloudflare Tunnel, but for access rather than inbound publishing.

## Prerequisites

- A Tailscale account (free tier), using a **personal** identity — not the work/Zscaler one.
- OPNsense able to reach the internet (it's the router, so yes).
- The MGMT VLAN live with OPNsense holding `10.0.10.1`.

## Procedure

### 1. Install the plugin

**System → Firmware → Plugins** → search **`os-tailscale`** → install. A **VPN → Tailscale**
menu appears afterward.

> Gotcha: `os-tailscale` is a **community** plugin. If it doesn't show in the list, enable
> **"Show Community Plugins"** in the firmware settings (this was the holdup here). If the
> catalog still looks stale, **System → Firmware → Status → Check for updates** refreshes it.

### 2. Authenticate

Generate a key in the Tailscale admin console (**Settings → Keys → Generate auth key**), then
**VPN → Tailscale → Settings** → Enable → paste it into **Auth key** → Save/Apply. OPNsense
registers as a machine in the tailnet. (CLI alternative: `tailscale up` from the shell prints
a login URL.)

### 3. Advertise the subnet

**VPN → Tailscale → Settings → Advertised routes** = `10.0.10.0/24` (add more lab subnets
later as needed). Apply.

### 4. Approve the route (mandatory — it's inactive until you do)

Tailscale admin console → **Machines** → OPNsense → **Edit route settings** → enable
`10.0.10.0/24`. Also **disable key expiry** on this node so the subnet router doesn't drop off
the tailnet in ~6 months.

### 5. Enroll clients

Install Tailscale on the laptop/phone, log in with the **same account**. Windows/Mac use the
approved routes automatically; on Linux, `sudo tailscale up --accept-routes`.

### 6. Verify (web *and* shell, from off-network)

```sh
ping 10.0.10.1
ssh root@10.0.10.3            # teejhost2 shell over Tailscale
# browser: https://10.0.10.3:8006
```

SSH and web both ride the same subnet route — normal credentials, just no longer dependent on
being on the flat network.

## If connected but can't reach `10.0.10.x`

Almost always the **firewall rule**. The plugin usually adds it, but if not: OPNsense creates a
**Tailscale** interface — add a **pass** rule on it (**Firewall → Rules → Tailscale**) allowing
traffic to the lab subnets.

## Notes

- **Normal SSH over the subnet route** is what's in use — works for every `10.0.10.x` host,
  including ones not running Tailscale (the Proxmox nodes, the Pi). Tailscale's own keyless
  **Tailscale SSH** feature only applies to hosts running the client directly (e.g. OPNsense
  itself) and is intentionally not used here.
- Don't enroll the work laptop — keeps the Zscaler corporate device off the lab.

## Lessons learned

- **"Show Community Plugins" was unchecked**, so `os-tailscale` was invisible in the plugin
  list. That toggle (not a missing package) was the whole problem.
- **Had to update OPNsense firmware first.** Since OPNsense is the router *and* a VM, that meant
  a brief lab-wide outage on reboot — recovery path was the VM console via Proxmox
  (teejhost2 → VM 100 → Console). Went cleanly.
- Reaching the lab over Tailscale needs **no port forwarding** — the key payoff given the
  shared-housing constraint.
