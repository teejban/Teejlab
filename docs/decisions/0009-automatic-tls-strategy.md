# 0009. Automatic TLS strategy — DNS-01 everywhere, native ACME for infra, reverse proxy for services

- **Status**: Accepted
- **Date**: 2026-06-23

## Context

`teejlab.dev` is HSTS-preloaded, so browsers force HTTPS and **refuse the self-signed
"proceed anyway" bypass** — valid certs are mandatory for every internal hostname, not
cosmetic. The lab has no inbound reachability (shared housing), so **HTTP-01 can't work** —
but DNS-01 only needs Cloudflare API access to write a TXT record. The goal is that **any box
or service spun up gets TLS automatically**, with minimal per-thing work.

## Decision

**DNS-01 (Cloudflare token) is the challenge method throughout.** TLS is handled in two tiers:

1. **Fixed infrastructure UIs** (Proxmox ×2, PBS, OPNsense) → each platform's **native ACME
   client** issues a per-host cert and auto-renews. Done. (Proxmox/PBS built-in ACME; OPNsense
   `os-acme-client`.)
2. **Services spun up going forward** → a **reverse proxy holding a `*.teejlab.dev` wildcard**
   (also DNS-01). The proxy terminates TLS; backends speak plain HTTP behind it. A new service
   = one route (or, with container auto-discovery, a label) — **no per-service cert issuance**,
   because the wildcard already covers it. Caddy is the simple option now; **Traefik** is the
   target in the Docker/k8s phase (label-based auto-discovery = "spin it up, it's TLS'd", and
   it's k3s's default ingress).

Combine with a wildcard **internal DNS** record (`*.teejlab.dev → proxy IP` in Unbound) so any
new name resolves to the proxy with no DNS edit either.

## Consequences

- Every internal name is browser-trusted and auto-renewing; one **Cloudflare token** is the
  shared dependency for all of it.
- The reverse proxy is the **scale mechanism** — adding services becomes route/label work, not
  cert work. Backends don't manage their own certs.
- Two exceptions: the **NAS UI** (OMV has no native ACME) → fronted by the reverse proxy; the
  **switch** (Easy Smart, HTTP-only) → not a TLS candidate at all.

## Alternatives considered

- **Self-signed certs**: HSTS on `.dev` hard-blocks them in browsers (no click-through).
- **HTTP-01**: needs inbound reachability the housing situation can't provide.
- **Per-service manual/native certs**: doesn't scale to "spin up anything and it's TLS'd."
