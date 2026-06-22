# Runbook: Internal DNS (split-horizon `teejlab.dev` via OPNsense Unbound)

## Summary

Internal lab names resolve to their VLAN 10 IPs via **Unbound on OPNsense** (the lab's
resolver). `teejlab.dev` is a real public domain on Cloudflare, but internally Unbound answers
the lab's own records with private IPs — classic **split-horizon DNS**: same name, different
answer depending on where you ask. Every lab host points at OPNsense (`10.0.10.1`) for DNS, so
they all resolve each other by name.

**Important — the firewall's record is an *alias*, not its hostname.** OPNsense's OS hostname is
`opnsense` (not `edge`). A firewall is multi-homed, and OPNsense auto-registers its *own
hostname* against **every** interface IP — so if the OS hostname were `edge`, `edge.teejlab.dev`
would return all five VLAN gateways + the WAN IP. Keeping the OS hostname as `opnsense` lets the
`edge` host override stand alone as a clean single `10.0.10.1` for management. (See lessons.)

Records (host overrides) — short single-label names per ADR-0008:

| Name | FQDN | IP | Role |
|------|------|-----|------|
| `edge` | `edge.teejlab.dev` | `10.0.10.1` | OPNsense management alias (MGMT IP) |
| `teejhost1` | `teejhost1.teejlab.dev` | `10.0.10.2` | hypervisor 1 |
| `teejhost2` | `teejhost2.teejlab.dev` | `10.0.10.3` | hypervisor 2 |
| `nas` | `nas.teejlab.dev` | `10.0.10.4` | Pi NAS / QDevice |
| `pbs` | `pbs.teejlab.dev` | `10.0.10.6` | backup server |

## 1. Unbound host overrides (the records)

OPNsense → **Services → Unbound DNS → Overrides → Host Overrides** → add each (Host, Domain
`teejlab.dev`, Type A, the IP), then **Apply**. Host overrides also auto-generate the reverse
(PTR) records.

The CLI-ish equivalent (Unbound → Advanced → Custom options):
```
local-zone: "teejlab.dev." transparent
local-data: "edge.teejlab.dev. IN A 10.0.10.1"
... (one local-data per host)
```

## 2. OPNsense itself must use its own Unbound

OPNsense → **System → Settings → General**:
- Keep upstream **DNS servers** (`1.1.1.1`, `8.8.8.8`) — these are where Unbound forwards
  external queries.
- **DNS search domain**: `teejlab.dev`
- **"Do not use the local DNS service as a nameserver for this system"**: **unchecked** — this
  is what makes OPNsense resolve via its own Unbound (`127.0.0.1`), so the firewall sees the
  internal records.
- **"Allow DNS server list to be overridden by DHCP/PPP on WAN"**: **unchecked** — keeps the
  upstream deterministic (your `1.1.1.1`/`8.8.8.8`), so the landlord's travel-router DNS isn't
  silently injected.

## 3. Point every lab host's resolver at OPNsense

The records do nothing unless each box actually asks Unbound.

- **Proxmox nodes + PBS** (Debian, static IP — file isn't auto-managed):
  ```sh
  printf 'search teejlab.dev\nnameserver 10.0.10.1\n' > /etc/resolv.conf
  ```
  (Or PVE GUI: node → System → DNS.)
- **NAS** (OMV): set it in the **UI** (System → Network → interface/General → DNS servers
  `10.0.10.1`, search domain `teejlab.dev`) — **not** `/etc/resolv.conf`, which OMV regenerates.

## 4. Verify

From **OPNsense** (FreeBSD — `drill`, not `nslookup`):
```sh
drill pbs.teejlab.dev @127.0.0.1     # 10.0.10.6
```

From any **Linux** node (teejhost1/2, pbs, nas) — paste this block to check everything at once:
```sh
echo "=== DNS test on $(hostname) ==="
echo "resolver(s): $(awk '/^nameserver/{print $2}' /etc/resolv.conf | tr '\n' ' ')"
fail=0
for pair in edge:10.0.10.1 teejhost1:10.0.10.2 teejhost2:10.0.10.3 nas:10.0.10.4 pbs:10.0.10.6; do
  n=${pair%%:*}; want=${pair##*:}
  got=$(getent hosts "$n.teejlab.dev" | awk '{print $1}')
  if [ "$got" = "$want" ]; then echo "  OK    $n.teejlab.dev -> $got"
  else echo "  FAIL  $n.teejlab.dev -> ${got:-none} (want $want)"; fail=1; fi
done
getent hosts pbs >/dev/null 2>&1 && echo "  OK    bare 'pbs' (search domain)" || { echo "  FAIL  bare 'pbs'"; fail=1; }
getent hosts github.com >/dev/null 2>&1 && echo "  OK    external (github.com)" || { echo "  FAIL  external"; fail=1; }
[ "$fail" = 0 ] && echo "=== ALL GOOD ===" || echo "=== SOME CHECKS FAILED ==="
```

## Still optional: resolve these names from the laptop over Tailscale

Tailscale clients use their own DNS, so add **split DNS** in the Tailscale admin console →
DNS → Nameservers → Custom: nameserver `10.0.10.1`, **restricted to `teejlab.dev`**. Then
`*.teejlab.dev` resolves from anywhere on the tailnet; everything else uses normal DNS.

## Lessons learned

- **`nslookup` isn't installed by default** on OPNsense (FreeBSD) or stock Debian. Use `drill`
  / `host` on OPNsense, `getent hosts` (no install) or `dig` (after `apt install dnsutils`) on
  Linux.
- **Records ≠ resolution.** Adding host overrides isn't enough — each client must *point at*
  `10.0.10.1`, or it never asks Unbound. The "oh duh" gap.
- **OPNsense resolving internally is a checkbox**, not a DNS-server entry: leave "Do not use the
  local DNS service…" unchecked so the firewall queries its own Unbound (don't try to add
  `127.0.0.1` to the DNS-servers list).
- **Split-horizon is the point**: `teejlab.dev` is public on Cloudflare, but the lab answers its
  own names with private IPs locally — no private IPs leak to the public zone.
- **Don't name the firewall the same as your clean management record.** A multi-homed firewall
  auto-registers its *own hostname* against every interface IP, so naming the OS hostname `edge`
  made `edge.teejlab.dev` return all five VLAN gateways + the WAN IP (the test caught it by
  resolving to `10.0.50.1` first). Fix: keep the OS hostname distinct (`opnsense`) and use a
  separate host override (`edge → 10.0.10.1`) as the management alias. OS hostname, VM display
  name, and DNS alias are three independent things — they don't have to match.
