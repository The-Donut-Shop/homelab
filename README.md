# homelab

Configuration-as-code for a single home server: everything that runs on it,
in public repos with no secrets (each repo has a `.env.example`; real values
live only on the server in git-ignored `.env` files).

| Repo | What |
|---|---|
| [homelab](https://github.com/The-Donut-Shop/homelab) | this repo: host setup, storage, disks, backups, runbooks |
| [homelab-media](https://github.com/The-Donut-Shop/homelab-media) | request → download → organise → watch/listen/read (Jellyfin, Navidrome, Seerr, Aurral, Radarr, Sonarr, Bazarr, Calibre-Web-Automated, …) |
| [homelab-edge](https://github.com/The-Donut-Shop/homelab-edge) | public access: Cloudflare Tunnel, Caddy, shared login gate |
| [homelab-photos](https://github.com/The-Donut-Shop/homelab-photos) | Immich photo library |
| [homelab-nvr](https://github.com/The-Donut-Shop/homelab-nvr) | Frigate NVR for the cameras |
| [homelab-services](https://github.com/The-Donut-Shop/homelab-services) | Gitea, MinIO, Kiwix, InvoicePlane, Portainer agent |

## Host

- 4-thread Intel i3 (Skylake) with HD Graphics 510 (Quick Sync / OpenVINO), 8 GB RAM, Ubuntu, Docker + Compose.
- Remote access: Tailscale (private) and Cloudflare Tunnel (public, see homelab-edge).
- After a power cut everything comes back by itself; see
  [docs/runbooks/power-cut.md](docs/runbooks/power-cut.md) for what to expect and how to check.

Host files in [`host/`](host/), and where each one is installed:

| File | Installed as | Why |
|---|---|---|
| `fstab.example` | `/etc/fstab` | disks by UUID, mergerfs pool that refuses to start without its disk |
| `smartd.conf` | `/etc/smartd.conf` | SMART monitoring and self-tests, per disk |
| `zram-generator.conf` | `/etc/systemd/zram-generator.conf` (`apt install systemd-zram-generator`) | compressed swap in RAM, ahead of the swap file on the slow OS disk |
| `sysctl-zram.conf` | `/etc/sysctl.d/92-zram.conf` | swap tuning that goes with zram |
| `sysctl-inotify.conf` | `/etc/sysctl.d/91-inotify.conf` | enough file watchers for Immich's library watcher |
| `sysctl-reserved-ports.conf` | `/etc/sysctl.d/90-reserved-ports.conf` | keeps listening ports out of the ephemeral range |
| `systemd/docker.service.d/` | `/etc/systemd/system/docker.service.d/` | Docker waits for the pool, never gives up restarting |
| `systemd/containerd.service.d/` | `/etc/systemd/system/containerd.service.d/` | containerd gets time to start on the slow OS disk |

After copying: `sudo systemctl daemon-reload && sudo sysctl --system && sudo systemctl restart smartd`.

## Storage

```
sda  500 GB   OS, Docker, app state (~/appdata)
sdb  1 TB     /mnt/disk2  ──┐
                            ├── mergerfs pool  /mnt/storage   (see host/fstab.example)
(new disks)   /mnt/diskN  ──┘
sdc  1 TB     /mnt/recovery  read-only, never auto-mounted (old data-recovery set, damaged filesystem)
```

```
/mnt/storage
├── data/          the media stack's single root (mounted as /data in containers, see homelab-media)
├── photos/        Immich library + external libraries
├── cctv/          Frigate recordings
├── archive/       personal archives
├── appdata/immich Immich database
└── backups/       config backups
```

Rules that keep data safe:

1. **The OS disk is never part of the pool.** Pool branches are only real data disks.
2. **A missing disk fails loudly.** The pool mount requires its disks
   (`x-systemd.requires-mounts-for`), and the empty mount-point folders are
   immutable (`chattr +i`), so nothing can silently write into the OS disk if a
   disk doesn't mount.
3. **Databases don't live on mergerfs.** Immich's Postgres points at the disk path directly.
4. **SMART is monitored** (`host/smartd.conf`): daily short test, monthly long
   test; the failing recovery disk is only watched, never tested.

See [docs/disks.md](docs/disks.md) for the current health of every disk and
[docs/runbooks/add-a-disk.md](docs/runbooks/add-a-disk.md) for replacing one.
