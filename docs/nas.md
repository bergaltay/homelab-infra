# NAS and backup design

## What it is

A separate machine used as a passive backup target for the Proxmox host: ECS NM70-I2 Mini-ITX board (Celeron 1037U), Debian 13, ZFS. It can be powered on remotely with Wake-on-LAN.

## Storage

| Pool | Disks | Used for |
|---|---|---|
| `pair2` | 2 x 500 GB mirror | Important backups |
| `backup` | 1 x 750 GB | Less critical backups and general storage |

Disks are referenced by `/dev/disk/by-id`, because drive letters change between boots.

## Backup design

- **Push model:** the Proxmox host sends ZFS snapshots (syncoid) and files (rsync) over SSH. The NAS never connects to the host.
- **Least privilege:** the account that receives backups can create and receive datasets but cannot destroy snapshots, and its SSH key is restricted to the host's address.
- **Snapshots:** taken and pruned with sanoid, 14 daily, 8 weekly and 6 monthly.
- **Firewall:** all inbound traffic is dropped by default. SSH is allowed from the LAN, and the monitoring endpoints accept a single host.

## Monitoring

- smartd for disk health and temperature, with scheduled short and long tests.
- ZFS event daemon for pool and disk state changes, and a monthly scrub.
- A script that checks backup freshness and publishes the result as JSON for a dashboard.
- A Beszel agent for host metrics.
- All alerts go through one script, so a delivery channel (ntfy, Telegram or mail) can be added in one place.

## In progress

- Setting up the scheduled backups from the host.
- A compact 3D-printed case with six 2.5" bays, designed in Fusion 360.
