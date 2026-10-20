# Roadmap: Homelab DevOps Portfolio

**Goal**: Build a self-hosted "fake MSP" environment to demonstrate DevOps skills (IaC, CI/CD, containers, GitOps, observability).

**Success Criteria**: Automated, reproducible, well-documented infrastructure that looks professional on a portfolio.

**Capstone**: a single portfolio site (`teejlab.dev`) where a recruiter or hiring manager can
**launch** my resume site *and* the full MSP stack, watch each build happen live (pipeline
steps, resources created, logs, timings), then use what they launched and its monitoring
dashboard. Every earlier phase is a building block for it: IaC creates the infrastructure,
config management configures it, containers package it, CI/CD runs the launch, and
observability is what the visitor watches. See [Phase 7](#phase-7-portfolio-launch-platform-capstone).

---

## Phase 1: Network Foundation ⭐ CURRENT

**Goal**: Segment the flat network, deploy OPNsense, enable VLAN routing and internal DNS.

**Why First**: All downstream services depend on stable, isolated networking. Easier to design right now than refactor later.

**Status**: Nearly complete. OPNsense routing + all five VLANs, the full VLAN 10 migration (hosts, corosync, QDevice, PBS — flat net retired from the lab), internal DNS, Tailscale remote access, and TLS on the infra UIs are all done. Remaining: the **firewall hardening pass** and the remaining per-VLAN switch access ports. (The reverse-proxy auto-TLS layer doubles as the bridge into the Docker phase.)

### Deliverables
- [x] VLAN configuration on TP-Link switch (ports tagged/untagged)
- [x] OPNsense VM deployed on teejhost2 with dedicated NICs (WAN `enp2s0f1` → vmbr1, LAN trunk `enp2s0f0` → vmbr0)
- [x] OPNsense: Uplink configured (WAN), internal routing (LAN)
- [x] DHCP (Kea) serving all five active VLANs
- [x] Proxmox `vmbr0` VLAN-aware on teejhost2, per-VM `trunks` set on the OPNsense NIC
- [x] End-to-end VLAN validation (client on Port 6 → 10.0.30.x lease via OPNsense)
- [ ] OPNsense: firewall hardening pass (default-deny inter-VLAN, DMZ → RFC1918 blocked)
- [ ] Remaining VLAN access ports configured on switch
- [x] Management interfaces migrated to MGMT VLAN 10 (flat IPs removed) — teejhost1 (`10.0.10.2`), teejhost2 (`10.0.10.3`), teejlab-pi-nas (`10.0.10.4`). Remaining: the TP-Link switch (firmware-permitting, since Easy Smart management-VLAN support is limited and lockout means a factory reset)
- [x] Corosync migrated to MGMT VLAN 10 — added `ring1`, then cut over to a single VLAN 10 link and removed flat `ring0`; QDevice moved to `10.0.10.4`. CIFS storage repointed to `10.0.10.4`.
- [x] PBS rebuilt VLAN-10-native (`pbs`, `10.0.10.6`), datastore on the Pi via NFS — closed the last flat-net dependency (see `docs/runbooks/pbs-rebuild.md`)
- [x] Flat network retired from the lab — VLAN 1 intentionally kept on the switch as a break-glass recovery net (the Easy Smart switch's own management lives there); not deprecating it by design
- [x] Internal DNS via OPNsense Unbound for `<service>.teejlab.dev` — host overrides for `edge`/`teejhost1`/`teejhost2`/`nas`/`pbs`, every box pointed at `10.0.10.1`, split-horizon working (see `docs/runbooks/internal-dns.md`)
- [x] TLS — real Let's Encrypt certs (DNS-01 / Cloudflare) on the four infra UIs (`teejhost1`/`teejhost2`/`pbs`/`edge`), auto-renewing (see `docs/runbooks/tls-letsencrypt-dns01.md`). Reverse proxy + `*.teejlab.dev` wildcard now live on docker1 (Traefik) for auto-TLS on services; NAS still to be fronted via it; switch is HTTP-only
- [x] Tailscale on OPNsense advertising lab subnets (remote access) — subnet router advertising `10.0.10.0/24` (now also VLANs 20 and 30), reachable web + SSH from off-network, + split-DNS so `*.teejlab.dev` resolves on the laptop
- [x] Network documentation updated (`docs/architecture/network-architecture.md`)

### Technical Steps
1. ~~Plan IP allocations and VLAN membership~~ ✅ (`10.0.<vlan>.0/24`, OPNsense as `.1`)
2. ~~Configure switch VLAN groups + tagging~~ ✅ (Port 5 trunk, Port 6 VLAN 30 access)
3. ~~Deploy OPNsense (ISO boot, install, basic OS config)~~ ✅
4. ~~Configure OPNsense NICs (bridge, IP assignment, VLAN handling)~~ ✅ (incl. per-VM `trunks` fix)
5. Harden OPNsense firewall rules (currently permissive "pass any" during buildout)
6. ~~Deploy internal DNS (OPNsense Unbound, `teejlab.dev`)~~ ✅ (split-horizon host overrides)
7. ~~Test inter-VLAN routing~~ ✅ — still need to verify isolation once default-deny is in place
8. ~~Document gotchas~~ ✅ (VLAN-aware bridge `trunks` writeup) — add MAC reservations if needed

### Risks
- **Lock yourself out**: OPNsense misconfiguration could break connectivity. Mitigation: console access always available, test in lab VLAN first.
- **Switch vlan misconfiguration**: Flat network becomes split. Mitigation: document current port memberships before any changes.

### Definition of Done
- [x] Proxmox nodes reach each other via MGMT VLAN 10 (flat net retired)
- [ ] Lab/test VLAN can reach the internet but is isolated from other VLANs (pending firewall hardening)
- [x] Internal DNS resolves `<service>.teejlab.dev` hostnames
- [x] Documentation is complete (`docs/architecture/network-architecture.md`)

---

## Phase 2: IaC Foundation

**Goal**: Automate infrastructure provisioning via Terraform and configuration management via Ansible.

### Deliverables
- [ ] Terraform provider for Proxmox configured
- [ ] Terraform state management (local or S3-like)
- [ ] Ansible inventory structure (hosts by role, VLAN, environment)
- [ ] First IaC resource: OPNsense VM (via Terraform)
- [ ] First playbook: OPNsense baseline configuration (via Ansible)
- [ ] Ansible `base` role — codify the VM baseline standard (hand-authored spec already written: `docs/runbooks/vm-baseline-setup.md`)
- [ ] CI/CD pre-check: Terraform plan runs on PR
- [ ] Documentation: "How to provision a new VM" guide

### Technical Steps
1. Create Terraform project structure (modules, main, variables, outputs)
2. Test Terraform against Proxmox (deploy a throwaway VM)
3. Refactor OPNsense into Terraform code
4. Create Ansible playbook for OPNsense post-install config
5. Test full workflow: Terraform → VM boots → Ansible configures
6. Set up GitHub Actions to validate Terraform on PR

### Risks
- **Terraform state corruption**: Lose track of real infrastructure. Mitigation: keep backups, start small.
- **Chicken-and-egg**: OPNsense needs IaC, but IaC needs working Proxmox. Mitigation: do Phase 1 first, then migrate OPNsense to Terraform.

### Definition of Done
- OPNsense VM fully managed via Terraform + Ansible
- No manual SSH changes (all via playbooks or drift detected)
- GitHub PR shows "Terraform plan" output
- Onboarding guide written: "Clone repo, terraform apply, services running"

---

## Phase 3: Docker Host & First Services

**Goal**: Deploy containerized services to learn container fundamentals before moving to Kubernetes.

### Deliverables

Hand-built track first (build-then-codify per [ADR-0010](decisions/0010-docker-host-build-then-codify.md)),
then reproduce via IaC:

- [x] `docker1` VM provisioned by hand on VLAN 20 SERVERS (`10.0.20.2`) — precursor to the IaC deploy
- [x] VM baseline standard established + applied to `docker1` — three pillars (Communicate / Report metrics / Administer securely): key-only SSH + sshd hardening, `unattended-upgrades`, `qemu-guest-agent`, `prometheus-node-exporter`, cross-VLAN internal DNS. Spec for the future Ansible `base` role — see `docs/runbooks/vm-baseline-setup.md`
- [x] Docker Engine + first workload on `docker1` (hand-built role — Traefik reverse proxy for `*.teejlab.dev` auto-TLS; `docs/runbooks/docker-host-setup.md`). Done 2026-10-16: Docker 29 from the official repo, Traefik v3 with a production `*.teejlab.dev` wildcard (DNS-01, dedicated scoped Cloudflare token as a Compose secret), `whoami.teejlab.dev` serving over valid HTTPS
- [x] Shared `proxy` Docker network + the nginx test site behind Traefik (`https://site.teejlab.dev`, no published ports). The pattern every service now follows.
- [ ] NAS UI behind Traefik (not a container, so via Traefik's file provider): valid HTTPS for `nas.teejlab.dev`
- [x] Move the docker1 Compose projects (`traefik`, `web`) into the repo under `infra/docker/`, with secrets, `acme.json` and `.env` git-ignored (the ACME email moved to `.env`; `.env.example` committed). Copied by file name, so secrets never left docker1.
- [ ] Run docker1 from a git checkout: clone the repo on docker1, place secrets/`.env` in the git-ignored paths, deploy with `git pull && docker compose up -d`. Changes flow one way: PC → PR → GitHub → docker1 (read-only), the manual version of what CI does later.
- [ ] Docker VM deployed via IaC (codify the hand-built build)
- [ ] First service containerized: e.g., Portainer, reverse proxy, monitoring
- [ ] Docker registry (local or Docker Hub)
- [ ] Basic compose file (docker-compose.yml in git)
- [ ] Service monitoring/logging pipeline started

### Technical Steps
1. Terraform: Deploy Ubuntu VM on VLAN 20 (Servers)
2. Ansible: Install Docker, set up daemon
3. Containerize a simple service (e.g., nginx reverse proxy)
4. Push image to registry
5. Deploy via docker-compose
6. Set up container restart policy + logging

### Risks
- **Storage meltdown**: Containers fill disk. Mitigation: set resource limits early.
- **Network isolation not working**: Containers can reach wrong VLANs. Mitigation: test in Phase 1 first.

### Definition of Done
- Docker host running 3+ services
- Services survive node restart (systemd service or compose restart policy)
- Service logs visible + searchable
- No manual `docker run` — everything in compose

---

## Phase 4: CI/CD via GitHub Actions

**Goal**: Automate builds, tests, and deployments triggered by git events.

### Deliverables
- [ ] GitHub Actions workflow (build, test, push image)
- [ ] Docker image builds on commit to main
- [ ] Images tagged with git SHA and pushed to registry
- [ ] Deployment trigger: image push → service update
- [ ] Secrets management (registry credentials, Proxmox API tokens) via GitHub Secrets

### Technical Steps
1. Create `.github/workflows/` directory
2. Write build workflow (docker build, test, push on main)
3. Deploy workflow (trigger on image push, update docker-compose, restart)
4. Test full flow: commit → image builds → service updates
5. Add status badges to README

### Risks
- **Runaway builds**: GitHub Actions quota. Mitigation: skip builds on documentation-only commits.
- **Secrets exposure**: Accidentally commit API tokens. Mitigation: use GitHub Secrets, scan pre-commit.

### Definition of Done
- Commit to main → new image pushed to registry within 5 min
- Registry push → service updated without manual intervention
- No secrets in git history (verified by `git-secrets` or similar)
- CI/CD status badge in README

---

## Phase 5: Kubernetes (k3s) & GitOps (ArgoCD)

**Goal**: Migrate from Docker Compose to Kubernetes for orchestration, then GitOps for deployment management.

### Deliverables
- [ ] k3s cluster deployed (3-node or HA setup)
- [ ] Helm charts for existing services
- [ ] ArgoCD deployed, syncing git repo → Kubernetes
- [ ] Declarative deployments (no manual `kubectl apply`)
- [ ] Monitoring + alerting for k3s cluster

### Technical Steps
1. Terraform + Ansible: Deploy k3s cluster (lightweight Kubernetes)
2. Install ArgoCD in k3s
3. Convert docker-compose services to Helm charts
4. Create ArgoCD Application manifests pointing to git repo
5. Test sync: commit chart changes → ArgoCD auto-applies
6. Set up cluster autoscaling + resource limits

### Risks
- **Complexity explosion**: K8s is harder than Docker. Mitigation: document decisions, keep things simple first.
- **Downtime during migration**: Services unavailable. Mitigation: blue-green deployment, canary releases.

### Definition of Done
- Services running on k3s, managed by ArgoCD
- Commit to repo → ArgoCD detects drift and auto-syncs
- All YAML in git (no manual imperative changes)
- Cluster survives node failure (tested)

---

## Phase 6: MSP Stack + Observability

**Goal**: Deploy a realistic set of services that look impressive on a portfolio, built to be
**reproducible from code**, since Phase 7 launches it on demand.

### Deliverables
- [ ] Monitoring stack: Prometheus + Grafana
- [ ] Centralized logging: Loki or ELK
- [ ] Alerting: AlertManager → email/Slack/webhook (email via the lab mail server below)
- [ ] Lab mail server (Mailcow on `lab.teejlab.dev`) — internal alerts/notices + user/app testing; **no public inbound** (per [ADR-0007](decisions/0007-hybrid-cloud-edge-vs-home-compute.md))
- [ ] Outbound mail relay via **Amazon SES** (the cloud trust-edge) + SPF/DKIM/DMARC on the subdomain via Cloudflare
- [ ] Backup automation: Proxmox Backup Server + retention policy
- [ ] MSP stack (Gitea, Tactical RMM, ticketing, BookStack, Uptime Kuma) deployable end-to-end from a single pipeline run, with no manual steps (the prerequisite for visitor launches)
- [ ] GitOps documentation + architecture diagram
- [ ] "Incident response runbook" (example: disk full, node down)

### Technical Steps
1. Deploy Prometheus + Grafana (container or Helm)
2. Scrape metrics from Proxmox, Docker, k3s
3. Set up Grafana dashboards (infrastructure, app-level, business metrics)
4. Deploy Loki for log aggregation
5. Configure AlertManager for incident escalation
6. Stand up the mail server (Mailcow) on `lab.teejlab.dev`; relay outbound through Amazon SES; add SPF/DKIM/DMARC; point lab services' SMTP at it
7. Document disaster recovery (restore from PBS)
8. Package the MSP stack so one pipeline run brings it up from nothing

### Risks
- **Over-engineering**: Build too much, run out of time. Mitigation: prioritize (monitoring > logging > mail).

### Definition of Done
- Grafana dashboards reflect real infrastructure health
- Alert fires (e.g., high CPU) and fires Slack notification
- Logs are centralized and searchable
- MSP stack comes up from a clean slate with one pipeline run
- All deployment documented in architecture diagrams + README

---

## Phase 7: Portfolio Launch Platform (capstone)

**Goal**: One public site where a visitor launches the **resume site** or the **MSP stack**,
watches it build in real time, then gets into what they launched and its dashboard.

**Why**: It shows the skills working instead of listing them. It's also the design that
everything earlier feeds into: one pipeline pattern, pointed at two targets.

### Design

One launch pipeline (GitHub Actions → Terraform → Ansible/containers → health checks), one
observability stack, two products:

| | Resume site | MSP stack |
|---|---|---|
| Launched by | Visitor | Visitor |
| Runs in | AWS (Azure added later for multi-cloud) | **Open decision**: leaning AWS for visitor copies, with the home lab running the permanent stack + monitoring backend |
| Visitor gets | Live build view → the site URL | Live build view → service logins + Grafana dashboard |
| Lifetime | Auto-teardown (≈30–60 min) | Auto-teardown (≈30–60 min) |

**Open decision: where do visitor-launched MSP stacks run?**
- **AWS (leaning)**: keeps anonymous launches off the home network entirely (consistent with
  [ADR-0007](decisions/0007-hybrid-cloud-edge-vs-home-compute.md): public, trust-requiring
  things live in the cloud). Shows one pipeline deploying to cloud *and* on-prem. Costs a few
  cents per launch with small, short-lived instances.
- **Home lab (DMZ VLAN 50 via Cloudflare Tunnel)**: free, and shows the real hardware, but puts
  strangers' launches on hardware on the home network, and is limited by the shared-housing uplink.

Record the outcome as an ADR before building.

### Build order: resume first

The resume is built first and polished, end to end. The MSP stack then reuses the same pipeline,
live view and guardrails, so it's mostly repetition with more services. Each resume milestone is
something working and demonstrable on its own:

- [ ] **R1a: Deploy by hand, on Proxmox.** Terraform (`bpg/proxmox`) clones an Ubuntu cloud-image template on VLAN 20; cloud-init applies the VM baseline, installs Docker and runs the resume container; `apply` brings it up, `destroy` removes it cleanly. It's the free, fast practice target, and it also covers Phase 2 (first Terraform) and Phase 3 (Docker).
- [ ] **R1b: Same, on AWS.** Swap the provider: the same container on EC2 via Terraform. Same `apply`/`destroy` loop.
- [ ] **R2: Deploy from CI.** A GitHub Actions workflow runs the same Terraform (`workflow_dispatch`), authenticating to AWS via **OIDC** (no long-lived keys in GitHub)
- [ ] **R3: Launch button.** The site's button calls a small API that triggers the workflow and returns a launch ID
- [ ] **R4: Live view.** The page shows that launch's steps, logs, resources created and timings as they happen
- [ ] **R5: Guardrails.** TTL auto-teardown + orphan sweeper, concurrency cap, Turnstile, AWS Budgets ceiling, all tested by trying to abuse it
- [ ] **R6: Polish.** Design, the "how it works" page, architecture diagram, write-up. Then point the same pipeline at Azure.

### Deliverables
- [ ] Portfolio site on `teejlab.dev`: resume/about, Launch buttons, live launch view, "how it works" (architecture + runbooks + this repo)
- [ ] Launch API: Launch button → pipeline run (GitHub Actions `workflow_dispatch` or equivalent), returns a launch ID
- [ ] Live launch view: streams pipeline steps, resources created, logs and timings for that launch ID
- [ ] Resume target: Terraform + deploy to AWS, with teardown
- [ ] MSP stack target: same pipeline, deployed per the open decision above, with teardown
- [ ] Per-launch dashboard: Grafana view scoped to the visitor's launch
- [ ] Azure as a second target for the resume (multi-cloud)
- [ ] ADR: where visitor launches run + abuse/cost controls

### Guardrails (required before going public)
- **Automatic teardown** of every launch (TTL ≈30–60 min), plus a sweeper that removes anything orphaned
- **Concurrency cap** (1–2 live launches) + per-visitor rate limiting
- **Bot protection** on the Launch button (Cloudflare Turnstile)
- **Hard cost ceiling**: AWS Budgets alarm + action; small/spot instances only
- **Throwaway credentials per launch**, shown only to that visitor; nothing reused across launches
- **Isolation**: launched stacks can't reach the home lab, its management VLAN, or each other

### Definition of Done
- A stranger can click Launch, watch the build live, and log into what they launched within a few minutes
- Every launch tears itself down, and nothing is left running or billing afterwards (verified)
- Spend stays under the budget cap under abuse (tested)
- The whole platform is described in the repo: architecture, ADR, runbook

---

## Maintenance & backlog

Cross-cutting tasks not tied to a single phase — done deliberately, as their own focused sessions.

- [ ] **PVE 8 → 9 major upgrade** (Debian 12 → 13 base). *No urgency — 8.4 is supported.* Standalone session: run `pve8to9 --full` on each node and clear all warnings; take full PBS backups (incl. the OPNsense VM) first; upgrade **rolling, one node at a time**, keeping quorum and console access. Blast radius: rebooting teejhost2 drops OPNsense → whole-lab outage (and Tailscale) for the window, so plan a maintenance window. Write it up as a runbook. Follow the official Proxmox "Upgrade from 8 to 9" wiki for exact steps.
- [ ] Confirm/clean up the second OPNsense WAN `192.168.8.x` address (VIP vs leftover). An ARP sweep shows two *different* Proxmox MACs (`.5`, `.161`), so it's not a VIP; find which VM owns the second.
- [ ] Switch management onto VLAN 10 (firmware-permitting) — see naming/break-glass notes.
- [x] **Pi NAS DR rebuild** (2026-10-11): the NAS didn't survive a cold start. Rebuilt on a new card (OMV 8), ZFS pool force-imported intact, config restored via `omv-rpc`, QDevice re-linked (3/3 votes). Write-up: `docs/runbooks/pi-nas-dr-rebuild.md`; operating doc: `docs/runbooks/lab-cold-start.md`.
- [ ] NAS hardening follow-ups: root SSH → key-only; persistent journal (writecache/flashmemory left no logs from the failed boots); scheduled OMV config backup + pre-flashed spare card; console access (micro-HDMI or Debug Probe).
- [x] OPNsense cert renewed after expiring while the lab was off (re-imported via `lecert.php --mode import`; valid to 2027-01-09). PBS datastore and `teejlab-share` CIFS confirmed back online after the rebuild.
- [ ] Confirm the "Restart OPNsense Web UI" automation is attached to the `edge.teejlab.dev` ACME cert; turn OPNsense SSH password login back off (keys only).

---

## Success Metrics

| Metric | Target | Purpose |
|--------|--------|---------|
| Automation % | >90% of deployments via git push | Shows DevOps maturity |
| MTTR (Mean Time To Recover) | <5 min for common failures | Operability |
| Code coverage | IaC + CI/CD validated in tests | Quality gate |
| Documentation | Every major component has a runbook | Knowledge transfer |
| Visibility | All services observable via dashboards | Observability |

---

## Current Status
- **Focus**: **Phase 7, resume track first.** Build the cloud resume launcher end to end and polish it. It's the smallest complete version of the capstone (one pipeline, one target, one live view), it doesn't depend on any unfinished lab work, and it's the template the MSP stack will reuse.
- **Lab track (parallel, lower priority)**: Phase 1 is nearly complete (firewall hardening is the last big item); Docker host + reverse proxy on docker1 is next there; then the lab mail server.
- **Blockers**: None
- **Next Immediate Task**: Resume launcher, milestone **R1a**: resume container deployed to **Proxmox** with Terraform from my machine, then torn down. Prerequisites: a least-privilege Proxmox API token for Terraform, an Ubuntu cloud-image template, and Terraform installed.

---

**Last Updated**: 2026-10-11  
**Revision**: 0.10
