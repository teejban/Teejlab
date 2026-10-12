# Linux troubleshooting cheat sheet

## Summary

A short reference for "it's down / it's slow / it won't boot" on any Linux box in the lab. Each
section is a question and the command that answers it, with real output from the 2026-10-11
NAS recovery ([pi-nas-dr-rebuild.md](pi-nas-dr-rebuild.md)) as the worked example. The method
is the same every time: start at the lowest layer, find the first one that's broken, and dig
there.

## Network, from the outside in

| Question | Command / check | How to read it |
|---|---|---|
| Is there a physical link? | Port lights on both ends | No light means cable, port or power |
| Is the box on the network at all? | `ping`, then the ARP/neighbour table (`Get-NetNeighbor` / `ip neigh`) | A MAC without a reply is L3 config. No MAC at all means it isn't talking. |
| Is the switch passing the VLAN? | The switch's 802.1Q page | The port must be a member (tagged/untagged) |
| Is a service up? | TCP connect to the port | **Refused**: the host is up, nothing is listening. **Timed out**: the host is unreachable, or filtered. |
| What does the box itself have? | `ip -br a`, `ip route`, `networkctl list` | Which interfaces exist, their addresses, and the default route |

The refused-versus-timed-out distinction did a lot of work on 2026-10-11. All three NAS ports
*refused* after the rescue boot, which proved the kernel and network were up and the services
simply hadn't started yet.

## Boot problems

| Question | Command |
|---|---|
| Did boot finish cleanly? | `systemctl is-system-running` (`running` is good; `degraded` means something failed) |
| What failed? | `systemctl --failed` |
| Is boot still waiting on something? | `systemctl list-jobs` |
| Where did the time go? | `systemd-analyze critical-chain` or `systemd-analyze blame` |
| Why did a unit fail? | `journalctl -b -u <unit>` (`-b -1` for the previous boot) |

**Read `critical-chain` for gaps.** Each line has an `@` start time. A big jump between two
lines means something was waiting:

```
multi-user.target @2min 9.575s
└─getty.target @2min 9.575s
  └─openmediavault-issue.service @2min 3.004s +6.570s
    └─network.target @6.449s
```

6 s to 2 min 3 s, and two minutes is a common systemd default timeout. Combined with
`systemctl --failed` showing `systemd-networkd-wait-online`, that's the whole diagnosis. Then
search the exact unit name plus the symptom ("systemd-networkd-wait-online slow boot"). It's a
well-known issue with a standard fix (`--any`).

## Disks and ZFS

| Question | Command |
|---|---|
| Are the drives visible? | `lsblk -o NAME,SIZE,MODEL` |
| Are they healthy? | `smartctl -H /dev/sdX` (`PASSED`), `dmesg \| grep -iE 'ata[0-9]\|error\|reset'` |
| Which pools exist, without importing them? | `zpool import` (read-only listing) |
| Is a pool healthy? | `zpool status <pool>` (`ONLINE`, `No known data errors`) |
| Is it mounted? | `zfs mount`, `findmnt <path>` |

## Services and exports

| Question | Command |
|---|---|
| Is the service running? | `systemctl status <svc>` |
| What's NFS actually exporting? | `exportfs -v` |
| What's Samba actually sharing? | `testparm -s` |
| What config does OMV think it has? | `omv-confdbadm read conf.<path>`, e.g. `conf.system.network.interface` |
| Which OMV API methods exist? | `grep -n registerMethod /usr/share/openmediavault/engined/rpc/<svc>.inc` |

## Habits that pay off

- **Copy exact error text and unit names into a search.** Most problems have been hit before.
- **Prefer read-only commands first** (`zpool import` with no name, `fsck -n`, `mount -o ro`,
  imaging a card before inspecting it). Make sure you understand the state before you change it.
- **Compare a working setup with a broken one.** Swapping a known-good card into the same
  hardware isolated hardware from software in minutes.
- **Don't run a command that could cut your connection from inside that connection.** Use
  `systemd-run`, a second path (another host on the same subnet), or a console.
