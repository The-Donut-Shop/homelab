# After a power cut

Everything comes back on its own. This page is what to expect, how to check it,
and what happened the last time.

## What recovers by itself

| Layer | How |
|---|---|
| Filesystems | ext4 replays its journal at boot. |
| Storage pool | `/mnt/storage` mounts only once `/mnt/disk2` is there (`x-systemd.requires-mounts-for`); the empty mount points are immutable, so nothing can land on the OS disk by mistake. |
| containerd | gets 10 minutes to open its metadata DB on the slow OS disk instead of being killed after 90 s ([`host/systemd/containerd.service.d/`](../../host/systemd/containerd.service.d/)). |
| Docker | starts only after the pool is mounted and never stops retrying ([`host/systemd/docker.service.d/`](../../host/systemd/docker.service.d/)). |
| Containers | every container with `restart: unless-stopped` / `always` starts again. Opt-in ones stay off: Byparr, Cleanuparr and Immich's local machine learning only exist when started with their compose profile. |
| Databases | the SQLite apps and Immich's Postgres recover themselves (Postgres logs `automatic recovery in progress` … `ready to accept connections`). |
| Memory | zram swap is created at every boot ([`host/zram-generator.conf`](../../host/zram-generator.conf)). |
| Second machine | donutmaster has Docker enabled; Byparr and the remote Immich ML start with it. |

## How long it takes

A cold start is limited by the OS disk (5400 rpm) and 8 GB of RAM: ~45
containers all start at the same moment and fight over the disk. On 2026-11-25
it took almost an hour until every app answered: most were up within 5–30
minutes, Frigate was recording again after 42 minutes, Seerr was last at about
55 minutes. zram has
been added since, which removed most of the swapping; an SSD for the OS disk
(and/or 16 GB of RAM) is what would make this take a few minutes. A UPS avoids it.

## Check

```bash
systemctl --failed                                   # expect: 0 units
docker ps -a --filter status=exited --filter status=restarting \
  --format '{{.Names}}\t{{.Status}}'                 # expect: only opt-in containers
swapon --show; zramctl                               # zram0 present, priority 100
cat /proc/loadavg                                    # settles below ~3 once everything is up
findmnt /mnt/storage; findmnt /mnt/disk2
```

Apps (from the server; public URLs should answer with a redirect to the login gate):

```bash
for u in 8080/ 9696/ping 7878/ping 8989/ping 6767/ 5055/api/v1/status 8096/health \
         3001/ 4533/ping 8184/ 8183/ 13378/healthcheck 2283/api/server/ping; do
  printf '%-22s %s\n' "$u" "$(curl -s -o /dev/null -m 15 -w '%{http_code}' http://127.0.0.1:$u)"
done
docker exec frigate curl -s http://127.0.0.1:5000/api/version   # Frigate's API isn't published on the host
```

CCTV: new recording segments should appear every ~10 s per camera under
`/mnt/storage/cctv/recordings/<date>/<hour>/<camera>/`.

## 2026-11-25 power cut

| Time (UTC) | |
|---|---|
| 05:15 | Power lost (last CCTV segment 05:15:17). |
| 05:16 | Boot. ext4 replayed its journal on the root disk and on disk2; the pool mounted. |
| 05:17–05:21 | containerd took >90 s to open its metadata DB and was killed twice by systemd; Docker failed once. Everything started ~4 minutes late. **Fixed:** containerd start timeout 10 min. |
| 05:21 | Docker started all 46 containers at once. OS disk at 95 % utilisation, load average 50, heavy swapping to the swap file on the same disk. |
| 05:22–05:45 | Most apps answering. Immich's local ML container came back although it had been stopped: Immich uses `restart: always`, which ignores a manual stop. **Fixed:** container removed (it's profile-gated), ML stays on donutmaster. |
| 05:27 | Immich Postgres crash recovery OK. |
| 05:58 | Frigate recording again (CCTV gap 05:15–05:58). |
| ~06:10 | Seerr answering, last app up. Still heavy swapping: the working set didn't fit in 8 GB and the swap file sits on the slow OS disk. |
| ~06:20 | **Added zram** (compressed swap in RAM): swap-outs to disk went from ~300/s to 0, load fell from ~13 to ~5. The 1.7 GB already in the swap file is left there and comes back on demand: draining it with `swapoff` ran at only ~14 MB/min on this disk (~2 h of a saturated disk for mostly-cold pages), so it was cancelled. |

Also found and fixed while checking:

- Immich's external-library watcher hit the inotify limit (`ENOSPC: System
  limit for number of file watchers reached`): ~120k files to watch, limit was
  60k. **Raised** to 524,288 ([`host/sysctl-inotify.conf`](../../host/sysctl-inotify.conf)).
- smartd was still on Ubuntu's default config. **Installed** [`host/smartd.conf`](../../host/smartd.conf):
  per-disk, daily short tests, monthly long tests, recovery disk monitored only.
- The lyrics container did a full library pass on every start, adding load
  during boot. It now remembers when it last did one.

Verified afterwards: all 20 SQLite app databases pass `PRAGMA quick_check`,
Postgres recovered cleanly, no container logged database errors, settings of
qBittorrent/Jellyfin/Radarr/Sonarr/Prowlarr/Bazarr intact, public URLs up.
