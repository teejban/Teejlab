# Runbook: Rebuild Proxmox Backup Server (VLAN-10-native, datastore on the Pi over NFS)

## Summary

The PBS VM was rebuilt from scratch directly on MGMT VLAN 10 (rather than migrating the old
flat-net one), with its datastore on the Pi's ZFS pool via NFS. End state: VM `pbs`
(VMID 101 on teejhost1), hostname `pbs`, IP `10.0.10.6`, datastore `teejlab-backups` backed by
`10.0.10.4:/export/teejlab-backups`. This was the last flat-net dependency in the lab.

The VM creation is scripted with `qm`; only the OS install itself is interactive (the PBS ISO
installer runs over the console). Everything else is CLI.

> Note: deleting/rebuilding the PBS *VM* does not touch backups when the datastore lives on
> external storage (the Pi) — the VM is just the OS. The datastore (chunks) persists
> independently. (In this rebuild the old datastore turned out empty, so nothing to re-adopt.)

## Create the VM (on teejhost1)

```sh
# remove the old/broken VM if present (datastore is on the Pi — nothing lost)
qm stop 101 2>/dev/null ; qm destroy 101 --purge 2>/dev/null

qm create 101 \
  --name pbs \
  --ostype l26 \
  --machine q35 \
  --bios seabios \
  --cpu host \
  --cores 2 \
  --memory 2048 \
  --scsihw virtio-scsi-single \
  --scsi0 local-lvm:32,discard=on,iothread=1,ssd=1 \
  --net0 virtio,bridge=vmbr0,tag=10 \
  --agent enabled=1 \
  --ide2 teejlab-share:iso/proxmox-backup-server_3.4-1.iso,media=cdrom \
  --boot order='scsi0;ide2'

qm start 101
```

The **`tag=10`** on the NIC is what makes it VLAN-10-native. Empty `scsi0` falls through to the
CD, so it boots the installer.

## Install (console)

Open the VM console and run the PBS installer:
- target = the 32 GB disk
- network: static `10.0.10.6/24`, gateway `10.0.10.1`, DNS `10.0.10.1`
- hostname `pbs` (→ FQDN `pbs.teejlab.dev`), set a root password

## Post-install (SSH to 10.0.10.6, as root)

```sh
# Proxmox ships only the enterprise repo (needs a subscription) -> swap to no-subscription
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pbs-enterprise.list
source /etc/os-release
echo "deb http://download.proxmox.com/debian/pbs ${VERSION_CODENAME} pbs-no-subscription" > /etc/apt/sources.list.d/pbs-no-subscription.list
apt update && apt install -y qemu-guest-agent nfs-common
systemctl start qemu-guest-agent

# mount the Pi's NFS export and persist it
mkdir -p /mnt/pbs-backups
echo "10.0.10.4:/export/teejlab-backups  /mnt/pbs-backups  nfs  defaults,_netdev  0 0" >> /etc/fstab
mount -a
df -h /mnt/pbs-backups          # confirms the ~1.3T Pi pool

# create the datastore on the mount
proxmox-backup-manager datastore create teejlab-backups /mnt/pbs-backups
```

## Pi-side prerequisites (OMV)

The datastore folder on the Pi must be exported correctly:
- **NFS export** of the folder (OMV exports it as `/export/teejlab-backups`, *not* the raw pool
  path `/teejlab-tank/teejlab-backups` — check with `showmount -e 10.0.10.4`).
- Export options include **`no_root_squash`** (and `rw,insecure`). Without it, datastore
  creation fails with `EPERM` (see lessons).
- The folder owned by **uid 34** (`backup`): `chown 34:34 /teejlab-tank/teejlab-backups` on the Pi.

## Add to the cluster (Datacenter -> Storage -> Add -> Proxmox Backup Server)

- ID `pbs`, Server `10.0.10.6`, Datastore `teejlab-backups`
- Username **`root@pam`** (the realm is `pam`, not the hostname)
- **Fingerprint**: paste PBS's SHA-256 — get it with `proxmox-backup-manager cert info | grep -i fingerprint`

CLI equivalent:
```sh
pvesm add pbs pbs --server 10.0.10.6 --datastore teejlab-backups \
  --username root@pam --password 'PW' --fingerprint <sha256>
```

Then run a test backup of a small VM/CT and confirm it appears in PBS Content.

## Lessons learned

- **`EPERM` on `datastore create` over NFS = `root_squash`.** PBS does part of the create as
  root (chowning the chunk store); the default `root_squash` maps root → `nobody`, which can't
  chown → `EPERM`. Fix: `no_root_squash` on the export (`root_squash` only squashes uid 0, so
  uid-34 writes pass through — but the root-side chown still needs this). The locked-down
  alternative is `all_squash,anonuid=34,anongid=34`.
- **OMV exports via `/export/<share>`**, a bind-mount alias — not the raw ZFS path. Always
  confirm the real export path with `showmount -e`.
- **Proxmox/PBS ship only the enterprise repo** — swap to `pbs-no-subscription` before any
  `apt install`, or even Debian-stock packages "can't be located."
- **Cluster storage username is `root@pam`** — the `@` is the auth *realm* (pam = system users),
  not the hostname. There's coincidentally a `pbs` realm, but it has no `root` user.
- **PBS self-signed cert → fingerprint required** when adding the storage (no CA to trust).
- **`rm -rf` near an NFS mount is dangerous** — a stray delete inside `/mnt/pbs-backups` hits
  the Pi; and a wrong path nuked the VM OS once. `df -h <path>` before deleting; the rebuild was
  cheap because the datastore lives on the Pi, not the VM.
