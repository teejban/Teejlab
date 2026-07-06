# Runbook: Docker host role (docker1) — repo, engine, first workload

> **Status: planned / in progress.** This is the *role* layer that runs on top of the
> universal [vm-baseline-setup.md](vm-baseline-setup.md). Steps here are the plan of record;
> they get marked `[x]` and annotated with real output as each is actually run on docker1.
> Commands are illustrative — the point is understanding each step before it becomes a script.

## Summary

Once a VM has the baseline (secure admin, comms, metrics), the **Docker-host role** turns it
into a container platform. This is the first *role* layer in the lab and the first real
IaC target (see [ADR-0010](../decisions/0010-docker-host-build-then-codify.md)): build it by
hand, understand every step, then reproduce it in Ansible as a `docker` role applied on top of
the `base` role.

Subject: **docker1** — Ubuntu 22.04, VLAN 20 SERVERS, `10.0.20.2`.

## Prerequisite

The baseline must be **complete and verified** first — this role assumes key-only SSH, patched
OS, `qemu-guest-agent`, node_exporter, and the repo prerequisites (`ca-certificates curl gnupg`)
are all in place. Don't start here until the baseline checklist is fully `[x]`.

## 1. Docker Engine from Docker's official apt repo  `[ ]`

**Not the Ubuntu `snap`, not the `docker.io` distro package** — the official repo, so you get
current Engine + the Compose v2 plugin and Docker's own update cadence. This is the concrete
example of the "role layer owns third-party repos" boundary from the baseline doc.

Shape of it (understand each piece, don't just paste):
- Add Docker's **GPG signing key** (de-armored into `/etc/apt/keyrings/`) — this is what the
  baseline's `gnupg` + `ca-certificates` were prerequisites for.
- Add the **repo definition** (`/etc/apt/sources.list.d/docker.list`) pinned to the key and
  the `jammy` release.
- `apt update` then install `docker-ce docker-ce-cli containerd.io docker-buildx-plugin
  docker-compose-plugin`.
- Verify: `sudo docker run hello-world` and `docker compose version`.

## 2. Post-install — run Docker without root  `[ ]`

Add your user to the `docker` group so you don't `sudo` every command:
`sudo usermod -aG docker teejadmin`, then **log out and back in** for the group to take effect.
Security note to record: the `docker` group is **root-equivalent** (it can mount the host FS) —
fine on a single-admin lab box, but worth stating honestly rather than glossing.

## 3. Internal DNS record  `[ ]`  *(OPNsense-side, not on the VM)*

Add an Unbound host override **`docker1.teejlab.dev → 10.0.20.2`** so the host is addressable by
name, consistent with the internal-DNS scheme. This is a hypervisor/router-side step, separate
from the VM.

## 4. First workload — reverse proxy (Traefik)  `[ ]`

The first real service, and the keystone of the lab's **automatic-TLS** plan
([ADR-0009](../decisions/0009-automatic-tls-strategy.md)): a reverse proxy holding a
`*.teejlab.dev` wildcard cert (Let's Encrypt DNS-01 via Cloudflare), with **Traefik** chosen for
label-based auto-discovery — new containers get TLS by adding labels, no per-service cert work.

Downstream of this (later, own steps):
- Internal wildcard DNS `*.teejlab.dev → 10.0.20.2` in Unbound so proxied hostnames resolve.
- Front the NAS UI (no native ACME) through the proxy.
- Firewall allows for client → proxy once the hardening pass lands.

## 5. Firewall — the UFW/Docker caveat  `[ ]`

**UFW does not protect a Docker host.** Docker writes its own `iptables` rules that bypass UFW,
so any **published** container port (`-p 8080:80`) punches straight through the firewall
silently. Decision to make here, deliberately:
- **Skip host UFW**, rely on OPNsense inter-VLAN rules + the reverse proxy (only publish what
  the proxy needs); or
- Use the **`ufw-docker`** fix that inserts rules into Docker's `DOCKER-USER` chain.

Tie this to the lab-wide firewall hardening pass rather than deciding it in isolation.

## Lessons / what tripped us up

*(fill in as we go — the gotchas found building this by hand are the spec the Ansible `docker`
role has to reproduce.)*
