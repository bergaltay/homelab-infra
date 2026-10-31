# Architecture

## Layers

1. **Compute and services:** one Proxmox host. Each function lives in its own VM or LXC: remote access, file sync, a container platform for self-hosted apps, GPU workloads, the stock-analysis pipeline, file sharing.
2. **Storage on the host:** four tiers on different media (below).
3. **Backup:** a separate NAS, so a backup does not share a failure domain with the host. See [nas-backup-design.md](nas-backup-design.md).
4. **Monitoring:** a Beszel hub on the container platform collects host and NAS metrics. A small status endpoint on the NAS exposes backup freshness as JSON for a dashboard.

## Storage tiers on the host

| Tier | Medium | Redundancy | Holds |
|---|---|---|---|
| LVM-thin (boot SSD) | 250 GB SATA SSD | None | OS disks of six guests, Proxmox itself |
| ZFS mirror `data` | 2 x 320 GB HDD | Mirror | Nextcloud data, git-forge and database volumes, SMB LXC root |
| Directory storage | 120 GB SSD | None | Stock-analysis VM |
| USB SSD | 1 TB | None | SMB share |

## What is lost if one disk fails (before replication exists)

| Failure | Consequence |
|---|---|
| Boot SSD | Proxmox, all guest OS disks, container-platform data, LLM model cache |
| 120 GB SSD | Stock-analysis database and object store |
| One ZFS mirror member | No data loss, but no redundancy until it is replaced |
| USB SSD | SMB share contents |

The SSD that carries the whole system is old: SMART reports about 5.8 years of power-on time and 120 TB written, above the manufacturer's 75 TBW rating (values reported by `smartctl`).

## RAM budget

The host has 32 GB. The five guests running at audit time use about 19 GB. Powering on the LLM VM brings the total to about 31 GB, so everything cannot run at once. ZFS ARC is capped at 3.1 GB and is not the problem; the guest allocations are. Which guests run together is a scheduling decision, not a given.

## Remote access

Remote access uses Tailscale. A dedicated LXC runs it as a subnet router: it advertises the home LAN to my tailnet, so remote devices can reach LAN hosts without installing Tailscale on each of them.

## How it is structured

- **One function per guest** (gateway, files, apps, GPU workloads, pipeline, share), so a guest can be rebuilt or replaced without touching the others.
- **The backup target is a different machine**, not another pool on the same host.
- **Local LLM serving:** the model servers listen on localhost only, and a gateway is the one exposed endpoint.
