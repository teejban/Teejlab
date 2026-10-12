# runbooks

Operational procedures and troubleshooting write-ups. Each opens with a short summary and ends
with what tripped me up.

## Operating the lab

- [lab-cold-start.md](lab-cold-start.md): power-on order, post-boot health check, every device's address and break-glass access path, plus the NAS rescue procedure
- [linux-troubleshooting.md](linux-troubleshooting.md): cheat sheet for down, slow or won't-boot problems, with real examples

## Incidents

- [pi-nas-dr-rebuild.md](pi-nas-dr-rebuild.md): 2026-10: the Pi NAS didn't survive a cold start; DR rebuild onto a new card (OMV 8, ZFS import, config via `omv-rpc`)

## Build and migration procedures

- [teejhost1-management-vlan-migration.md](teejhost1-management-vlan-migration.md), [teejhost2-management-vlan-migration.md](teejhost2-management-vlan-migration.md), [pi-nas-management-vlan-migration.md](pi-nas-management-vlan-migration.md): moving each host onto MGMT VLAN 10
- [corosync-vlan-migration.md](corosync-vlan-migration.md): moving the cluster network onto VLAN 10 without a split
- [pbs-rebuild.md](pbs-rebuild.md): PBS rebuilt VLAN-10-native, datastore on the Pi over NFS
- [internal-dns.md](internal-dns.md): split-horizon `teejlab.dev` via OPNsense Unbound
- [tailscale-subnet-router.md](tailscale-subnet-router.md): remote access without port forwarding
- [tls-letsencrypt-dns01.md](tls-letsencrypt-dns01.md): real certs on the infra UIs via DNS-01
- [vm-baseline-setup.md](vm-baseline-setup.md): first-boot hardening standard for every VM
- [docker-host-setup.md](docker-host-setup.md): Docker host role on docker1 (in progress)
