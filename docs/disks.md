# Disk health log

Update this file whenever a disk is added, replaced or tested
(`sudo smartctl -a /dev/sdX`, `sudo smartctl -l selftest /dev/sdX`).

## 2026-11-24 audit

| Disk | Model | Hours | Role | SMART | Notes |
|---|---|---|---|---|---|
| sda | WD Green 500 GB (WD5000AZRX) | 53,415 (~6 yrs) | OS, Docker, app state | PASSED | 1 pending sector, 3 offline-uncorrectable, 1,413 historic CRC errors (cable). Old: plan to replace. |
| sdb | Seagate 1 TB (ST1000DM010) | 14,261 | pool data disk (`/mnt/disk2`) | PASSED, but **extended self-test: read failure** at LBA 144,806,592 | 8 pending sectors, 5,829 reported-uncorrectable (last logged at ~9,670 h). The failing LBA maps to free space (no file affected) on 2026-11-24. **Treat as failing: replace and keep an off-box copy of anything irreplaceable.** |
| sdc | Seagate 1 TB (ST1000DM010), label `RECOVERY` | 9,271 | not in use (`/mnt/recovery`, read-only) | PASSED | ext4 has errors (multiply-claimed blocks); boot fsck failed every boot since ≥ 2026-09-10. 65,535 reported-uncorrectable, read errors ("aborted command") under load. Holds an old PhotoRec data-recovery set (~544 GiB), kept untouched. |

### What happened

`/etc/fstab` mounted sdc1 at `/mnt/disk1` with `nofail` and pooled
`/mnt/disk1:/mnt/disk2` with mergerfs. When sdc1's filesystem broke (errors
first logged 2026-08-16), boot fsck failed, sdc1 silently didn't mount, and
mergerfs kept using the *empty `/mnt/disk1` folder on the OS disk* as half of
the pool. For ~2.5 months new files were split between disk2 and the OS disk,
including Immich's Postgres files, and everything already on sdc1 vanished from
`/mnt/storage`.

Fixed on 2026-11-24: OS-disk data copied onto disk2 and verified, OS disk
removed from the pool, sdc1 moved to a read-only, never-auto-mounted
`/mnt/recovery`, photos that only existed on sdc1 copied back (SHA-256
verified), pool now refuses to start if its disk is missing.
