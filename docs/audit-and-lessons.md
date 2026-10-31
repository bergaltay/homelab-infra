# Self-audit and lessons

## Audit of the Proxmox host (20 Oct 2026)

I reviewed the host and wrote down what was wrong. All items below were still open on 31 Oct.

| # | Finding | Why it matters |
|---|---|---|
| 1 | The thin pool for guest disks was 86.9% full and over-allocated (206 GB allocated on a 136.5 GB pool) | If it fills, six guests stop with I/O errors |
| 2 | One ZFS mirror member showed 2 pending sectors and 2 read errors | A second fault on the other disk would lose the data |
| 3 | No scheduled backup and nothing stored off the machine | Any disk loss is permanent |
| 4 | After moving the stock-analysis pipeline to its own VM, old copies of its database and object store stayed on the container platform | Duplicate data on the fullest disk, and confusion about which one is live |
| 5 | The Nextcloud container could see the whole data pool, including backup dumps | No isolation, and no separate quota or snapshot policy possible |
| 6 | Several data services listen on all LAN interfaces | Unneeded exposure inside the LAN |
| 7 | Guests together need about all of the host RAM | Swap or out-of-memory kills if everything runs |

Order of work I planned: free space on the thin pool, test the failing disk and move critical data to the NAS, only then restructure.

## Lessons from the build

- **Drive letters change between boots.** On the NAS I used `by-id` paths from the start. The host's pool still uses `sdX` names, which is on the cleanup list.
- **Snapshots on the same pool are not a backup.** The nightly snapshots and a manual dump on the host sit on the pool they protect.
- **Thin-provisioned guests need `discard`.** Without it, blocks freed inside a guest are not returned to the pool. One guest showed 97% in the pool while being 80% full inside.
- **Check before deleting a migrated service.** For the old database and object store I looked for connected clients and last-write times before treating them as unused.
- **A leftover `policy-rc.d` blocked services** from starting after package installs on the NAS. It came from the OS installer and was removed.
- **ZFS pool names cannot start with `mirror`, `raidz`, `draid` or `spare`.** I renamed a pool.
- **Wake-on-LAN needed a BIOS setting**, found by checking whether packets reached the machine at all before changing more Linux settings.
- **Least privilege needs testing.** The `rollback` permission was only found necessary by running a real receive.
