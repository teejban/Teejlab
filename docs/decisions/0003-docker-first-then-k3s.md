# 0003. Docker first, then k3s

- **Status**: Accepted
- **Date**: 2026-06-18

## Context

The lab needs to containerize services and eventually orchestrate them. The question is
whether to start with plain Docker/Compose or jump straight to Kubernetes (k3s).

## Decision

Learn containers with **Docker Compose first**, then migrate to **k3s** once the fundamentals
are solid (planned a few months in).

## Consequences

- Lower barrier to the first deployed service; Compose is simpler to reason about and debug
  when something breaks.
- Produces a genuine portfolio narrative — "grew from a single Docker host to a distributed
  k3s cluster" — rather than starting at maximum complexity.
- A migration step is incurred later (Compose → Helm/manifests), which is accepted as the cost
  of the gentler learning curve.

## Alternatives considered

- **k3s from day one**: steeper curve, harder to debug early, and easy to cargo-cult without
  understanding the container basics underneath.
- **Both in parallel**: unnecessary complexity for a solo lab.
