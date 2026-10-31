# Architecture

## Layers

1. **Compute and services:** one Proxmox host. Each function lives in its own VM or LXC: remote access, file sync, a container platform for self-hosted apps, GPU workloads, the stock-analysis pipeline, file sharing.
2. **Storage:** a SATA SSD for Proxmox and guest OS disks, a ZFS mirror of two HDDs for Nextcloud data and git/database volumes, a separate SSD for the stock-analysis VM, and a USB SSD for the SMB share.
3. **Backup:** a separate NAS, so backups do not live on the same machine as the data. See [nas.md](nas.md).
4. **Monitoring:** a Beszel hub on the container platform collects metrics from the host and the NAS. A small status endpoint on the NAS reports backup freshness as JSON for a dashboard.

## Remote access

Remote access uses Tailscale. A dedicated LXC runs it as a subnet router: it advertises the home LAN to my tailnet, so remote devices can reach LAN hosts without installing Tailscale on each of them.

## How it is structured

- **One function per guest** (gateway, files, apps, GPU workloads, pipeline, share), so a guest can be rebuilt or replaced without touching the others.
- **The backup target is a different machine**, not another pool on the same host.
- **Local LLM serving:** the model servers listen on localhost only, and a gateway is the one exposed endpoint.
