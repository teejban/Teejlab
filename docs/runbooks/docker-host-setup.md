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

## 0. Bring docker1 back after the shutdown  `[x]`  *(2026-10-15)*

docker1 had been off since July. Before installing anything, the baseline was re-checked: still
on `10.0.20.2` with its default route via `10.0.20.1`, and `ssh qemu-guest-agent
prometheus-node-exporter unattended-upgrades` all `active`. Then `sudo apt update && sudo apt
upgrade -y` (108 packages pending after three months). Key-only SSH did its job: logging in from
teejhost1 got `Permission denied (publickey)`, because the only authorized key is the Windows
dev box's, used via the `ssh docker1` alias in `~/.ssh/config`. VLAN 20 is reachable from the
PC because Tailscale on OPNsense advertises `10.0.20.0/24` (plus 10 and 30).

## 1. Docker Engine from Docker's official apt repo  `[x]`

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

Done 2026-10-15 using the key in `/etc/apt/keyrings/docker.asc` (ASCII-armored, readable by
`_apt`) and a `signed-by=` repo line written with `sudo tee`. Result:

```
Docker version 29.9.1, build dfc520d
Docker Compose version v5.6.0
  Installed: 5:29.9.1-1~ubuntu.22.04~jammy
        500 https://download.docker.com/linux/ubuntu jammy/stable amd64 Packages
```

Service `active` + `enabled`; no competing `docker.io`/snap installs.

## 2. Post-install — run Docker without root  `[x]`

Add your user to the `docker` group so you don't `sudo` every command:
`sudo usermod -aG docker teejadmin`, then **log out and back in** for the group to take effect.
Security note to record: the `docker` group is **root-equivalent** (it can mount the host FS) —
fine on a single-admin lab box, but worth stating honestly rather than glossing.

`-a` in `usermod -aG` matters: without it `-G` *replaces* the group list and drops you out of
`sudo`. Verified with `id -nG` showing `docker` and `docker run hello-world` working without sudo.

## 2b. First containers, from `docker run` to Compose  `[x]`

Learning steps before Traefik, each building on the last:

1. `docker run -d --name web -p 8080:80 nginx`: detached container, named, host port 8080 →
   container port 80. Reachable at `http://10.0.20.2:8080`; `docker logs web` shows the request.
2. Swap in a bind mount, `-v ~/site:/usr/share/nginx/html:ro`, to serve my own page. You don't
   edit a container; you `docker rm -f` it and run a new one. Edits to the host folder show up
   live, because the content lives outside the container.
3. The same thing as `~/web/compose.yaml`, with `restart: unless-stopped` added. Flags map to
   keys (`--name` → `container_name`, `-p` → `ports`, `-v` → `volumes`, image → `image`; `-d`
   moves to `docker compose up -d`). `docker compose config` validates the file first, like
   `terraform validate`. `down` then `up -d` recreates it identically in seconds.

## 3. Internal DNS record  `[x]`  *(OPNsense-side, not on the VM)*

Unbound host overrides **`docker1.teejlab.dev → 10.0.20.2`** and **`whoami.teejlab.dev →
10.0.20.2`** (the Traefik test service). Each proxied service gets **its own override pointing at
docker1**, because Traefik routes by requested name, so every name lands on the same IP.

Gotcha: the overrides were saved but returned NXDOMAIN until **Apply** was clicked. It's a small
button at the bottom of the Overrides page, the same "saved isn't live" pattern as the switch and
OMV. Deliberately **no catch-all `*.teejlab.dev` override**: it would capture typos and any name
meant to resolve publicly. Per-service records for now; automate them later via the OPNsense
API (Terraform/Ansible), or use a dedicated wildcard subdomain such as `*.apps.teejlab.dev`.

## 4. First workload — reverse proxy (Traefik)  `[x]`  *(2026-10-16)*

The keystone of the lab's **automatic-TLS** plan
([ADR-0009](../decisions/0009-automatic-tls-strategy.md)): a reverse proxy holding a
`*.teejlab.dev` wildcard cert (Let's Encrypt DNS-01 via Cloudflare), with **Traefik** chosen for
label-based auto-discovery. New containers get TLS by adding labels, with no per-service cert
work.

**Layout** (`~/traefik/`): `compose.yaml` (Traefik + `traefik/whoami` test service),
`secrets/cf_dns_api_token` (`700` dir, `600` file, not in git), `letsencrypt/acme.json`
(created by Traefik, root-owned, `600`).

**Cloudflare token:** a dedicated token, `traefik-docker1-dns01`, scoped to **`teejlab.dev` only**,
with **DNS Write + Zone Read** (Traefik looks the zone up, then edits records), no expiry and no IP
filter (the residential IP changes). Kept in Bitwarden as the master copy. On docker1 it was
written with `read -rsp` so it never appears on screen or in history, and Traefik reads it as a
Compose secret via `CF_DNS_API_TOKEN_FILE=/run/secrets/...`, not an environment variable (which
`docker inspect` would show).

Verifying the token tripped me up: Cloudflare's new token UI creates **account-owned** tokens,
and `/user/tokens/verify` rejects those with `Invalid API Token` even when the token is fine. Test
what Traefik actually needs instead:

```
curl -s -H "Authorization: Bearer $(cat ~/traefik/secrets/cf_dns_api_token)" \
  "https://api.cloudflare.com/client/v4/zones?name=teejlab.dev"   # → "success":true
```

**Key Traefik settings and why:**
- `providers.docker.exposedbydefault=false`: only containers labelled `traefik.enable=true`
  get routed. Nothing is exposed by accident.
- `web` (:80) redirects to `websecure` (:443). Only Traefik publishes ports; backends have none.
- `dnschallenge.resolvers=1.1.1.1:53,1.0.0.1:53`: check challenge propagation against **public**
  DNS, because internal Unbound answers `teejlab.dev` itself (split-horizon).
- A wildcard cert requested once on the whoami router (`tls.domains[0].main=teejlab.dev`,
  `sans=*.teejlab.dev`) and shared by every service.
- `docker.sock` mounted `:ro`. Socket access is effectively root on the host, so it's limited to
  Traefik.

**Staging first, then production.** Started against the Let's Encrypt **staging** CA (no rate
limits). Result:

```
Issuer : CN=(STAGING) Ersatz Emmer YR2, O=Let's Encrypt, C=US
Names  : DNS Name=*.teejlab.dev, DNS Name=teejlab.dev
```

A browser won't accept a staging cert on `.dev` (HSTS, so there's no click-through), so test with
`curl -k` or a TLS check instead. whoami's reply shows the proxying: `IP: 172.19.0.3` (whoami on
the Docker network) and `RemoteAddr: 172.19.0.2` (Traefik). Then: delete the `caserver` line,
`sudo rm letsencrypt/acme.json` (otherwise the staging cert is kept), `docker compose up -d`.
Production cert: issuer `YR1`, valid 2026-10-16 → 2027-01-14. `https://whoami.teejlab.dev`
loads with a valid padlock.

Downstream of this (own steps):
- [x] A shared external Docker network so services in *other* Compose projects are reachable by Traefik (4b)
- [x] Put the nginx test site behind it (4b)
- [ ] The NAS UI (no native ACME). It isn't a container, so it needs Traefik's **file provider**.
- [ ] Firewall allows for client → proxy once the hardening pass lands.
- [ ] Pin the image to an exact version (`traefik:v3.x.y`) instead of the floating `v3` tag.

## 4b. Shared `proxy` network: the nginx site behind Traefik  `[x]`  *(2026-10-19)*

Each Compose project gets its own private bridge network (`traefik_default`, `web_default`), so
Traefik couldn't see the nginx container in the separate `~/web` project. The fix is one network
that both projects join, created **by hand, outside any Compose file**. If one project owned it,
`docker compose down` there would try to delete it from under the others.

```
docker network create proxy
```

**Traefik** (`~/traefik/compose.yaml`) joins it and keeps its own network, because once a
service lists `networks:` it's on *only* those, and it still needs `default` to reach whoami:

```yaml
    networks:
      - proxy
      - default
# …and as the LAST top-level block of the file:
networks:
  proxy:
    external: true          # exists already; use it, never create/delete it
```

**nginx** (`~/web/compose.yaml`): `ports:` removed, so 8080 is closed and Traefik is the only way
in. It gets the same labels as whoami under a new router name, and no `tls.domains`, because the
wildcard cert already exists and is reused:

```yaml
    labels:
      - traefik.enable=true
      - traefik.http.routers.site.rule=Host(`site.teejlab.dev`)
      - traefik.http.routers.site.entrypoints=websecure
      - traefik.http.routers.site.tls.certresolver=cf
    networks:
      - proxy
```

Plus an Unbound override `site.teejlab.dev → 10.0.20.2` (and **Apply**). The router is named
`site`, not `web`, so it isn't confused with Traefik's `web` entrypoint. Result:

```
web: Up  ports=[80/tcp]           ← container port only, nothing published on the host
networks: proxy
site.teejlab.dev DNS -> 10.0.20.2
https://site.teejlab.dev -> <h1>Hello from teejlab</h1>…
http://10.0.20.2:8080 -> connection refused
```

That's the pattern every service now follows: **container on `proxy` + four labels + one DNS
name = HTTPS with no published ports.** It works like an ALB with host-based routing to targets
in a private subnet: one entry point (`10.0.20.2:443`), and backends only reachable through it.

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

- **Stopped containers aren't deleted.** Both `hello-world` runs sat in `docker ps -a` as
  `Exited (0)`. Use `--rm` for throwaway runs, or `docker container prune` to clean up.
- **Compose list items go on their own line.** `ports: - "8080:80"` on one line is invalid;
  the `- "8080:80"` belongs on the next line, indented. `docker compose config` catches it.
- **Bind-mount paths must still exist.** After moving `~/site` into `~/web/site`, a compose
  file still pointing at `~/site` would mount a freshly created *empty* folder, and nginx
  returns 403. Relative paths (`./site`) keep the project portable.
- **`sudo echo … > /etc/file` fails.** The redirect runs in your unprivileged shell, so use
  `| sudo tee`.
- **A top-level YAML key ends the section above it.** The `networks: proxy: external: true` block
  pasted *between* the two services made Compose read `whoami` as a network:
  `networks.whoami additional properties 'image', 'restart', 'container_name' not allowed`.
  Top-level blocks (`services:`, `networks:`, `secrets:`) sit at zero indent; put the external
  network block at the very end.
- **Editing the file doesn't change a running container.** After adding `networks:`, Traefik
  still showed only `traefik_default` (and "Up 2 days") until `docker compose up -d` recreated it.
- **`docker compose config` doesn't validate labels.** `rule-Host('site.teejlab.dev')` (dash
  instead of `=`, quotes instead of backticks) passed validation, because to Compose a label is
  just a string. Traefik is what parses it, so label mistakes only show up in Traefik's
  behaviour and logs. Traefik rules need backticks: ``Host(`name`)``.
- **The DNS name and the `Host()` rule must match.** DNS gets the client to docker1; the label
  tells Traefik which container the name belongs to. Missing either half means no route.
