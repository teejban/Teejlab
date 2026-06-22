# 0008. Short hostnames under the domain (drop the redundant `teejlab-` prefix)

- **Status**: Accepted
- **Date**: 2026-06-21

## Context

The original convention used a `teejlab-<service>` prefix for "services and infrastructure"
(e.g. `teejlab-pi-nas`), while internal DNS was meant to be `<service>.teejlab.dev`. Applied
to a hostname's FQDN, that produces redundancy: `teejlab-pbs.teejlab.dev` carries "teejlab"
twice. As more VMs/services come online, an inconsistent mix had already crept in
(`teejlab-pi-nas` with a hyphen vs. a VM named `teejlab.opnsense` with a dot).

## Decision

For anything **DNS-addressable**, the hostname is the **short service/role name** and the
domain provides the namespace: `<service>.teejlab.dev`. Drop the `teejlab-` prefix on
hostnames. The `teejlab-` prefix is kept only for identifiers that have **no domain** to carry
the namespace.

- DNS-addressable hosts/services: `pbs`, `opnsense`, `nas` → `pbs.teejlab.dev`, etc. The
  Proxmox VM display name matches the OS hostname.
- Hypervisors: `teejhost<N>` (role-based; not redundant with "teejlab").
- Non-DNS identifiers: keep `teejlab-` (e.g. ZFS pool `teejlab-tank`, project handle `teejlab`).

OS hostnames are always a **single label** (e.g. `pbs`), never dotted (`teejlab.pbs` would be
parsed as host `teejlab` in domain `pbs`).

## Consequences

- Clean, non-redundant FQDNs (`pbs.teejlab.dev`), and the VM name == hostname, so no
  dot/FQDN confusion between the Proxmox label and the guest's hostname.
- Some existing names don't match yet and will be renamed as convenient:
  - `teejlab.opnsense` (VM) → `opnsense`
  - the rebuilt PBS uses `pbs` from the start
  - `teejlab-pi-nas` → `nas` eventually (low priority; it's a working OMV host)
- A naming decision is now documented, so future services follow one rule instead of drifting.

## Alternatives considered

- **Keep `teejlab-<service>` hostnames**: produces the redundant `teejlab-x.teejlab.dev` FQDNs
  and duplicates the namespace the domain already provides.
- **Dotted VM/hostnames** (`teejlab.opnsense`): valid as a Proxmox *display* name, but wrong as
  an OS hostname (a hostname must be a single label; the dot implies an FQDN split).
