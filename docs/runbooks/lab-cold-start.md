# Runbook: Cold-starting the lab, and break-glass access

## Summary

How to bring the whole lab up from fully off, what to check afterwards, and how to reach each
device when its normal path is broken. Written after the 2026-10-11 cold start
([pi-nas-dr-rebuild.md](pi-nas-dr-rebuild.md)), where the repo explained how everything was
built but not how to operate it once it was down.

## Power-on order

Turning everything on at once doesn't damage anything, but some things look broken for a few
minutes while they wait on each other. The order that avoids that:

1. **Switch.** Everything goes through it.
2. **Pi NAS.** It's the cluster's third vote, holds PBS's datastore (NFS), and serves the
   `teejlab-share` ISO/CIFS share. It's the slowest to boot.
3. **teejhost2.** It runs OPNsense, so routing, DHCP, DNS and Tailscale come up with it. Give it
   a minute or two before expecting names to resolve.
4. **teejhost1**, then the VMs on it (PBS, docker1).

**Dependencies to keep in mind:**

- Nothing has an IP or DNS until OPNsense is up.
- PBS's datastore won't mount until the Pi is serving NFS.
- **Any VM with an ISO attached from `teejlab-share` won't start while the Pi is down.**
  Detach install ISOs (CD/DVD → *Do not use any media*).
- With the Pi down the cluster is at 2/3 votes. It's still quorate, but **don't reboot a node**
  until the QDevice is back.

## Post-boot health check

Work from the bottom up. The first layer that fails is the one to dig into.

| Check | How | Good |
|---|---|---|
| Quorum | `pvecm status` on a node | `Quorate: Yes`, `Total votes: 3` |
| Routing | From a node: `ping 10.0.10.1`, `ping 1.1.1.1` | Both answer |
| DNS | `nslookup pbs.teejlab.dev` | `10.0.10.6` |
| Remote access | Tailscale on, open `https://teejhost1.teejlab.dev:8006` | Loads |
| Certs | Check the padlock on all four infra UIs | Valid. If one expired while the lab was off, renew it manually. |
| NAS services | `nas.teejlab.dev` on ports 22, 80, 445, 2049 | All open |
| Backups | PBS → Datastore `teejlab-backups` | Online |
| VMs | Proxmox UI | PBS and docker1 running ("Start at boot" set) |

After a long break, apply updates **one box at a time**. Updating teejhost2 can mean a reboot,
and that takes OPNsense, and with it the whole lab, down briefly.

## Addresses and break-glass access

| Device | Normal access | Break-glass |
|---|---|---|
| OPNsense `edge` | `https://edge.teejlab.dev` (`10.0.10.1`) | `https://10.0.10.1` (an IP isn't HSTS-pinned, so you can click past an expired cert); console via teejhost2 → VM 100 |
| teejhost1 | `https://teejhost1.teejlab.dev:8006` (`10.0.10.2`) | Local console |
| teejhost2 | `https://teejhost2.teejlab.dev:8006` (`10.0.10.3`) | Local console |
| NAS `nas` | `http://10.0.10.4` (web), SSH `teejadmin@10.0.10.4` | `ip=` rescue below; the web UI needs the IP, since `.dev` forces HTTPS and the NAS has no cert |
| PBS | `https://pbs.teejlab.dev:8007` (`10.0.10.6`) | Console via teejhost1 → VM 101 |
| Switch (TL-SG108E) | `http://192.168.8.156` | **Only reachable from VLAN 1** (the travel router's network, e.g. its Wi-Fi), not over Tailscale. Factory default is `192.168.0.1`. |
| Travel router | `http://192.168.8.1` | Its client list is the quickest way to find anything on the flat network |

## NAS rescue: give the network port an address from the boot partition

Use this when the NAS boots but has no usable network, for example after an OMV network apply
went wrong. It needs no monitor, just a card reader.

1. Power the Pi off (short press of the button, or unplug if it's unresponsive). Put the SD card
   in any PC. If Windows offers to format it, **cancel**.
2. On the FAT `bootfs` partition, back up `cmdline.txt`, then append this to the **end of the
   single line**:
   ```
   ip=192.168.8.230::192.168.8.1:255.255.255.0:nas:eth0:off
   ```
   The kernel brings the port up with that address before any of the OS's network config runs.
   It uses the kernel's own name `eth0`, even though userspace renames it `end0`. `.230` is free
   on the flat network.
3. Boot, wait a few minutes, then `ssh teejadmin@192.168.8.230`. *Connection refused* means the
   kernel is up and services are still starting. Wait.
4. Fix the real config (OMV: `sudo omv-confdbadm read conf.system.network.interface`; the
   `devicename` must match `ip -br a`), apply, then **restore the original `cmdline.txt`**.

## Reading a Pi SD card from Windows

`wsl --mount` won't attach removable media (`0x8007000f`). Instead, image the card with an
elevated raw read of `\\.\PHYSICALDRIVE<N>` to a `.img` file, then in WSL:

```sh
L=$(sudo losetup -r -f -P --show /mnt/c/path/old-pi-sd.img)
sudo mount -o ro,noload ${L}p2 /mnt/oldpi       # read-only, no journal replay
```

Check the disk number in Windows (`Get-Disk`) before reading anything. There are several disks
on the dev box. If a failed `wsl --mount` leaves the card marked Offline, `Set-Disk -Number <N>
-IsOffline $false` (admin) brings it back.
