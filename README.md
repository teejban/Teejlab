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

See [`docs/architecture/network-architecture.md`](docs/architecture/network-architecture.md) for the full design — topology, VLAN scheme, IP plan, traffic flow.

## Status

**Phase 1 — Network Foundation** (in progress). OPNsense routing and all five VLANs are live; management interfaces are migrating from the flat network onto a dedicated MGMT VLAN. Up next: IaC with Terraform and Ansible.

The full phased plan and progress live in [`docs/roadmap.md`](docs/roadmap.md).

## Repo layout

```
docs/
  roadmap.md      phased plan and progress
  architecture/   high-level design, network topology
  runbooks/       operational procedures, troubleshooting writeups
  decisions/      architecture decision records (ADRs)
infra/
  terraform/      Proxmox VM provisioning           (planned)
  ansible/        host config management            (planned)
  docker/         self-hosted services              (planned)
  k8s/            manifests and Helm charts         (planned after Docker phase)
```

## Notes

This lab is documented as it's built rather than retroactively cleaned up. Commits include real debugging, dead-ends, and corrections. The git log itself is part of the portfolio.

Most of the architecture's quirks — triple NAT, Cloudflare Tunnel for inbound, no traditional port forwarding — flow from the housing constraint. Naming them explicitly is more honest than pretending the setup is greenfield.
