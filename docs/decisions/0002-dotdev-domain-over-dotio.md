# 0002. `.dev` domain over `.io`

- **Status**: Accepted
- **Date**: 2026-06-20

## Context

Early docs claimed `teejlab.io` was owned, but a WHOIS check (authoritative `whois.nic.io`)
returned **"Domain not found"** — it was never registered. So a domain had to be chosen and
bought before wiring it into TLS, DNS, and service hostnames. `.io` is relatively expensive
(~$35–60/yr) and carries a geopolitical question mark (British Indian Ocean Territory /
Chagos sovereignty transfer).

## Decision

Register **`teejlab.dev`** (2-year term) instead of `.io`.

## Consequences

- Roughly a third of `.io`'s cost, and a stable Google-run gTLD with no ccTLD uncertainty.
- `.dev` is on the **HSTS preload list**, so browsers force HTTPS on these hostnames by
  default — which aligns with the all-TLS internal plan and is a small but real security signal.
- All references migrated from `teejlab.io` to `teejlab.dev` across the repo and OPNsense
  (system domain + five per-VLAN DHCP domain names).

## Alternatives considered

- **`.io`**: more expensive, and the sovereignty transition is a non-zero long-term risk for a
  domain meant to anchor a portfolio identity.
- **`.net`**: safe and cheap, but reads less distinctively for a DevOps-focused lab.
- **`.com` / `.org`**: both already registered by a third party (since 2016).
