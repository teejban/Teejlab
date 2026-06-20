# 0001. Cloudflare DNS over Route 53

- **Status**: Accepted
- **Date**: 2026-06-20

## Context

The lab needs a public DNS zone for the owned domain — to drive internal TLS via Let's
Encrypt **DNS-01** and to publish inbound services. Two obvious hosts: Cloudflare and AWS
Route 53. There was a temptation to register/host DNS at Route 53 "because we'll use AWS
anyway."

## Decision

Host the DNS zone on **Cloudflare** (free tier), and treat the registrar choice as
independent of where any cloud workloads run.

## Consequences

- The inbound plan is **Cloudflare Tunnel**, which *requires* the zone to live on Cloudflare —
  this satisfies it directly.
- The same Cloudflare account provides the API token for ACME DNS-01, so internal TLS works
  with no extra moving parts.
- Avoids Route 53's per-hosted-zone monthly charge.
- If real public AWS endpoints ever appear, we delegate a subdomain (e.g.
  `aws.teejlab.dev`) to a Route 53 hosted zone rather than moving the whole zone.

## Alternatives considered

- **Route 53**: would add a hosted-zone cost, and — more importantly — putting the zone on AWS
  is incompatible with Cloudflare Tunnel, which is core to the inbound design. The "use AWS
  for everything" instinct conflated registrar/DNS-host with workload location; they're
  separable, and DNS host is the constraint that mattered.
- **ACM for certs**: not usable for homelab services — ACM certificates can't be exported off
  AWS services — so Let's Encrypt remains the TLS path regardless of any AWS usage.
