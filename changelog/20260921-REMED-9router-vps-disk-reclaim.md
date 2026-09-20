# Task: 9router-deploy — VPS disk reclaim + BuildKit cache ceiling

- **Date:** 2026-09-21
- **Ticket:** REMED (placeholder — replace with real ticket id)
- **Status:** Done
- **Branch:** `master`

## Problem

VPS at 44–55% disk after the earlier image-leak fix. A second audit found the remaining bulk was **not** Docker images:

1. Two active swap files: `/swap.img` (2GB) + `/swap2.img` (4GB) = 6.3GB. `swap2` was idle — present in `/etc/fstab` and `swapon --show`, but carrying **~0 bytes** of paging. Pure dead weight on a 40GB disk.
2. BuildKit cache held ~6GB. `deploy.yml` bounded it by **age** only (`--filter until=72h`), never by size, so the ceiling was "however much 72 hours of builds produce".

Gap closed arithmetically: 9.4G overlay2 + 3.4G `/usr` + 6.3G swap + logs + root ≈ 21G used.

Config reading alone was a trap here — verification required live `du`/`swapon` output from the VPS.

## Changes

### 1. VPS host (no repo change)

- `swapoff /swap2.img` → removed its line from `/etc/fstab` (backup kept as `/etc/fstab.bak`) → `rm /swap2.img`.
  **Freed 4GB.** Disk 21G → 17G used, 55% → 44%.
- `/swap.img` (2GB) **kept**: the 1.7GB-RAM box was actively paging ~826MB into it. Removing it risks OOM during the build step.

### 2. `.github/workflows/deploy.yml` — prune step

```diff
- docker builder prune -f --filter "until=72h"
+ docker builder prune -f --filter "until=72h" --keep-storage 3G
```

- `--keep-storage 3G` adds a **hard size ceiling**; the age filter alone had none. Both apply: entries older than 72h go, and the cache is trimmed down to 3G if it exceeds it.
- 3G chosen to keep ~2 recent builds hot — a tighter cap forces cold rebuilds, and the build step is the OOM-riskiest step on the 1.7GB VPS.
- Comment in-file updated to record the measured ~6GB drift as the reason for the ceiling.

## Verification

- Before/after on the live VPS: `df -h /`, `swapon --show`, `free -h` — swap dropped 6.3G → 2G, disk 55% → 44%.
- All 5 services healthy post-change; no OOM traces in `dmesg`/journal; listening ports unchanged.
- `python3 -c "import yaml; yaml.safe_load(...)"` on `deploy.yml` → parses clean.
- `git diff` reviewed: comment + one flag; no unrelated edits.

## Risks / Limitations

- `--keep-storage 3G` is a **soft** target — BuildKit trims to it on the next prune, it is not a hard disk quota. A pathological build burst can still exceed 3G until the next deploy runs the prune.
- Cache pruning to 3G means more frequent partial rebuilds; accepted trade for a bounded disk footprint on a 1.7GB box.
- `.bak` of `/etc/fstab` left on the VPS. Harmless, but it is a stale copy — remove if the swap change is confirmed permanent.
- **Not done (available, ~250M):** capping journald via `SystemMaxUse=256M` in `/etc/systemd/journald.conf`. Host-level change, no repo artifact — left out of this change to keep the diff to a single flag.
