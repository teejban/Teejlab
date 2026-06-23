# 0010. Build the first Docker host by hand now, codify it in IaC later

- **Status**: Accepted
- **Date**: 2026-06-23

## Context

The roadmap sequences **IaC (Phase 2) before Docker (Phase 3)** — the rationale being that VMs
should be Terraform-provisioned and Ansible-configured for reproducibility. But the
auto-TLS-for-services layer (a reverse proxy) is needed now and is best run as a container, and
the natural place for that is a Docker host. Standing one up now jumps ahead of the IaC phase.

## Decision

Stand up the **first Docker host manually now** to unblock the reverse proxy / automatic TLS
(and to start the container journey), explicitly accepting that it's **hand-built, not yet
codified**. Treat this exact host as the **concrete target for the IaC phase**: when Phase 2
lands, reproduce its provisioning in Terraform and its configuration in Ansible as the practice
exercise.

## Consequences

- Unblocks the reverse proxy / auto-TLS and kicks off the Docker phase now, with a *real*
  first workload (not a throwaway demo) — consistent with the "build something real" goal.
- The host is initially un-codified. Acceptable: it's set-once infrastructure with low churn,
  and it becomes the **IaC practice target** rather than throwaway work.
- A slight, deliberate reordering of the roadmap (Docker partially precedes IaC). Recorded here
  so it reads as a choice, not drift.
- Sizing/placement: a Debian/Ubuntu VM on **teejhost1** (the larger node — 62 GiB RAM, two
  disks), on the **SERVERS VLAN (10.0.20.x)**, started modest (~2 vCPU / 4 GiB) and resized as
  workloads grow. Compose-driven, files in `infra/docker/`.

## Alternatives considered

- **Strict roadmap order (IaC first)**: delays the reverse proxy and auto-TLS for no real
  benefit; you can't Terraform-provision a Docker host meaningfully before learning Docker.
- **Caddy in a throwaway LXC**: solves TLS now but gets replaced once Docker arrives — wasted
  effort versus making the Docker host the permanent home.
