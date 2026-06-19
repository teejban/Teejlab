# Decisions Log

Rationale for major architectural and operational choices. Helps future-you remember *why* something was chosen, not just *what*.

---

## Decision: GitHub for Source Control (not self-hosted Gitea)

**Date**: 2026-06-18  
**Status**: ✅ Accepted  
**Participants**: TJ

### Problem
Need a source control strategy for IaC, CI/CD pipelines, and container configurations.

### Options Considered
1. **GitHub (Public)** — Leverage portfolio visibility, free tiers, native Actions
2. **Self-hosted Gitea** — Full control, learns ops skill, but adds operational burden
3. **GitLab** — Feature-rich, but more complex than GitHub for this scope

### Decision
**GitHub (Public)** — Use GitHub for code repository. Make infrastructure code public to showcase skills. Use private GitHub Secrets for credentials (API tokens, registry auth).

### Reasoning
- **Portfolio visibility**: Recruiters can see actual code and commit history
- **CI/CD out of the box**: GitHub Actions free, no separate runner infrastructure needed
- **Lower ops burden**: Don't have to maintain Gitea backup/updates
- **Ecosystem**: Better integration with OSS tools (Dependabot, CodeQL, etc.)
- **Risk accepted**: Some infra details exposed (IP scheme, hostnames). Mitigated by: using generic names, no secrets in code, DMZ for public services only

### Trade-offs
- ✅ Easier CI/CD setup
- ❌ Less "production-like" (Gitea more realistic MSP scenario)
- ❌ Infra details visible to internet

### Related Decisions
- Use GitHub Secrets for all API keys/tokens
- Use branch protection rules (require PR reviews, status checks)
- Document security posture in README (what's public, what's hidden)

---

## Decision: Docker First, Then k3s

**Date**: 2026-06-18  
**Status**: ✅ Accepted  
**Participants**: TJ

### Problem
Need to containerize services and eventually orchestrate them. Docker or Kubernetes first?

### Options Considered
1. **Docker Compose (simple)** → then upgrade to k3s
2. **k3s from the start** — Kubernetes from day one, steeper learning curve
3. **Both in parallel** — Overkill for this scope

### Decision
**Phase 3 = Docker/Compose, Phase 5 = k3s** — Learn fundamentals with Compose, then graduate to Kubernetes orchestration.

### Reasoning
- **Learning progression**: Containers (Phase 3) → orchestration (Phase 5)
- **Risk mitigation**: Docker Compose is simpler to debug if something breaks
- **Faster early wins**: Deploy first service faster with Compose
- **k3s justification**: Easier to justify k3s in portfolio when showing "grew from single host to distributed system"

### Trade-offs
- ✅ Lower barrier to entry (Docker before k3s)
- ✅ Faster Phase 3 completion
- ❌ Will need to migrate services (one-way work)
- ❌ Different deployment patterns between phases

### Lessons Learned
None yet (planned).

---

## Decision: OPNsense as VM Router (vs. hardware appliance)

**Date**: 2026-06-18  
**Status**: ✅ Accepted  
**Participants**: TJ

### Problem
Need a firewall/router for VLAN routing and inter-VLAN policies.

### Options Considered
1. **OPNsense VM** — Runs on Proxmox, uses NIC passthrough, fully reproducible in IaC
2. **pfSense hardware appliance** — Dedicated hardware, but not learnable/IaC-able
3. **Linux VM (nftables/iptables)** — Lower level control, steeper ops burden
4. **Proxmox firewall** — Limited routing capabilities

### Decision
**OPNsense VM on Node 1** — Deploy OPNsense in a VM with 2 NICs (passthrough), route VLAN traffic through it.

### Reasoning
- **IaC-friendly**: Can version control OPNsense config (backup.xml)
- **Learnable**: Full routing/firewall stack you'll encounter in prod MSP work
- **Disaster recovery**: Reproducible from Terraform + Ansible (disaster recovery demo)
- **Resource efficiency**: Doesn't need dedicated hardware, runs on existing cluster

### Trade-offs
- ✅ Everything in code (portable, versionable)
- ✅ Demonstrates "software-defined networking" skill
- ❌ VM overhead (negligible for homelab)
- ❌ Requires NIC passthrough (more complex Terraform)
- ❌ If Node 1 dies, router down (but PBS backup exists)

### Risk Mitigations
- Console access always available (even if networking broken)
- Backup OPNsense config to PBS regularly
- Document rollback process

---

## Decision: Internal Domain (.internal vs. home.arpa)

**Date**: 2026-06-18  
**Status**: ⏳ Pending Decision  
**Participants**: TJ

### Problem
Services need an internal DNS namespace. Which TLD?

### Options Considered
1. **.internal** — Colloquial, works most places, not RFC-official
2. **.home.arpa** — RFC 8375 special-use domain, "correct" choice
3. **.local** — Deprecated (mDNS conflict), avoid
4. **.lab** — Common, but not RFC-safe

### Decision
**TBD** — Choose before Phase 1 completes (need DNS serving this domain).

### Reasoning Needed
- .internal vs. .home.arpa trade-off clarity
- Compatibility with tools (e.g., does Kubernetes DNS prefer one?)
- Corporate env familiarity (which will I encounter in real jobs?)

### Note to Future Self
- Document chosen decision in network-plan.md
- Update all DNS records once chosen

---

## Decision: VLAN Numbering Scheme

**Date**: 2026-06-18  
**Status**: ✅ Accepted  
**Participants**: TJ

### Problem
Need to segment network into security zones and operational boundaries.

### Options Considered
1. **Dense numbering** (1–256 per tenant) — Complex, overkill for homelab
2. **Sparse, meaningful numbering** (10, 20, 30, etc.) — Clear intent, room to grow
3. **RFC 1918 + VxLAN** — Overthinking for flat 2-node cluster

### Decision
**Use /24 subnets on 10.0.x.0/16**: 10 (mgmt), 20 (servers), 30 (lab), 40 (trusted), 50 (dmz), 99 (iot)

### Reasoning
- **Meaningful**: Each VLAN ID telegraphs its purpose (10 = tier 1 management)
- **Room to grow**: Can add 11, 12, etc. if needed later
- **Standard**: Matches common enterprise practices
- **Simple subnet math**: 10.0.10.0/24 is easier than 192.168.x.y

### Trade-offs
- ✅ Clear, easy to remember
- ✅ Follows conventions (recruiters will recognize pattern)
- ❌ Slightly "wasteful" (not using 1–9, 21–29)

### Related Decisions
- Management (VLAN 10) is read-only to servers (VLAN 20)
- Lab (VLAN 30) can reach servers (test→prod gateway)
- IoT (VLAN 99) is completely isolated

---

## Decision: Phase-Based Roadmap vs. Continuous Delivery

**Date**: 2026-06-18  
**Status**: ✅ Accepted  
**Participants**: TJ

### Problem
Should roadmap be strict phases, or more fluid?

### Options Considered
1. **Strict phases** — Each phase fully complete before next starts
2. **Continuous/fluid** — Work on phases in parallel, cherry-pick as you go
3. **Agile/sprints** — 2-week sprints, reprioritize weekly

### Decision
**Strict phase-based**, but with flexibility to hotfix.

### Reasoning
- **Learning progression**: Each phase builds on previous (network → IaC → containers)
- **Portfolio narrative**: Clear story ("built network, then automated, then containerized")
- **Risk management**: Don't overwhelm with simultaneous complexity
- **Measurable progress**: "Phase N complete" is a clear milestone

### Exception Rule
If a blocker appears (e.g., "need CI/CD to iterate faster"), revisit and reprioritize.

### Trade-offs
- ✅ Clear roadmap, easy to communicate progress
- ✅ Each phase has clear deliverables
- ❌ Less agile (can't pivot quickly based on learnings)
- ❌ Possible "sunk time" in a phase that should've been shorter

---

## Decision: Who Should Know These Details?

**Date**: 2026-06-18  
**Status**: ✅ Accepted  
**Participants**: TJ

### Problem
Portfolio should be impressive but not over-engineered. How much detail to expose?

### Options Considered
1. **Full transparency** — Document every decision, every gotcha, every trade-off
2. **Marketing-first** — Only show polished, successful paths
3. **Balanced** — Show decisions + trade-offs, honesty builds credibility

### Decision
**Balanced** — Decisions log is for your learning; README is for recruiters. Be honest about trade-offs, but lead with wins.

### Reasoning
- **Interview readiness**: Recruiter asks "Why VLAN 10 for management?" → You have a credible answer
- **Credibility**: Admitting trade-offs and gotchas shows maturity
- **Reality**: No homelab is perfect; real MSP work involves constraints
- **Learning**: Documenting decisions prevents re-learning the same lessons

### Note
- README: Focus on "what works" + quick architecture diagram
- decisions-log.md: Show reasoning, trade-offs, risks mitigated
- Code comments: Link to decisions-log.md where relevant (e.g., "See decisions-log.md: OPNsense as VM")

---

## Decision Template (for future decisions)

```
## Decision: [Title]

**Date**: YYYY-MM-DD  
**Status**: ✅ Accepted | ⏳ Pending | ❌ Rejected  
**Participants**: [Who was involved]

### Problem
[What's the question?]

### Options Considered
1. **Option A** — Description, key trade-off
2. **Option B** — Description, key trade-off
3. **Option C** — Description, key trade-off

### Decision
[What did you choose and why?]

### Reasoning
- Bullet-point rationale

### Trade-offs
- ✅ Pro 1
- ✅ Pro 2
- ❌ Con 1
- ❌ Con 2

### Related Decisions
- Other decisions that build on this one

### Lessons Learned
[To be filled in after living with the decision for a while]
```

---

## Summary

| Decision | Choice | Conviction | Risk |
|----------|--------|------------|------|
| Source Control | GitHub Public | High | Low (infra details visible, but public code is portfolio asset) |
| Containers | Docker → k3s | High | Medium (migration work in Phase 5) |
| Router | OPNsense VM | High | Low (console fallback, PBS backup) |
| Internal Domain | TBD | — | Low (easy to change before DNS goes live) |
| VLAN Scheme | 10, 20, 30, 40, 50, 99 | High | Low (standard practice) |
| Roadmap | Phase-based | High | Low (flexible exception rule exists) |

---

**Last Updated**: 2026-06-18  
**Next Review**: After Phase 1 completion
