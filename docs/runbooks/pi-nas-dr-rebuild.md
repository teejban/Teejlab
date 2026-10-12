# Incident + runbook: Pi NAS didn't survive a cold start — DR rebuild onto a new card

## Summary

On 2026-10-11 the lab was powered back on after roughly six weeks off (the Pi's
`fake-hwclock` last saved 2026-08-28). Everything came back except the Pi NAS (`nas`,
`10.0.10.4`): it powered up, showed link lights, and read its drives, but never put an IP
address on the network. The old SD card turned out to be physically healthy and its config
intact, but it still wouldn't boot to a working state, and with no monitor and its logs kept in
RAM there was no way to see why. So I treated it as the DR drill I'd been meaning to write: a
fresh OS on a spare high-endurance card, OpenMediaVault 8, force-import the existing ZFS pool,
and rebuild the config. **No data was lost**: the raidz1 pool imported with all four drives
`ONLINE`.

The rebuild was done almost entirely through OMV's own RPC API (`omv-rpc`) instead of the web
UI, which turned up a handful of OMV 8 quirks worth keeping. Two problems were self-inflicted
along the way: an apply that wiped the network (the new OS calls the NIC `end0`, not `eth0`),
and a two-minute boot stall caused by `systemd-networkd-wait-online`. Both are written up below.
Total time from "Pi won't come up" to "cluster back at 3/3 votes" was about four hours, much of
it spent on theories that turned out wrong.

## What the cold start looked like

The whole lab was switched on at once. From my PC (on Tailscale), OPNsense, both nodes and
internal DNS all answered. Three things didn't:

| Target | Result |
|---|---|
| Pi NAS `10.0.10.4` | No ping, no SSH, no web UI |
| PBS `10.0.10.6` | Down |
| docker1 `10.0.20.2` | Down |
| OPNsense cert | **Expired 2026-09-20**. The lab was off when it was due to renew. |

**PBS and docker1** were a side issue. Proxmox refused to start them because each still had
its install ISO attached from `teejlab-share`, the Pi's CIFS share. An ISO you'll never boot
from again still makes the VM depend on the NAS. Fix: Hardware → CD/DVD Drive → *Do not use
any media*. The PBS build command in [pbs-rebuild.md](pbs-rebuild.md) attached
`teejlab-share:iso/...` and nothing ever detached it.

## Narrowing down the Pi

| Layer | Check | Result |
|---|---|---|
| Power | Penta SATA HAT on a 12V barrel jack (it powers the Pi through the GPIO header) | Fine |
| Link | Pi and switch Port 2 link lights; switch showed traffic | Up |
| Switch | 802.1Q page: VLAN 10 members `2-3,5`, tagged `2-3,5`; VLAN 1 untagged `1-5,7-8` | Port 2 exactly as documented, and it survived the power cut |
| L3 | Ping sweep of `192.168.8.0/24` plus ARP table, looking for Pi OUIs (`2C-CF-67`, `D8-3A-DD`) | Pi MAC nowhere. Found the switch at **`192.168.8.156`**, which hadn't been written down anywhere. |

Not seeing the Pi on the flat network was expected, since its flat IP was removed back in June.
Not seeing it on VLAN 10 meant it either wasn't finishing boot or its network wasn't coming up.
Without a micro-HDMI cable there was no console, so the next step was to read the card.

## Reading the old card on Windows

`wsl --mount` refuses removable media:

```
Failed to attach disk '\\.\PHYSICALDRIVE6' to WSL2: The system cannot find the drive specified.
Error code: Wsl/Service/AttachDisk/MountDisk/HCS/0x8007000f
```

So I imaged it instead. An elevated PowerShell read of `\\.\PHYSICALDRIVE6` copied it into a
`.img` file, and WSL loop-mounted the image read-only (`losetup -r -P`, `mount -o ro,noload`).
The card never got written to. Side effect: the failed `wsl --mount` left the disk **Offline** in
Windows, which `Set-Disk -IsOffline $false` (admin) undoes.

What the card showed:

- **Hardware:** `31914983424` bytes read, **0 read errors**.
- **Filesystem:** `e2fsck -fn` was clean apart from the journal replay you'd expect after an
  unclean power-off. The FAT boot partition had only its dirty bit set.
- **Network config:** intact and correct (`eth0.10` → `10.0.10.4/24`, gateway `.1`, `eth0` IPv4
  disabled).
- **ZFS:** `zfs-dkms` 2.3.2 was built for the running kernel `6.12.93`. No half-finished
  kernel update.
- **Logs:** none worth reading. `openmediavault-flashmemory` (folder2ram) keeps `/var/log` in
  RAM, so the failed boots left nothing on the card. The setting that protects the SD card also
  removed the evidence.
- **`config.txt`** had `dtparam=pciex1_gen=3`, which forces the HAT's PCIe link to Gen 3. That's
  unsupported on a Pi 5. I commented it out as a test, and **the card still didn't boot**, so it
  wasn't the cause. I left it at Gen 2 anyway, because the Pi's 1 GbE port (~115 MB/s) is the
  bottleneck long before Gen 2 PCIe (~400–450 MB/s) is.

**The root cause on the old card was never confirmed.** Hardware, filesystem and config all
checked out. With the old card in, the HAT's SATA lights flashed all together every so often and
nothing came up. The new card, with the same HAT and drives, booted fine, so it's something
in that OS install. I stopped digging: the image is archived, and the rebuild was faster than
continuing to guess blind.

## The rebuild

**Base:** Raspberry Pi Imager → Raspberry Pi OS Lite (64-bit, Debian trixie), with SSH enabled
in the customisation settings, written to the spare high-endurance card. It booted with DHCP on
the untagged flat network, then the OMV install script set up **OMV 8**.

**Drive health first, before ZFS touched anything:**

```
== /dev/sda
SMART overall-health self-assessment test result: PASSED
... (sdb, sdc, sdd identical)
```

**ZFS:** `apt install openmediavault-zfs` (it builds the DKMS module, so give it time). Then list
the pool **without** importing it:

```
  pool: teejlab-tank
 state: ONLINE
status: The pool was last accessed by another system.
        teejlab-tank                                ONLINE
          raidz1-0                                  ONLINE
            ata-T-FORCE_512GB_TPBF2502130030100167  ONLINE
            ata-T-FORCE_512GB_TPBF2502130030100092  ONLINE
            ata-T-FORCE_512GB_TPBF2502130030100137  ONLINE
            ata-T-FORCE_512GB_TPBF2502130030100113  ONLINE
```

"Last accessed by another system" is expected after an OS reinstall, so `zpool import -f
teejlab-tank`. Result: `1.86T  43.0G alloc  ONLINE`. **Don't accept a pool upgrade** while
there's any chance of going back to the old OS.

**Config through the OMV API.** Every value came from the old card's
`/etc/openmediavault/config.xml`, so nothing was rebuilt from memory. The pattern for settings
objects is to read, change one field, and write back, which avoids having to know every field OMV
expects:

```sh
sudo omv-rpc -u admin 'SMB' 'setSettings' "$(sudo omv-rpc -u admin 'SMB' 'getSettings' \
  | python3 -c 'import sys,json; d=json.load(sys.stdin); d["enable"]=True; print(json.dumps(d))')"
```

New objects use OMV's "new object" UUID, `fa4b1c66-ef79-11e5-87a0-0002b3a176b4`, and OMV
replaces it with a real one when it saves.

| Step | Call | Notes |
|---|---|---|
| Register the pool with OMV | `Zfs doDiscover {}` | Importing at the CLI isn't enough. OMV's filesystem list stays `[]` until you discover. |
| Shared folders | `ShareMgmt set` ×2 | `reldirpath` set to the existing dirs (`teejlab-share/`, `teejlab-backups/`), so it points at the data rather than creating new folders. |
| NFS export for PBS | `NFS setShare` | Needs **`mntentref`** set to the new-object UUID, so OMV creates the `/export/...` bind mount. Client `10.0.10.6`, options `rw` + `subtree_check,insecure,no_root_squash,crossmnt`. |
| SMB | `SMB setSettings` + `SMB setShare` ×2 | No guest access, writable, browseable. |
| Hostname / timezone | `Network setGeneralSettings`, `System setTimeSettings` | `nas` / `teejlab.dev`, `America/Los_Angeles`. |
| Apply | `Config applyChanges {"modules":[],"force":false}` | This is the yellow banner in the UI. |

The old export allowed `10.0.10.6/24`, which really means the whole subnet. The new one allows
only `10.0.10.6`. The old VLAN interface also used the Pi itself (`10.0.10.4`) as its DNS server.
The new one uses OPNsense (`10.0.10.1`).

**Find method names, don't guess them.** I guessed `NFS set`, `Zfs` and `SSH getSettings` and
was wrong each time. The real names are one grep away:

```sh
grep -n 'registerMethod' /usr/share/openmediavault/engined/rpc/nfs.inc
```

Schema errors list missing fields one at a time (`Missing 'required' attribute 'devicename'`,
then `'altmacaddress'`), so expect a few round trips.

## Self-inflicted outage #1: the apply that wiped the network

After setting the hostname I ran `applyChanges`, and SSH dropped. The Pi was gone from both
networks. OMV's interface list showed why:

```
"devicename":"eth0","method":"dhcp"
```

Raspberry Pi OS trixie names the port **`end0`**. The old card had a `.link` file forcing `eth0`.
The OMV install script created an interface entry for `eth0`, so on apply OMV wrote a netplan
file for a device that doesn't exist and removed the one that had been configuring `end0`. The
real port ended up with no address.

**Rescue without a console:** the kernel can configure an interface itself, before userspace
starts. With the card in the PC, I appended this to `/boot/firmware/cmdline.txt` (FAT, editable
from Windows, still one line, original saved as `cmdline.txt.bak`):

```
ip=192.168.8.230::192.168.8.1:255.255.255.0:nas:eth0:off
```

It booted, and port 22 returned *connection refused*. That's a good sign: refused means the box
answered. It took a couple more minutes (see #2) before SSH was actually up.

**Permanent fix:** retarget OMV's entry to `end0` with the **same** static address it already
had, so the next apply wouldn't change the address under me:

```sh
# read → filter to fields setEthernetIface accepts → devicename=end0, static .230 → write
sudo omv-rpc -u admin 'Network' 'setEthernetIface' "$(...)"
```

After that, applies were safe and OMV wrote `20-openmediavault-end0.yaml`.

## Getting onto VLAN 10 without locking myself out

Same sequence as the hosts in June: add, verify from inside the VLAN, then cut over.

1. `Network setVlanIface`: `end0.10`, `10.0.10.4/24`, **no gateway** yet, DNS `10.0.10.1`. It needs
   `devicename` and `altmacaddress` even though the UI never asks for them.
2. Apply. **My SSH session dropped mid-apply and killed it**, so `end0.10` never got created.
   I re-ran it detached from the session:
   ```sh
   sudo systemd-run --unit=omv-net-apply omv-salt deploy run systemd-networkd
   ```
3. Verified **from teejhost1** (same subnet, so no gateway needed): ping plus `ssh
   teejadmin@10.0.10.4`. Testing from my PC wouldn't have worked yet, because without a gateway
   the Pi can't send replies back to the Tailscale range.
4. From that node session: gateway `10.0.10.1` onto `end0.10`, `end0` → `method: manual` (no
   IPv4), restore the original `cmdline.txt`, apply, reboot.

End state: `end0.10@end0 UP 10.0.10.4/24`, default route via `10.0.10.1`, flat address gone,
`nas.teejlab.dev` resolving, and SSH, web, SMB and NFS reachable from my PC over Tailscale.

## Self-inflicted outage #2: the two-minute boot

After the reboot, SSH was up but the web UI, SMB and NFS weren't, and `systemctl
is-system-running` said `degraded`:

```
● systemd-networkd-wait-online.service loaded failed failed Wait for Network to be Configured

multi-user.target @2min 9.575s
└─getty.target @2min 9.575s
  └─openmediavault-issue.service @2min 3.004s +6.570s
    └─network.target @6.449s
```

The gap is the clue: network up at 6 s, nothing until 2 min 3 s. `wait-online` waits for every
managed link to be online, and `end0` deliberately has no IPv4 now. It's only the carrier for
VLAN 10, so it never qualifies. The service sat out its 2-minute timeout and failed, and
everything ordered after `network-online.target` (nginx, smbd, NFS) started late. Fix (a systemd
drop-in, not OMV config, so OMV leaves it alone):

```ini
# /etc/systemd/system/systemd-networkd-wait-online.service.d/override.conf
[Service]
ExecStart=
ExecStart=/usr/lib/systemd/systemd-networkd-wait-online --any
```

This probably explains the "it's just taking a while" moments during the rescue boots too.

## QDevice

The cluster ran on 2 of 3 votes throughout. With a dead tie-breaker, losing a node would have
cost quorum, so neither node got rebooted until this was back.

```sh
# Pi
sudo apt install -y corosync-qnetd        # generates a new CA and certs
# add teejhost1 and teejhost2 root public keys to /root/.ssh/authorized_keys

# teejhost1
pvecm qdevice remove                      # old link has certs the new qnetd doesn't know
pvecm qdevice setup 10.0.10.4
```

```
Quorate:          Yes
Expected votes:   3
Total votes:      3
Flags:            Quorate Qdevice
```

Quorum held at every step: 2/3 before, 2/2 after `remove`, 3/3 after `setup`. VMs never noticed.

## Reconnecting Proxmox storage

Nothing needed removing and re-adding. The storage entries point at the same IP and share
names, so they only needed their other end to work again:

```
teejlab-pbs: error fetching datastores - 500 Can't connect to pbs.teejlab.dev:8007 (No route to host)
connection check for storage 'teejlab-share' failed - session setup failed: NT_STATUS_LOGON_FAILURE
```

- **`teejlab-share` (CIFS): `NT_STATUS_LOGON_FAILURE`.** The rebuilt NAS had no SMB password
  for `teejadmin`. Samba keeps its own password database, separate from the Linux login, and
  it doesn't survive an OS reinstall. Fix: read the password Proxmox already stores
  (`/etc/pve/priv/storage/teejlab-share.pw`) and set the same one on the NAS with
  `sudo smbpasswd -a teejadmin`. It went `active` within seconds.
- **`teejlab-pbs`: `No route to host`.** On the same subnet, that means nothing was answering at
  `10.0.10.6`: the PBS VM wasn't running. Clear the leftover ISO and start it
  (`qm set 101 --ide2 none,media=cdrom`, `qm start 101`). PBS remounted its NFS datastore
  from the NAS on boot and went `active`.

The nodes don't use NFS at all. Only PBS does, so restricting the export to `10.0.10.6` was
safe.

## What tripped me up / what I'd do differently

- **I had no console.** Every hour of this was me inferring state from the outside: link
  lights, ARP tables, "refused" versus "timed out". A micro-HDMI cable or a Raspberry Pi Debug
  Probe on the UART would have shown where the old card stopped in the first five minutes. That
  goes on the shopping list.
- **flashmemory kept the logs in RAM, so the failed boots left nothing.** On a box whose
  failure mode is "doesn't boot," make `/var/log/journal` persistent, or ship logs off-box.
  Lower SD wear isn't worth losing the logs.
- **I chased physical-layer theories too long.** PCIe Gen 3, the ribbon cable, the HAT, the SD
  card: none of them were it. The useful comparison was swapping cards with the same hardware.
  The new card booting fine pointed at the OS within minutes, and I should have run that test
  first.
- **Check what OMV thinks the network is before the first apply.** Read
  `conf.system.network.interface` on any fresh OMV install. If `devicename` doesn't match
  `ip -br a`, the first apply will take the box offline.
- **Run OMV applies detached** (`systemd-run`), or from a session that won't be affected by the
  change. An apply running inside the SSH session it's reconfiguring can kill itself halfway.
- **Install ISOs don't belong on the NAS.** Detach the CD after installing, or keep ISOs on
  node-local storage. Otherwise every VM quietly depends on the NAS being up.
- **Write down break-glass addresses.** The switch's management IP (`192.168.8.156`) took a
  ping sweep to find. It's in the hardware inventory now.
- **The `cmdline.txt` `ip=` trick is the cheapest console substitute there is.** It works even
  when the OS's network config is completely broken, and all it needs is a card reader.

## Follow-ups

- [ ] Reboot test confirming the `wait-online` fix (services up in seconds, `running` not `degraded`)
- [ ] Tighten Pi root SSH to key-only (`sshd -T` shows `permitrootlogin yes`; root has no password, so it's key-only in practice, but make it explicit)
- [x] PBS datastore and Proxmox `teejlab-share` CIFS storage back online (SMB password re-set, PBS VM started)
- [x] OPNsense cert renewed (expired 2026-09-20 while the lab was off; a double-click race lost the import. See the TLS runbook's lessons.)
- [ ] Enable/verify `openmediavault-writecache` (flashmemory's OMV 8 successor), but make the journal persistent
- [ ] OMV config backup on a schedule plus a pre-flashed spare card, so the next recovery is a restore, not a rebuild
- [ ] Console access: micro-HDMI cable or Debug Probe
