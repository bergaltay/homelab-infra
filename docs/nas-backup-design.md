# NAS and backup design

## Goal

A passive backup target for the Proxmox host. The NAS never reaches into the host. The host pushes snapshots and files, and the NAS can only receive them.

## Hardware

ECS NM70-I2 Mini-ITX board (Celeron 1037U, about 4.8 GB RAM), Debian 13, ZFS 2.3.9, ARC capped at 1.5 GB. Only two SATA ports run at 6 Gbps, so some disks run at 3 Gbps. Disks are referenced by `/dev/disk/by-id` everywhere, because drive letters change between boots.

## Pools

| Pool | Disks | Redundancy | Intended for |
|---|---|---|---|
| `pair2` | 2 x 500 GB | Mirror (450 GB usable) | Important backups |
| `backup` | 1 x 750 GB | None | Less critical backups, plus `backup/depo` for general storage |

`backup/depo` is **not a backup**: one disk, one copy. Snapshots there only protect against accidental deletion. There is no copy between the two pools. Each is fed from the host separately.

Both mirror disks are old. The SMART load-cycle counter is exhausted on one and high on the other. APM was raised so the heads stop parking, and smartd watches both. If either fails, the plan is `zpool replace`.

## Push model and least privilege

- A dedicated user receives backups. Its SSH key is restricted (`restrict`, `from=<host>`).
- Delegated ZFS permissions on the receive datasets: `create, mount, receive, rollback, hold, release`. `destroy`, `snapshot` and `send` are not granted. If the Proxmox host were compromised, the snapshots on the NAS could not be deleted through this account.
- `rollback` is needed because syncoid receives with `-F` by default.
- The firewall drops all inbound traffic by default. SSH is allowed from the LAN. The metrics agent and the status endpoint accept a single host.

## What was tested

| Test | Result |
|---|---|
| Initial receive with `-u`, then incremental receive | Worked |
| Receive without `-u` | Failed at mount, as expected for a non-root user |
| `destroy` through the backup account | Denied |
| Reboot | Pools import and mount automatically |
| Scrub completion alert, smartd test alert | Delivered to the notification script |
| Firewall rules | Applied with a 5-minute automatic rollback, then a fresh SSH connection was confirmed |
| Copy of old NTFS data (12,835 files, 65.8 GB) before wiping the disk | Verified by file count, total bytes and per-file checksum. A permanent snapshot was kept |

## Monitoring

| What | How | Frequency |
|---|---|---|
| Disk health and temperature | smartd, warn at 55 C | Every 30 min. Short test weekly, long test monthly |
| Pool and disk state, I/O and checksum errors | ZFS event daemon | On event |
| Scrub | Cron | Monthly |
| Backup freshness | Custom script, flags anything older than 26 hours | Daily |
| Backup status as JSON | Same check without alerts, read by a dashboard | Hourly |
| Host metrics | Beszel agent | Continuous |

All alerts go through one script, so adding a delivery channel later is a change in one place. No channel is configured yet.

Snapshot retention is 14 daily, 8 weekly and 6 monthly. This is a first guess and will be revised.

## Wake-on-LAN

The NAS is meant to be powered on when needed. Linux-side settings alone did not work in either off or sleep states. Capturing traffic showed the magic packets did arrive, so the network was not the cause. The fix was a BIOS option named "Resume By PME". It works after a normal shutdown, but may not after a forced power-off or a power cut.

## Cooling

Before a fan was added, one disk reached 55 C while idle, which is the alert threshold. Shortly after boot with the fan, disks read 28 to 33 C. That reading was taken minutes after boot, so the long-run effect is **not yet verified**. The board's fan sensors are not visible from Linux yet.

## Not done yet

- Replication from the host (source side not configured yet).
- An alert delivery channel.
- A second mirror pool from the two 320 GB disks that are still in the Proxmox host. One of them has read errors (see the audit), so this depends on that disk being replaced or checked first.
- The 3D-printed case.
