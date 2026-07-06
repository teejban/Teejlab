# Runbook: VM baseline setup (first-boot hardening for a fresh Ubuntu VM)

## Summary

Every new Linux VM in the lab gets the same baseline before it does any real work. The baseline
is defined by what a VM must be able to do to be a first-class citizen of the environment,
organised into three pillars:

1. **Communicate** with the rest of the environment — networking, DNS, and a shared clock.
2. **Report** stats and metrics — so the box is observable, never a blind spot.
3. **Be administered securely** — key-only SSH, hardening, automatic patching, clean lifecycle.

This runbook is the hand-run checklist for that baseline, worked one chunk at a time so each
step is understood rather than pasted.

The end goal is to *stop* doing this by hand. Once the steps are proven manually, they become
a startup script that runs after install — evolving **bash → cloud-init → Ansible**. Doing it
manually first is deliberate: you can't automate a process you haven't debugged, and the
checklist below is the spec that automation has to reproduce.

First subject: **`docker1`**, the first Docker host (Ubuntu Server). The Ubuntu installer
already created a non-root sudo user at install, so the very first checklist item was done
before this runbook started.

## The order matters

SSH key auth comes **before** disabling passwords, and disabling passwords comes before
anything else risky. The rule: never turn off a login method until the replacement is proven.
Prove key login works, *then* slam the password door. If you harden first and the key is
wrong, you've locked yourself out of a box whose only other door you just welded shut. (Console
access via the Proxmox UI is the safety net throughout — keep a session open.)

## Checklist

Items are listed in **execution order** (which matters — see above), each tagged with the
pillar it serves: **[Comms]**, **[Metrics]**, **[Admin]**. Status reflects `docker1` as of this
writing. `[x]` done, `[ ]` pending, `→` next.

### 1. Non-root sudo user  · [Admin]  `[x]`
Created by the Ubuntu installer. You SSH in as this user and use `sudo`; you never log in as
root. (If an installer didn't make one, create it before anything else while you still have
console access.)

### 2. SSH key auth + sshd hardening  · [Admin]  `[x]`
The big one, and the most important security step on any box. Broken into chunks:

- **Chunk 0 — orientation.** Confirm what the installer left you (user exists, SSH reachable).
- **Chunk 1 — confirm SSH is running** on docker1.
- **Chunk 2A — key on Windows.** `ssh-keygen -t ed25519 -C "teejadmin@docker1"` in Windows
  PowerShell. Saves to `C:\Users\TJ\.ssh\`. The private key never leaves this machine; only
  the `.pub` half travels.
- **Chunk 2B — public key onto docker1.** Append the `.pub` line to
  `~/.ssh/authorized_keys` for your sudo user. Prove key login works *before* the next step.
- **Chunk 3 — harden sshd** via a drop-in, `/etc/ssh/sshd_config.d/01-hardening.conf`. The
  `01` prefix matters: sshd drop-ins are **first-wins**, so this has to sort before the
  installer's `50-cloud-init.conf`. Set `PermitRootLogin no`, `PasswordAuthentication no`,
  `PubkeyAuthentication yes`. The success signal is a *refusal*: a password-only login now
  gets `Permission denied (publickey)` — the door is shut.
- **Quick win — `~/.ssh/config` entry** so `ssh docker1` just works (no IP to remember, no
  password prompt). This is the payoff for doing it the DevOps way — the same config skill
  transfers to Linux, Mac, and every CI runner.

### 3. System updates + reboot  · [Admin]  `[x]`
`sudo apt update && sudo apt upgrade`, then reboot to clear the pending
"System restart required" from install.

### 4. Automatic security patches  · [Admin]  `[x]`
Install and enable `unattended-upgrades` so security updates apply themselves — set-and-forget,
so the box doesn't rot between hands-on sessions. On docker1: enabled via
`dpkg-reconfigure`, `20auto-upgrades` has both periodic knobs at `"1"`, and auto-reboot is on
at 03:00 (`Automatic-Reboot "true"` + `Automatic-Reboot-Time "03:00"` in `50unattended-upgrades`)
so kernel patches land unattended off-hours. Verified: the `--dry-run --debug` Allowed-origins
list included `jammy-security` + the ESM security pockets, with `-updates`/`-backports` pinned
out — security-only, as intended.

### 5. Time & locale  · [Comms]  `[x]`
Set the timezone (`timedatectl set-timezone …`); `systemd-timesyncd` is usually already syncing
time. Correct time matters for TLS, logs, and anything cluster-aware. On docker1: left on
**UTC** deliberately (fleet-wide log correlation, no DST) — `System clock synchronized: yes`,
`NTP service: active`, `RTC in local TZ: no`. No change needed; it was already correct.

### 6. Proxmox integration  · [Admin]  `[x]`
Install and enable `qemu-guest-agent` so Proxmox sees the VM's IP, can do clean shutdowns, and
can quiesce for backups. **Two halves:** the package inside the guest
(`apt install qemu-guest-agent` + `systemctl enable --now`) *and* the VM Options → QEMU Guest
Agent → Enabled toggle in Proxmox — the latter needs a full **power cycle from the Proxmox UI**
(not a guest-side `reboot`) to attach the virtio-serial channel. Verified on docker1: the
Summary tab's IPs field now shows `10.0.20.2`.

### 7. Base tooling  · [Admin]  `[x]`
Two tiers. **Load-bearing:** `ca-certificates curl gnupg` — the prerequisites for adding
third-party apt repos over HTTPS (Docker's key + repo come next), so front-loaded here.
**Quality-of-life (in the baseline):** `git vim htop` — clone configs, edit, watch load. Kept
deliberately lean; add one-off tools per-host when you actually reach for them, so the baseline
doesn't become a dumping ground.

### 8. Metrics agent  · [Metrics]  `[x]`
Install `prometheus-node-exporter` — exposes host metrics (CPU/RAM/disk/net) on `:9100` for
Prometheus to scrape. Safe to bake in **before** Prometheus exists because it's **pull-model**:
no backend to configure, it just exposes and waits. Every VM is scrape-ready the day the
Prometheus server lands — zero retrofit. Install as a **system package** (not a container) so
the baseline never assumes Docker. On docker1: installed, `active (running)`, and
`curl -s localhost:9100/metrics` returns the metrics payload.

### 9. DNS / search domain  · [Comms]  `[x]`
Confirm the VM resolves `*.teejlab.dev` via the internal Unbound resolver (`10.0.10.1`) and has
the right search domain. On docker1: DNS resolves via `10.0.10.1`. Note this is a **cross-VLAN**
lookup (docker1 on VLAN 20 → resolver on VLAN 10), so a working result also proves inter-VLAN
routing SERVERS→MGMT is open — a flow the future default-deny hardening pass must explicitly
re-allow (`SERVERS → 10.0.10.1:53`), alongside the node_exporter `:9100` allow.

### Reserved — log shipper  · [Metrics]
The logs counterpart to node_exporter: **Grafana Alloy** (or promtail) shipping journald/app
logs to Loki. Deliberately **not** in the baseline yet — it's **push-model**, so it needs a
Loki destination that doesn't exist. Add to the baseline the day Loki comes online. Noted here
so it isn't forgotten.

### Deferred — UFW / firewall caveat  · [Admin]
Hold on host-level UFW until Docker is installed. Docker manipulates `iptables` directly and
bypasses UFW's rules in ways that surprise people; decide the host firewall story alongside the
lab-wide firewall hardening pass, not before. (`fail2ban` was considered and skipped: password
auth is already off and the box isn't internet-facing, so it guards logins that don't exist.)

## What's *not* in the baseline (the role layer)

The baseline is the **universal** layer — everything here applies to *every* VM regardless of
what it's for. Anything purpose-specific lives one layer up, in a **role** that runs *after*
the baseline. The dividing line is a single test: *would every VM need this?* If no, it's a
role, not baseline.

The clearest example is **third-party apt repositories**. The baseline installs the repo
*prerequisites* (`ca-certificates curl gnupg`, item 7) but adds **no** actual third-party repo,
because adding one asserts a purpose. Docker's GPG key + `docker.list` + the engine belong to a
**Docker-host role**, not here — see [docker-host-setup.md](docker-host-setup.md).

| Layer | Scope | Contents | Applies to |
|-------|-------|----------|------------|
| **Baseline** (this doc) | universal | comms, metrics, secure admin, repo *prereqs* | every VM |
| **Role** (separate doc) | purpose-specific | third-party repos, app packages, service config | matching hosts only |

This split is deliberate and forward-looking: it's exactly the **Ansible** shape the automation
lands on — a `base` role applied to all hosts, plus a `docker` role (and later `k3s`, etc.)
applied only where relevant. Building it layered by hand now means the automation is a clean
mapping, not a rewrite.

## The automation ladder (the actual goal)

Once the checklist is proven by hand on docker1, it gets automated in three stages, each a real
DevOps skill:

1. **bash** — a single `first-boot.sh` that runs the checklist top to bottom. Teaches the
   mechanics: what each command does, what order, what's idempotent.
2. **cloud-init** — the same steps declared in a cloud-init config baked into a VM template.
   Clone the template and it injects the user, SSH key, packages, and a `runcmd` baseline at
   first boot, automatically. This is the Proxmox-native path.
3. **Ansible** — the checklist as a role, run against any number of hosts, idempotent and in
   version control. The end state, and what reproduces this whole thing from scratch.

## Lessons / what tripped us up

- **Prove before you disable.** The single rule that keeps you from locking yourself out. Key
  login proven → *then* passwords off. Keep a Proxmox console session open as the net.
- **sshd drop-ins are first-wins, not last-wins.** The `01-` prefix on the hardening file is
  load-bearing — without it, the installer's `50-cloud-init.conf` can win and your hardening
  silently doesn't apply. Verify with `sudo sshd -T | grep -E 'permitrootlogin|passwordauth'`.
- **The refusal is the success.** `Permission denied (publickey)` on a password attempt means
  hardening worked. A scary-looking error is the intended outcome here.
- **Do it by hand once.** The point of the manual pass isn't the box — it's the spec. Every
  gotcha found here (the drop-in ordering, the UFW/Docker interaction) is one the automation
  would otherwise reproduce as a silent bug.
- **`unattended-upgrades` "active (running)" ≠ actually patching.** The running service is
  `unattended-upgrade-shutdown --wait-for-signal` — a shutdown-delay helper that idles 24/7. The
  real switch is `/etc/apt/apt.conf.d/20auto-upgrades` (both periodic knobs `"1"`) plus the
  `apt-daily*` timers. Verify the config, not the service status. Prove scope with
  `unattended-upgrade --dry-run --debug` and read the *Allowed origins* line.
- **The `//` is load-bearing.** Options in `50unattended-upgrades` (auto-reboot, etc.) ship
  commented out; leaving the `//` on is the #1 way a change silently no-ops. Confirm the value
  actually took with `apt-config dump | grep -i <option>`, which reads it apt's way — not just
  eyeballing the file.
- **"Running on the development release"** in the dry-run output is a cosmetic/misleading line,
  not an error — the Allowed-origins list below it (correctly naming `jammy-security`) is what
  proves the release was identified right. (docker1 is Ubuntu **22.04 jammy**.)
- **Pull-model agents can go in the baseline before their backend exists; push-model can't.**
  node_exporter just exposes `:9100` and waits to be scraped — nothing to point at, so it's
  scrape-ready the day Prometheus lands. A log shipper (Alloy/promtail) *pushes* to Loki, so it
  needs a live destination and can't be baked in until Loki exists. This distinction is what
  decides *when* a monitoring agent joins the baseline.
- **node_exporter binds `:9100` on all interfaces** (necessary — Prometheus scrapes it across
  VLANs). Note it for the firewall hardening pass: allow *Prometheus host → `:9100`*
  specifically rather than leaving the port open lab-wide.
