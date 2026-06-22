# 0007. Hybrid model — cloud for the trust-requiring edge, home lab for compute

- **Status**: Accepted
- **Date**: 2026-06-21

## Context

Shared housing gives the lab no static IP, no control over reverse DNS (PTR), a residential/
shared egress IP that sits on policy blocklists (Spamhaus PBL), and no inbound port forwarding.
That makes a class of workloads effectively impossible to run *from home* — not for lack of
compute, but for lack of a **trusted, static, controllable internet presence**: public email
delivery, anything needing a clean sending reputation or matching PTR, and public inbound
services on arbitrary ports.

A guiding principle is needed for "what runs at home vs. in the cloud," so the decision is made
on the right axis each time rather than ad hoc.

## Decision

Split workloads by **trust/reachability requirement, not by raw capability**:

- **Home lab** — compute, storage, internal services, experimentation. Anything that does not
  need to be publicly trusted or reachable.
- **Cloud (AWS / Cloudflare / etc.)** — the public-facing, trust-requiring **edge**: outbound
  email via a reputable relay (Amazon SES), static IP + PTR for any public inbound, public
  endpoints, DNS. Small, cheap, but it's where the internet's trust machinery lives.

This is already the de-facto pattern: Cloudflare Tunnel fronts public web (so home IP is never
exposed) and Cloudflare hosts DNS. SES for outbound mail is the same move for email.

## Consequences

- Solves deliverability and public-reachability problems the residential connection can't —
  relayed mail uses trusted IPs, public inbound can have a real PTR.
- Keeps cloud spend minimal: the edge is cheap (SES ≈ $0.10/1,000 emails; a tiny relay
  instance is a few $/month) while the expensive compute stays home.
- Adds some cloud surface to manage — accounts, IAM credentials, a small bill to watch.
- Requires discipline: **do not lift-and-shift whole workloads to cloud** "because cloud" —
  that's how homelab bills balloon. Use cloud only for what only cloud can do.

## Alternatives considered

- **All on-prem**: can't deliver public email or accept public inbound from a residential,
  PTR-less, blocklisted connection — a hard physical/policy limit, not a config gap.
- **All cloud**: defeats the purpose (and cost model) of a home lab; wastes the hardware.

## Notes

- First concrete application: **Amazon SES** as the outbound relay for the lab mail server
  (see the mail-server roadmap items). Internal lab-to-lab mail needs no relay; only mail bound
  for the real internet uses SES.
- If real public AWS endpoints appear, delegate a subdomain (e.g. `aws.teejlab.dev`) to Route
  53 rather than moving the zone off Cloudflare — consistent with [ADR-0001](0001-cloudflare-dns-over-route53.md).
- Public **inbound** email (receiving from the internet) remains out of scope unless/until a
  cloud instance with an Elastic IP + PTR fronts it; not needed for alerts or user/app testing.
