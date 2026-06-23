# Roadmap: Homelab DevOps Portfolio

**Goal**: Build a self-hosted "fake MSP" environment to demonstrate DevOps skills (IaC, CI/CD, containers, GitOps, observability).

**Success Criteria**: Automated, reproducible, well-documented infrastructure that looks professional on a portfolio.

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
- [x] TLS — real Let's Encrypt certs (DNS-01 / Cloudflare) on the four infra UIs (`teejhost1`/`teejhost2`/`pbs`/`edge`), auto-renewing (see `docs/runbooks/tls-letsencrypt-dns01.md`). Pending: reverse proxy + `*.teejlab.dev` wildcard for auto-TLS on services; NAS fronted via that proxy; switch is HTTP-only
- [x] Tailscale on OPNsense advertising lab subnets (remote access) — subnet router advertising `10.0.10.0/24`, reachable web + SSH from off-network, + split-DNS so `*.teejlab.dev` resolves on the laptop
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
- [ ] Docker VM deployed via IaC
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

## Phase 6: MSP Stack + Observability + Resume Site

**Goal**: Deploy a realistic set of services that look impressive on a portfolio.

### Deliverables
- [ ] Monitoring stack: Prometheus + Grafana
- [ ] Centralized logging: Loki or ELK
- [ ] Alerting: AlertManager → email/Slack/webhook (email via the lab mail server below)
- [ ] Lab mail server (Mailcow on `lab.teejlab.dev`) — internal alerts/notices + user/app testing; **no public inbound** (per [ADR-0007](decisions/0007-hybrid-cloud-edge-vs-home-compute.md))
- [ ] Outbound mail relay via **Amazon SES** (the cloud trust-edge) + SPF/DKIM/DMARC on the subdomain via Cloudflare
- [ ] Backup automation: Proxmox Backup Server + retention policy
- [ ] Ephemeral resume website (spun up on-demand, deployed by CI/CD)
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
8. Build resume site (static HTML or lightweight Node app)
9. Deploy via CI/CD workflow

### Risks
- **Over-engineering**: Build too much, run out of time. Mitigation: prioritize (monitoring > logging > resume site).
- **Resume site as distraction**: Spend too much time on appearance. Mitigation: keep it simple (HTML + CSS).

### Definition of Done
- Grafana dashboards reflect real infrastructure health
- Alert fires (e.g., high CPU) and fires Slack notification
- Logs are centralized and searchable
- Resume site accessible via public URL (DMZ VLAN 50)
- All deployment documented in architecture diagrams + README

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
- **Phase**: 1 (Network Foundation) — nearly complete; firewall hardening is the last big item. Bridging early into Phase 3 (Docker) for the reverse-proxy auto-TLS layer.
- **Blockers**: None
- **Next Immediate Task**: Stand up the first **Docker host + reverse proxy** (Caddy/Traefik) for automatic TLS on services (also fronts the NAS UI), then the **firewall hardening pass**, then the lab mail server.

---

**Last Updated**: 2026-06-23  
**Revision**: 0.8
