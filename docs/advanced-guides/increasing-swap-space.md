# Increasing Swap Space on a UGREEN NAS

## Background

UGREEN NAS devices ship with model-dependent RAM configurations and a swap configuration:

- **zram** — (compressed RAM-based swap), which improves effective memory usage with low latency but depends on available RAM and CPU, and
- **A dedicated swap partition** on the system NVMe (typically `nvme2n1p5`).

Total out-of-the-box swap is around 10 GB. On an 8 GB NAS running 40–60 Docker
containers this fills within days or weeks. Once swap is nearly full the system
reaches a tipping point where **new processes are killed before they finish
starting**.

---

## Symptoms

- A container is stuck in a restart loop with **exit code 137** and
  `OOMKilled: false` in `docker inspect`
- `docker logs` for the crashing container cuts off mid-startup
- Load average is persistently high relative to CPU count
- `free -h` shows swap nearly or completely full
- High I/O wait (`wa`) in `vmstat` or `top`

```bash
free -h
```

```
               total        used        free      shared  buff/cache   available
Mem:            7.5Gi       6.1Gi       432Mi       282Mi       1.9Gi       1.3Gi
Swap:           9.8Gi       9.7Gi       100Mi
```

```bash
vmstat 1 3
```

Watch the `si`/`so` columns (swap-in / swap-out). Heavy, sustained activity
confirms the system is thrashing.

---

## Why OOMKilled is false

UGREEN UGOS ships with [earlyoom](https://github.com/rfjakob/earlyoom), a
userspace daemon that proactively kills processes before the kernel OOM killer
fires. Because earlyoom sends SIGKILL directly — outside of Docker — Docker
never learns a kill happened, so `OOMKilled` stays false and the exit code is
137 (128 + signal 9).

Check that earlyoom is running and read its thresholds:

```bash
# Is earlyoom running?
pgrep -a earlyoom
```

```bash
# What are its thresholds?
# -m 5      = kill when available memory < 5%
# -M 65536  = kill when available memory < 65536 KiB (64 MiB)
# -s 20     = kill when swap free < 20%
cat /etc/default/earlyoom
```

```bash
# Recent victims
grep earlyoom /var/log/syslog | grep SIGKILL | tail -10
```

With `-s 20` and 10 GB total swap, earlyoom's kill threshold is 2 GB of free
swap. A system at 97 % swap usage is permanently below that line, so every new
process that allocates significant memory gets killed immediately.

### Confirming earlyoom is the culprit

```bash
# Memory pressure by container (resident RAM only, excludes swap)
docker stats --no-stream --format 'table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}' \
  | sort -k3 -nr
```

```bash
# Swap usage by host process (identifies what is paged out)
for pid in $(ls /proc | grep -E '^[0-9]+$'); do
  swap=$(awk '/VmSwap/{print $2}' /proc/$pid/status 2>/dev/null)
  comm=$(cat /proc/$pid/comm 2>/dev/null)
  [ -n "$swap" ] && echo "$swap $comm $pid"
done | sort -rn | head -20
```

---

## Where to put the swapfile

Run `lsblk` and `df -h` to orient yourself:

```bash
lsblk
df -h
```

On a UGREEN NAS the system NVMe (`nvme2n1`) typically has a large `/overlay`
partition (100+ GB, usually less than 30 % used) and a smaller OS partition.
`/overlay` is on fast NVMe storage and is the right place for extra swap.

**Do not** use:

- `/volume1` or any HDD RAID array — high latency random I/O makes swap
  thrashing dramatically worse
- A network filesystem (NFS, SMB)
- A BTRFS or ZFS volume without extra configuration

Check headroom before sizing:

```bash
df -h /overlay
```

Leave at least 50 % of the partition free after the swapfile. A 16 GiB
swapfile on a 107 GiB `/overlay` partition with 23 GiB already used leaves
68 GiB free — well within that margin.

---

## Creating the swapfile

```bash
# 1. Allocate (instant with fallocate; dd works but is slow)
sudo fallocate -l 16G /overlay/swapfile

# 2. Lock down permissions — the kernel rejects world-readable swapfiles
sudo chmod 600 /overlay/swapfile

# 3. Format
sudo mkswap /overlay/swapfile

# 4. Activate for this session (until next reboot)
sudo swapon /overlay/swapfile

# 5. Confirm
free -h
sudo swapon --show
```

---

## Making it persistent across reboots — use a polling systemd unit, not fstab

Two obvious approaches both fail on UGOS:

**`/etc/fstab`** — `systemd-fstab-generator` converts swap entries into
transient `.swap` units. Those units get an `After=overlay.mount` ordering
dependency but no `Requires=`. UGOS early-boot scripts call `swapon -a` before
`/overlay` is mounted; the unit fails, systemd does not retry, the swapfile
never activates.

**A `.swap` unit with `WantedBy=swap.target`** — `swap.target` fires during
early boot before `/overlay` is mounted. Same result.

**A service unit with `WantedBy=multi-user.target` and `After=overlay.mount`**
— close, but UGOS-specific services pull `multi-user.target.wants` units into
the dependency graph before `multi-user.target` itself is reached, and before
`/overlay` finishes mounting. The `After=overlay.mount` ordering constraint is
not reliably respected on UGOS.

The fix is a `Type=oneshot` service that **polls for the mount itself** inside
`ExecStart`. This makes the unit self-contained: it does not depend on systemd's
ordering guarantees and works regardless of what triggers it or when.

```bash
sudo tee /etc/systemd/system/overlay-swapfile.service << 'EOF'
[Unit]
Description=Activate swapfile on /overlay

[Service]
Type=oneshot
TimeoutStartSec=300
ExecStart=/bin/sh -c 'until mountpoint -q /overlay; do sleep 2; done; grep -q /overlay/swapfile /proc/swaps || /sbin/swapon /overlay/swapfile'
ExecStop=/sbin/swapoff /overlay/swapfile
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now overlay-swapfile.service
```

Verify it is active and enabled:

```bash
systemctl status overlay-swapfile.service
free -h
```

`ExecStart` polls `mountpoint -q /overlay` every two seconds until the
filesystem is ready, then calls `swapon` only if the swapfile is not already
active. `TimeoutStartSec=300` caps the wait at five minutes — `/overlay`
normally mounts within the first 30 seconds of boot. If the mount never
appears, the service fails without blocking the rest of the boot.

---

## Verifying earlyoom backs off

earlyoom's `-s 20` threshold on a 26 GB swap device sits at ~5 GB free. Adding
16 GB of extra swap puts you ~20 GB above that line under normal load:

```bash
free -h
grep earlyoom /var/log/syslog | grep -v 'fre:\|Fre:' | tail -5
```

If earlyoom has stopped logging kills, there is enough headroom.

---

## Reverting

```bash
# Stop and disable the service
sudo systemctl disable --now overlay-swapfile.service

# Remove the unit file
sudo rm /etc/systemd/system/overlay-swapfile.service
sudo systemctl daemon-reload

# Delete the swapfile
sudo rm /overlay/swapfile
```

---

## Sizing guidelines

| RAM   | Containers | Extra swap to add |
| ----- | ---------- | ----------------- |
| 8 GB  | 20–40      | 8–16 GB           |
| 8 GB  | 40–60+     | 16–24 GB          |
| 16 GB | 40–60      | 8 GB              |
| 16 GB | 60+        | 16 GB             |

Keep earlyoom's `-s` threshold in mind: `total_swap × 0.20` is the kill line.
Size the swapfile so that normal steady-state usage stays well below it.

Adding swap extends the runway. If swap is perpetually full the underlying
problem is too many containers for the available RAM — adding more swap only
delays the next crisis.

---

## Further reading

- [earlyoom](https://github.com/rfjakob/earlyoom)
- [`man swapon(8)`](https://man7.org/linux/man-pages/man8/swapon.8.html)
- [`man systemd.swap(5)`](https://www.freedesktop.org/software/systemd/man/systemd.swap.html)
