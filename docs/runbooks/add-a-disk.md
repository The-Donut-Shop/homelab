# Add (or replace) a data disk

Typical case: a new, bigger disk replaces a failing one. Nothing is deleted
until the copy is verified.

## 1. Prepare the disk

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL            # identify it, e.g. /dev/sdX
sudo smartctl -t long /dev/sdX             # optional burn-in: wait for it, then
sudo smartctl -l selftest /dev/sdX         # must say "Completed without error"
sudo parted -s /dev/sdX mklabel gpt mkpart data ext4 0% 100%
sudo mkfs.ext4 -L disk3 /dev/sdX1
sudo mkdir -p /mnt/disk3
lsblk -f /dev/sdX1                          # copy the UUID
```

Add it to `/etc/fstab` (see `host/fstab.example`), then:

```bash
sudo mount /mnt/disk3
```

## 2a. Grow the pool

Append `:/mnt/disk3` to the pool's branch list and to
`x-systemd.requires-mounts-for` in `/etc/fstab`, then remount (stop the
stacks first: `docker compose stop` in each repo):

```bash
sudo umount /mnt/storage && sudo mount /mnt/storage
```

## 2b. Move everything off a failing disk

```bash
# stop every stack that uses /mnt/storage, then copy with all metadata:
sudo rsync -aHAXx --numeric-ids --info=progress2 /mnt/disk2/ /mnt/disk3/
# verify: second pass must transfer nothing, and a checksum dry-run must list nothing
sudo rsync -aHAXx --numeric-ids --checksum --dry-run --itemize-changes /mnt/disk2/ /mnt/disk3/
```

Then replace `/mnt/disk2` with `/mnt/disk3` in the pool line, remount, start
the stacks, check the apps, and only then retire the old disk.

## 3. Lock the empty mount points

So that a disk that fails to mount can never cause writes into the OS disk:

```bash
sudo umount /mnt/storage /mnt/disk3
sudo chattr +i /mnt/storage /mnt/disk3      # immutable while nothing is mounted on them
sudo mount -a
```
