# docker

Compose projects that run on **docker1** (`10.0.20.2`, VLAN 20 SERVERS), one folder per project.
This repo is the source of truth for them: changes are made here, merged by PR, then pulled onto
docker1 and applied with `docker compose up -d`. docker1 only ever reads from GitHub.

| Project | What it runs | URL |
|---|---|---|
| [`traefik/`](traefik/) | Traefik v3 reverse proxy holding the `*.teejlab.dev` wildcard (Let's Encrypt DNS-01 via Cloudflare), plus a `whoami` test service | `https://whoami.teejlab.dev` |
| [`web/`](web/) | nginx serving a static test page, routed through Traefik with no published ports | `https://site.teejlab.dev` |

Build notes, design decisions and the gotchas hit along the way are in
[`docs/runbooks/docker-host-setup.md`](../../docs/runbooks/docker-host-setup.md).

## What is deliberately not in git

| Path (per project) | Why |
|---|---|
| `traefik/secrets/` | The Cloudflare API token. It can edit `teejlab.dev` DNS. |
| `traefik/letsencrypt/` | `acme.json`: the certificate's private key and the ACME account key. Generated, never committed. |
| `traefik/.env` | Personal/environment values (`ACME_EMAIL`). `.env.example` shows what to fill in. |

Each project's `.gitignore` enforces this. When copying from docker1, files were copied **by
name**, so the secrets never left the host. The `.gitignore` is the second safety net, not the first.

## Setting up a fresh Docker host

Assumes Docker Engine + the Compose plugin are installed and the user is in the `docker` group
(steps 1–2 of the runbook).

```sh
git clone https://github.com/teejban/Teejlab.git ~/Teejlab
cd ~/Teejlab/infra/docker

# 1. Shared network that Traefik and every proxied service join (owned by no single project)
docker network create proxy

# 2. Traefik: environment values + the Cloudflare token (prompted, never echoed or in history)
cd traefik
cp .env.example .env && nano .env                      # set ACME_EMAIL
mkdir -p secrets letsencrypt && chmod 700 secrets
read -rsp "Cloudflare token: " CF && printf '%s' "$CF" > secrets/cf_dns_api_token && unset CF && echo
chmod 600 secrets/cf_dns_api_token
docker compose up -d                                   # requests the wildcard cert on first start

# 3. Everything else
cd ../web && docker compose up -d
```

Each service also needs an Unbound host override (`<name>.teejlab.dev → 10.0.20.2`, then
**Apply**). The Cloudflare token needs **DNS Write + Zone Read on `teejlab.dev` only**.

## Adding a service behind Traefik

Put the container on the `proxy` network, add four labels and add one DNS name. Don't publish any
ports.

```yaml
services:
  myapp:
    image: ...
    labels:
      - traefik.enable=true
      - traefik.http.routers.myapp.rule=Host(`myapp.teejlab.dev`)
      - traefik.http.routers.myapp.entrypoints=websecure
      - traefik.http.routers.myapp.tls.certresolver=cf
    networks:
      - proxy

networks:            # last top-level block in the file
  proxy:
    external: true
```

If the app listens on a port other than 80, add
`traefik.http.services.myapp.loadbalancer.server.port=<port>`.
