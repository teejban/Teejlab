# Architecture Decision Records

This directory holds **ADRs** — short, dated records of significant decisions: what was
decided, why, what was rejected, and the tradeoffs accepted. The point isn't ceremony; it's
so that six months later (or in an interview) the *reasoning* behind a choice is recoverable,
not just the outcome.

## Format

One file per decision, numbered in order: `NNNN-short-title.md` (e.g.
`0001-cloudflare-dns-over-route53.md`). Each follows this shape:

```markdown
# NNNN. Title

- **Status**: Proposed | Accepted | Superseded by [ADR-XXXX] | Deprecated
- **Date**: YYYY-MM-DD

## Context
What problem or question prompted this? What constraints were in play?

## Decision
What we chose, stated plainly.

## Consequences
What this makes easier, what it makes harder, and the tradeoffs accepted.

## Alternatives considered
What else was on the table and why it lost.
```

## Conventions

- ADRs are **append-only**. Don't rewrite history — if a decision changes, write a new ADR and
  mark the old one `Superseded by [ADR-XXXX]`.
- Number monotonically; don't reuse numbers.
- Keep them short. An ADR is a paragraph or two per section, not an essay.

## Index

- [0001 — Cloudflare DNS over Route 53](0001-cloudflare-dns-over-route53.md)
- [0002 — `.dev` domain over `.io`](0002-dotdev-domain-over-dotio.md)
- [0003 — Docker first, then k3s](0003-docker-first-then-k3s.md)
- [0004 — OPNsense as a VM, not a hardware appliance](0004-opnsense-as-vm.md)
- [0005 — Single-NIC management via VLAN dual-homing](0005-single-nic-vlan-dual-homing.md)
- [0006 — Tailscale for remote and cross-VLAN access](0006-tailscale-for-remote-access.md)
- [0007 — Hybrid model: cloud for the trust-requiring edge, home lab for compute](0007-hybrid-cloud-edge-vs-home-compute.md)
- [0008 — Short hostnames under the domain (drop the redundant `teejlab-` prefix)](0008-short-hostnames-under-domain.md)
