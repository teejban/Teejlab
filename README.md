# Homelab

A self-hosted infrastructure lab built on Proxmox to learn and demonstrate DevOps practices end-to-end: networking, infrastructure-as-code, containerization, observability, and GitOps. Built in shared housing on consumer hardware, with the design tradeoffs that come with both constraints.

This repo contains the architecture, decisions, and — over time — the code that runs it.

## At a glance

| | |
|---|---|
| **Hypervisor** | Proxmox VE, 2-node cluster (ThinkCentre M920q ×2) with Pi 5 QDevice |
| **Storage** | ZFS pool on Pi 5 + 4× SSD SATA HAT (~1.3 TiB), served via SMB |
| **Network** | OPNsense (VM) routing 5 segmented VLANs over a TP-Link L2 managed switch |
| **Upstream** | GL-iNet travel router behind landlord WiFi (no control of upstream) |
| **Backups** | Proxmox Backup Server |
| **Domain** | `teejlab.dev` (Cloudflare DNS, enables Let's Encrypt DNS-01 for internal TLS) |

## Architecture

OPNsense is the lab's actual router. The travel router is treated as an opaque ISP-equivalent: no control over its upstream, no port-forwarding through the building's network, no static IP. The lab lives behind two layers of NAT, and inbound publishing uses Cloudflare Tunnel rather than the conventional port-forward route.

Layer-2 segmentation comes from the managed switch passing 802.1Q tagged frames to a VLAN-aware Proxmox bridge, which trunks them through to OPNsense's LAN interface. Each VLAN has its own subnet, DHCP scope, and firewall policy.

See [`docs/network/network-architecture.md`](docs/network/network-architecture.md) for the full design — topology, VLAN scheme, IP plan, traffic flow.

## Repo layout

```
docs/
  architecture/   high-level design, decision records
  network/        VLAN design, OPNsense, switch config
  runbooks/       operational procedures, troubleshooting writeups
infra/
  terraform/      Proxmox VM provisioning           (planned)
  ansible/        host config management            (planned)
  docker/         self-hosted services              (planned)
  k8s/            manifests and Helm charts         (planned after Docker phase)
```

## Roadmap

- [x] Multi-VLAN segmented network with OPNsense routing
- [x] End-to-end validation: tagged frames through Proxmox VLAN-aware bridge to client
- [ ] Terraform module for Proxmox VM provisioning (`bpg/proxmox` provider)
- [ ] Ansible playbooks for host and VM configuration
- [ ] Docker host with self-hosted services (Gitea, BookStack, monitoring)
- [ ] Migrate workloads from Docker Compose to k3s
- [ ] GitOps with ArgoCD
- [ ] Observability stack: Prometheus, Grafana, Loki, Alertmanager
- [ ] Public portfolio site behind Cloudflare Tunnel with ephemeral per-visitor demos

## Notes

This lab is documented as it's built rather than retroactively cleaned up. Commits include real debugging, dead-ends, and corrections. The git log itself is part of the portfolio.

Most of the architecture's quirks — triple NAT, Cloudflare Tunnel for inbound, no traditional port forwarding — flow from the housing constraint. Naming them explicitly is more honest than pretending the setup is greenfield.
