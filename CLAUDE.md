# CLAUDE.md

Persistent context for Claude Code when working in this repository.

## What this project is

This repo documents a personal homelab built on a 2-node Proxmox cluster, with the explicit goal of demonstrating DevOps practices to land a DevOps-focused role. Everything in this repo should read as if a hiring manager might see it — clear, evidence-based, honest about constraints, and showing actual debugging and design work rather than tutorial output.

Project handle: `teejlab` (also the owned domain `teejlab.dev`).

## My goals

1. Land a DevOps job. The repo and the homelab itself are portfolio material.
2. Learn the full DevOps stack hands-on: IaC, config management, containers, orchestration, CI/CD, GitOps, observability.
3. Build something real and useful for myself in the process — not just demo workloads.

## Hardware inventory

### Hypervisor cluster

- **teejhost1** — Lenovo ThinkCentre M920q (mini PC). 1 NIC (`eno1`). Proxmox node; also hosts the PBS VM (VMID 101). Dual-homed: `192.168.8.119` (flat) + `10.0.10.2` (MGMT VLAN 10, tagged `vmbr0.10`).
- **teejhost2** — Lenovo ThinkCentre M920q (mini PC). 4 NICs (added 4-port card). Proxmox node. Hosts OPNsense VM. IP `192.168.8.138`.
- Cluster totals: 12 CPUs, ~94 GiB RAM, ~888 GiB storage.

### Storage

- **teejlab-pi-nas** — Raspberry Pi 5 (8GB), SATA HAT with 4 SSDs, ZFS pool `teejlab-tank` (~1.32 TiB). Runs OpenMediaVault 7. Doubles as cluster QDevice and SMB target for Proxmox + other clients. NIC `eth0`. Dual-homed: `192.168.8.230` (flat) + `10.0.10.4` (MGMT VLAN 10, tagged `eth0.10`).

### Networking

- **TP-Link TL-SG108E** managed switch (8 ports, 802.1Q VLANs). Port assignments:
  - Port 2 → teejlab-pi-nas (trunk: untagged VLAN 1 + tagged VLAN 10)
  - Port 3 → teejhost1 (trunk: untagged VLAN 1 + tagged VLAN 10)
  - Port 4 → travel router uplink (kept for legacy flat network during transition)
  - Port 5 → teejhost2 (trunk port to OPNsense)
  - Port 6 → VLAN 30 access (lab test port)
  - Ports 1, 7, 8 → currently free
- **GL-iNet GL-A1300 (Slate Plus)** travel router — upstream link, runs OpenWRT, connects to landlord's WiFi as WAN.

### Other

- Dev machine: Windows + Linux (CachyOS / Hyprland for the Linux side).

## Network architecture

OPNsense (VM on teejhost2) is the lab's actual router. Travel router is treated as an opaque ISP-equivalent upstream.

### VLAN scheme

| VLAN | Name    | Subnet         | Gateway     | Purpose                          |
|------|---------|----------------|-------------|----------------------------------|
| 1    | Default | 192.168.8.0/24 | 192.168.8.1 | Legacy flat net (transitional)   |
| 10   | MGMT    | 10.0.10.0/24   | 10.0.10.1   | Infrastructure management        |
| 20   | SERVERS | 10.0.20.0/24   | 10.0.20.1   | Production services              |
| 30   | LAB     | 10.0.30.0/24   | 10.0.30.1   | Ephemeral test workloads         |
| 40   | TRUSTED | 10.0.40.0/24   | 10.0.40.1   | Personal admin devices           |
| 50   | DMZ     | 10.0.50.0/24   | 10.0.50.1   | Internet-facing services         |
| 99   | IOT     | 10.0.99.0/24   | 10.0.99.1   | Reserved (DHCP not enabled yet)  |

Triple NAT (lab → OPNsense → travel router → landlord). Inbound for public services handled via Cloudflare Tunnel.

## Domain and TLS

- Domain: `teejlab.dev` (owned — registered for 2 years). DNS hosted on Cloudflare.
- Why `.dev`: cheaper and more stable than `.io`, and reads well for a DevOps portfolio. `.dev` is on the HSTS preload list, so browsers force HTTPS on it — which lines up with the all-TLS plan below.
- Why Cloudflare for DNS: the inbound plan is Cloudflare Tunnel, which requires the zone to live on Cloudflare. Same account serves the Tunnel and the DNS-01 API token. (Registrar choice was deliberately decoupled from cloud — DNS host is what matters here, not where AWS workloads run. If real public AWS endpoints ever appear, delegate a subdomain like `aws.teejlab.dev` to Route 53 rather than moving the zone.)
- Internal hostname pattern: `<service>.teejlab.dev` even for internal-only services.
- TLS plan: Let's Encrypt via DNS-01 challenge, since the public DNS is mine. No browser cert warnings for internal services — same UX as public ones. (Note: ACM certs are not usable here — they can't be exported off AWS services — so Let's Encrypt is the path for homelab TLS regardless of any AWS usage.)

## Naming conventions

- Hypervisors: `teejhost<N>` (e.g., `teejhost1`, `teejhost2`)
- Services and infrastructure: `teejlab-<service>` (e.g., `teejlab-pi-nas`, `teejlab-tank`)
- Internal DNS: `<service>.teejlab.dev`
- VLANs: ALL CAPS short names (`MGMT`, `SERVERS`, etc.) matching their internal purpose
- IP scheme: `10.0.<vlan-id>.0/24`, OPNsense as `.1`, DHCP pools start at `.10`

## Key constraints (always relevant)

- **Shared housing**: upstream is the landlord's WiFi via travel router. No control over upstream router, no static IP, no port forwarding through the building's network. This is *the* constraint that shapes most decisions — inbound publishing via Cloudflare Tunnel, no traditional NAT setups, double/triple-NAT accepted as a fact of life.
- **Asymmetric NIC count**: teejhost1 has 1 NIC (everything trunked), teejhost2 has 4 (OPNsense gets dedicated NICs for WAN and LAN-trunk).
- **TP-Link Easy Smart switch quirks**: 802.1Q config across three menus (VLAN, PVID, port), requires explicit Save to persist across reboots, requires at least one port member to create a VLAN.
- **Work-laptop segmentation**: corporate device with Zscaler. Should not have lab network access. Future plan is Tailscale-based bridge for specific cross-device needs (Barrier KVM).

## Tooling and current state

### Already in place
- Proxmox cluster with QDevice quorum
- Proxmox Backup Server (VM 101 on teejhost1)
- OPNsense routing, DHCP per VLAN, basic firewall
- Switch with 802.1Q VLANs configured
- End-to-end DHCP validated on LAB VLAN
- teejhost1 and teejlab-pi-nas dual-homed onto MGMT VLAN 10 (corosync still on flat net pending the cluster-wide ring migration; teejhost2 not yet migrated)

### Planned / in progress (DevOps focus)
- IaC: Terraform + `bpg/proxmox` provider for VM provisioning
- Config management: Ansible
- Container platform: Docker first (to learn fundamentals properly), migrate to k3s after a few months
- CI/CD: GitHub Actions (portfolio visibility), with self-hosted runners in the lab
- GitOps: ArgoCD or Flux after k3s migration
- Observability: Prometheus, Grafana, Loki, Alertmanager
- Secrets: TBD (probably Vault or just sops with age for now)
- Remote access: Tailscale on OPNsense, advertising lab subnets

### Self-hosted services planned (the "MSP stack" theme)
- Gitea (self-hosted git mirror of public GitHub repos)
- Tactical RMM (open-source MSP tool — fits the lab's theme)
- Zammad or GLPI (ticketing)
- BookStack (documentation / runbooks)
- Uptime Kuma (simple monitoring)
- A public-facing portfolio site behind Cloudflare Tunnel, ideally with an interactive demo (ephemeral container spawned per visitor) — long-term goal

## Repo structure

```
docs/
  architecture/      # high-level design, diagrams, decisions
  network/           # VLAN design, OPNsense setup, switch config
  runbooks/          # operational procedures, troubleshooting writeups
  decisions/         # ADRs (architecture decision records)
infra/
  terraform/         # IaC for Proxmox VMs (planned)
  ansible/           # config management (planned)
  docker/            # compose files for self-hosted services
  k8s/               # manifests / helm charts (after migration)
```

## Documentation style preferences

When writing or editing documentation in this repo:

- **Show the why, not just the what.** A diagram of the network is less interesting than a paragraph explaining the constraint that drove the design.
- **Be honest about constraints.** "Triple NAT due to shared housing" is more compelling than a sanitized version pretending the setup is greenfield.
- **Use real evidence in troubleshooting writeups.** Include actual tcpdump output, real `bridge vlan show` snapshots, the actual error messages. Replace specifics like MAC addresses only if there's a real reason to.
- **Headlines first, details second.** Each doc should open with a one-paragraph summary so a reader can stop after 30 seconds and still know what it's about.
- **Lessons learned are gold.** Always include a "what I'd do differently" or "what tripped me up" section in runbooks. This is what hiring managers actually want to see.
- **Avoid AI-generated tone.** No "let's dive in," no "in this comprehensive guide," no "embark on a journey." Direct, technical, conversational where appropriate.
- **Format**: prefer prose with selective use of tables, lists, and code blocks. Avoid heavy bullet-point walls. Use H2 for major sections, H3 sparingly.

## Conventions

- Conventional commits: `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`
- Branch from `main`, PR back, even when working solo (builds interview-friendly habits)
- Sign commits (SSH or GPG) — the green "Verified" badge matters for portfolio visibility
- No secrets in the repo, ever. Use sops + age for anything sensitive.

## Lessons learned (running list — add to this as we go)

### Proxmox VLAN-aware bridge requires explicit `trunks` per VM NIC
When making a Proxmox bridge VLAN-aware, the bridge itself accepting tagged frames is only half the work. Each VM NIC's `tap` interface defaults to VLAN 1 (PVID) only, so tagged frames hit the bridge but get dropped before reaching the VM. Fix: add `trunks=10;20;30;40;50` (semicolon-separated) to the VM NIC line in `/etc/pve/qemu-server/<vmid>.conf`, or set the Trunks field in the GUI if your Proxmox version exposes it. Verify with `bridge vlan show`.

Diagnostic flow that found it: tcpdump on the physical NIC (saw tagged DHCP) → tcpdump on vmbr0 (still saw it) → tcpdump on the VM's tap (saw nothing). The "missing" hop is the bug.

### Single-NIC host management on a VLAN = dual-homing, not a second NIC
A host with one NIC can still live on a management VLAN: keep the existing untagged IP on the bridge and add a tagged VLAN sub-interface (`vmbr0.10` on Proxmox, `eth0.10` on the Pi) for the new IP, then make the switch port a trunk (untagged VLAN 1 PVID + tagged VLAN 10). Both IPs ride one cable. Keep the flat IP in place — for a clustered node it carries corosync, and it's your no-lockout fallback. Only one default gateway across the two interfaces.

Debugging note: `Destination Host Unreachable` *from your own VLAN IP* means ARP isn't resolving — an L2 problem (the switch port wasn't trunking the VLAN yet), not an IP/config problem. The actual fix that bit us: the TP-Link needs **Modify** to apply a VLAN membership change, then a separate **Save Config** to persist it.

### OpenMediaVault owns its network config — don't hand-edit
OMV stores config in its own database and regenerates the live `systemd-networkd` files on apply, so editing config files directly gets silently overwritten. Add VLAN interfaces through the OMV UI (Network → Interfaces → Create → VLAN), then **Apply** the pending-changes banner. Equivalent from the shell: `sudo omv-salt deploy run systemd-networkd` (must be root, or it fails with a salt-cache `PermissionError`). A reboot does *not* deploy pending changes.

### TP-Link Easy Smart switch requires at least one port to create a VLAN
The 802.1Q VLAN config form won't accept a new VLAN unless at least one port is set to Tagged or Untagged. Use any unused port as a temporary placeholder, then re-edit the VLAN to set real port memberships afterward.

### TP-Link Easy Smart switches don't auto-save
Apply makes changes live in RAM; Save Config (under System Tools) commits to flash. Without the explicit Save, a reboot wipes the config. Confirmed by rebooting after each major change and verifying the config persists.
