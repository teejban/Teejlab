# Runbook: VM baseline setup (first-boot hardening for a fresh Ubuntu VM)

## Summary

Every new Linux VM in the lab gets the same baseline before it does any real work: key-only
SSH, automatic security patches, correct time, the Proxmox guest agent, and a small set of
base tools. This runbook is the hand-run checklist for that baseline, worked one chunk at a
time so each step is understood rather than pasted.

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

Status reflects `docker1` as of this writing. `[x]` done, `[ ]` pending, `→` next.

### 1. Non-root sudo user  `[x]`
Created by the Ubuntu installer. You SSH in as this user and use `sudo`; you never log in as
root. (If an installer didn't make one, create it before anything else while you still have
console access.)

### 2. SSH key auth + sshd hardening  `[x]`
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

### 3. System updates + reboot  `→ next`
`sudo apt update && sudo apt upgrade`, then reboot to clear the pending
"System restart required" from install.

### 4. Automatic security patches  `[ ]`
Install and enable `unattended-upgrades` so security updates apply themselves — set-and-forget,
so the box doesn't rot between hands-on sessions.

### 5. Time & locale  `[ ]`
Set the timezone (`timedatectl set-timezone …`); `systemd-timesyncd` is usually already syncing
time. Correct time matters for TLS, logs, and anything cluster-aware.

### 6. Proxmox integration  `[ ]`
Install and enable `qemu-guest-agent` so Proxmox sees the VM's IP, can do clean shutdowns, and
can quiesce for backups.

### 7. Base tooling  `[ ]`
`curl ca-certificates gnupg` and friends — the common prerequisites for adding third-party apt
repos (e.g. Docker's) later.

### 8. DNS / search domain  `[ ]`
Confirm the VM resolves `*.teejlab.dev` via the internal Unbound resolver (`10.0.10.1`) and has
the right search domain.

### Deferred — UFW / firewall caveat
Hold on host-level UFW until Docker is installed. Docker manipulates `iptables` directly and
bypasses UFW's rules in ways that surprise people; decide the host firewall story alongside the
lab-wide firewall hardening pass, not before.

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
