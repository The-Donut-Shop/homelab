# 2026-11 media overhaul

## Before

- One Compose project per app under `~/media-stack`, each mounting different slices
  of storage at different container paths. No hardlinks, imports became copies, and a
  custom organizer script tried to make up for it.
- Music through Lidarr + soularr + a custom request app + a custom lyrics script.
- Storage pool `/mnt/storage` = mergerfs over `/mnt/disk1` + `/mnt/disk2`, where disk1's
  filesystem had been failing boot fsck since September. mergerfs silently used the
  empty mount-point folder on the **OS disk** as half of the pool.

## What changed

1. **Storage**
   - The OS-disk half of the pool was copied to disk2 with `rsync -aHAXx` and verified:
     every file by size and mtime, 6,862 critical files by SHA-256.
   - The pool became disk2-only (live, via mergerfs runtime branches; no unmount).
   - `fstab` was rewritten: the pool requires its disks, and the broken disk is never auto-mounted.
   - The empty mount points were made immutable.
   - Immich's Postgres moved to a direct disk path.
2. **Layout:** everything media lives under one root, `/mnt/storage/data`
   (`downloads/` + `media/{movies,series,music,books,audiobooks,podcasts}`), mounted as
   `/data` in every container.
3. **Library:** movies and series were moved into the new layout and renamed by
   Radarr/Sonarr themselves. Every file kept its inode (pure renames, nothing copied).
4. **Apps:** one Compose project ([homelab-media](https://github.com/The-Donut-Shop/homelab-media))
   with pinned versions:
   - Seerr replaced Jellyseerr.
   - Aurral replaced Lidarr, soularr and the custom music-request app.
   - beets-getlrc replaced the custom lyrics script.
   - New: Bazarr, Calibre-Web-Automated, Shelfmark, Audiobookshelf, Homepage, Cleanuparr.
5. **Edge:** the Caddy + gate config moved to [homelab-edge](https://github.com/The-Donut-Shop/homelab-edge):
   - fixed an open redirect in the gate, made the cookie `Secure`, and baked Flask into its image;
   - added the `home.`, `books.` and `book-requests.` hosts.
6. **Music:** the old music library was deliberately deleted by the owner, to be rebuilt through Aurral.

## Lessons

- `nofail` on a pool member hides failures; pair it with `x-systemd.requires-mounts-for`
  and immutable mount points.
- Published container ports inside the ephemeral range (32768–60999, e.g. slskd's 50300)
  can collide with outgoing connections; reserve them with `net.ipv4.ip_local_reserved_ports`
  (`host/sysctl-reserved-ports.conf`).
- Torrent clients write files out of order, so big downloads end up heavily fragmented and
  read slowly from HDDs. Hardlink-based imports avoid ever having to copy them.
