# Runbook: TLS for internal services — Let's Encrypt via DNS-01 (Cloudflare)

## Summary

Internal management UIs get **real, browser-trusted, auto-renewing** Let's Encrypt certs for
their `*.teejlab.dev` names, using the **DNS-01** challenge against Cloudflare. DNS-01 is the
right (only) fit here: the services aren't publicly reachable (no inbound), so HTTP-01 can't
work — but DNS-01 only needs API access to create a TXT record, which the Cloudflare token
provides.

This was **mandatory, not cosmetic**: `.dev` is on the HSTS preload list, so browsers force
HTTPS *and refuse the "proceed anyway" bypass* for a bad/self-signed cert. Accessing a `.dev`
hostname therefore requires a valid cert.

Done for: `teejhost1`, `teejhost2` (Proxmox), `pbs` (PBS), `edge` (OPNsense).

## Prerequisite — Cloudflare API token

Cloudflare dashboard → My Profile → API Tokens → Create Token → **"Edit zone DNS"** template →
scope to **`teejlab.dev`** (`Zone → DNS → Edit`). Copy the token (shown once). Also grab the
zone's **Account ID** and **Zone ID** from the `teejlab.dev` Overview page (right sidebar).

Token-auth fields, everywhere: set **`CF_Token`** (+ **`CF_Account_ID`**, and **`CF_Zone_ID`**
to be safe). Leave the legacy **`CF_Email`/`CF_Key`** (global API key) blank.

## Proxmox VE (cluster) — built-in ACME

The ACME **account + DNS plugin are Datacenter-level (shared by all nodes)**; only the cert
order is per-node.

1. **Datacenter → ACME → Accounts → Add**: name `default`, your email, directory = Let's Encrypt
   (production), accept ToS.
2. **Datacenter → ACME → Challenge Plugins → Add**: DNS → Cloudflare → `CF_Token=…`,
   `CF_Account_ID=…`, `CF_Zone_ID=…`.
3. Per node — **<node> → System → Certificates → ACME**: add domain `<node>.teejlab.dev` (plugin
   = Cloudflare, account = default) → **Order Certificates Now**. Installs to `pveproxy`,
   auto-renews. (teejhost2 only needed this step — account/plugin were already shared.)

## PBS — built-in ACME (same idea, separate system)

PBS isn't in the Proxmox cluster, so it gets its own account + plugin:
- **Configuration → Certificates → ACME**: register account; add DNS → Cloudflare plugin (same
  token data); add domain `pbs.teejlab.dev` → Order. UI then green on `:8007`.

## OPNsense — `os-acme-client` plugin

1. Install **`os-acme-client`** (System → Firmware → Plugins; enable "Show Community Plugins").
2. **Services → ACME Client → Settings** → Enable Plugin → Apply.
3. **Accounts** → add (name e.g. `letsencrypt`, email, Let's Encrypt **production**).
4. **Challenge Types** → add: DNS-01 → DNS Service **Cloudflare** → API Token (+ Account ID).
   *(Name it `cloudflare-dns`, not `letsencrypt` — avoids confusion with the account name.)*
5. **Certificates** → add: Common Name `edge.teejlab.dev`, ACME Account + the Cloudflare
   Challenge Type, Auto Renewal on. Save, then **reissue (↻)** to issue now; watch
   **Services → ACME Client → Log**.
6. **Assign to the GUI**: System → Settings → Administration → **SSL Certificate** → select
   `edge.teejlab.dev` → Save.
7. **Renewal automation** (critical): edit the cert → Automations → add the **restart/reconfigure
   web GUI** automation. Without it the cert renews but the GUI keeps serving the old one.
8. **DNS rebind**: accessing the GUI by hostname trips OPNsense's rebind protection. Add
   `edge.teejlab.dev` to System → Settings → Administration → **Alternate Hostnames** (keeps the
   protection on; don't disable the whole check).

## Lessons learned

- **DNS-01 is the only option from home** — no inbound for HTTP-01; DNS-01 just needs the
  Cloudflare API.
- **`.dev` makes valid certs mandatory** — HSTS preload = no click-through on a bad cert.
- **Cloudflare token method**: `CF_Token` + `CF_Account_ID` (+ `CF_Zone_ID`); leave the global
  key/email blank. A narrowly-scoped token may need the Zone ID supplied explicitly.
- **Proxmox**: account + plugin are cluster-shared; only the order repeats per node.
- **OPNsense has three easy-to-miss steps after issuance**: assign the cert to the GUI, add the
  renewal automation (or renewals don't apply), and whitelist the hostname for the rebind check.
- Access infra by **IP** to recover when a hostname/cert/rebind issue locks you out of the FQDN
  (IPs aren't `.dev`, so the browser lets you click through).
- **A lab that's switched off can't renew.** The OPNsense cert expired 2026-09-20 while the lab
  was off. The Proxmox/PBS certs survived only because they happened to renew before shutdown.
  After any long power-off, check every UI's cert first (`docs/runbooks/lab-cold-start.md`).
- **Click renew once.** The renew button gives no feedback, so it's easy to click twice. On
  2026-10-11 two runs overlapped: run 1 got the cert (`Cert success.`, files written), run 2
  failed finalising the same order (`orderNotReady … Order's status ("valid")`), and the store
  ended up still holding the *old* cert. Most likely run 2 wrote back a stale config over run
  1's import. Symptom: the log says "successfully renewed", but System → Trust still shows the
  old dates and the GUI serves the old cert even after `configctl webgui restart`.
- **Fix: re-import the cert acme.sh already has on disk.** No new issuance needed. Check the file,
  then run the plugin's import with the cert's **config UUID** (not the `id` used in the folder
  names; passing that crashes with an empty `LeAccount('')`):
  ```sh
  # from the PC; the VM console can't paste, and the root shell mangles pastes (0~ / 1~)
  ssh root@10.0.10.1 "openssl x509 -noout -dates -in /var/etc/acme-client/certs/<id>/cert.pem"
  ssh root@10.0.10.1 'grep -A3 "<certificate uuid=" /conf/config.xml'     # uuid + id + name
  ssh root@10.0.10.1 "/usr/local/opnsense/scripts/OPNsense/AcmeClient/lecert.php --mode import --cert <uuid>"
  ssh root@10.0.10.1 "configctl webgui restart"
  ```
  Plugin action names live in `/usr/local/opnsense/service/conf/actions.d/actions_acmeclient.conf`.
  Look them up instead of guessing.

## Not yet done (future)

Auto-TLS for **services you spin up** is a separate layer: a reverse proxy (Caddy now / Traefik
in the Docker phase) holding a `*.teejlab.dev` **wildcard** (also DNS-01), so any new
`name.teejlab.dev` gets TLS by adding a route/label — no per-service cert issuance. See
[ADR-0009](../decisions/0009-automatic-tls-strategy.md).
