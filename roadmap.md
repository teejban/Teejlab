# Roadmap: Homelab DevOps Portfolio

**Goal**: Build a self-hosted "fake MSP" environment to demonstrate DevOps skills (IaC, CI/CD, containers, GitOps, observability).

**Success Criteria**: Automated, reproducible, well-documented infrastructure that looks professional on a portfolio.

---

## Phase 1: Network Foundation ⭐ CURRENT

**Goal**: Segment the flat network, deploy OPNsense, enable VLAN routing and internal DNS.

**Why First**: All downstream services depend on stable, isolated networking. Easier to design right now than refactor later.

### Deliverables
- [ ] VLAN configuration on TP-Link switch (ports tagged/untagged)
- [ ] OPNsense VM deployed on Node 1 with 2 NICs in passthrough
- [ ] OPNsense: Uplink configured (WAN), internal routing (LAN)
- [ ] OPNsense: Firewall rules for basic inter-VLAN policies
- [ ] Internal DNS (TBD: AdGuard Home, Dnsmasq, or OPNsense DNS)
- [ ] VLAN isolation verified (cross-VLAN traffic blocked)
- [ ] All existing services migrated to VLAN 10 (management)
- [ ] Network documentation updated

### Technical Steps
1. Plan IP allocations and VLAN membership (network-plan.md)
2. Configure switch VLAN groups + tagging
3. Deploy OPNsense (ISO boot, install, basic OS config)
4. Configure OPNsense NICs (bridge, IP assignment, VLAN handling)
5. Set up OPNsense firewall rules
6. Deploy internal DNS service
7. Test inter-VLAN routing and isolation
8. Document any gotchas or MAC-address reservations

### Risks
- **Lock yourself out**: OPNsense misconfiguration could break connectivity. Mitigation: console access always available, test in lab VLAN first.
- **Switch vlan misconfiguration**: Flat network becomes split. Mitigation: document current port memberships before any changes.

### Definition of Done
- Proxmox nodes can reach each other via VLAN 10
- Lab/test VLAN can ping each other but not other VLANs
- Internal DNS resolves .internal hostnames
- Documentation is complete (network-plan.md updated)

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
- [ ] Alerting: AlertManager → Slack/webhook
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
6. Document disaster recovery (restore from PBS)
7. Build resume site (static HTML or lightweight Node app)
8. Deploy via CI/CD workflow

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
- **Phase**: 1 (Network Foundation)
- **Blockers**: None
- **Next Immediate Task**: Plan VLAN configuration, then deploy OPNsense

---

**Last Updated**: 2026-06-18  
**Revision**: 0.1
