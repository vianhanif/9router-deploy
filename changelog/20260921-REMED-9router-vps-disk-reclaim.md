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
+ docker builder prune -f --filter "until=72h" --max-used-space 3G
```

- `--max-used-space 3G` adds a **size ceiling**; the age filter alone had none. Both apply: entries older than 72h go, and the cache is trimmed to 3G (oldest-first) if it exceeds it.
- Replaced `--keep-storage 3G` — deprecated in Docker CLI 27.5.1, identical semantics. Confirmed against the live daemon (CLI 27.5.1): the deprecated flag still runs but emits a deprecation warning.
- 3G chosen to keep ~2 recent builds hot — a tighter cap forces cold rebuilds, and the build step is the OOM-riskiest step on the 1.7GB VPS.
- Comment in-file updated to record the measured ~6GB drift as the reason for the ceiling.

## Verification

- Before/after on the live VPS: `df -h /`, `swapon --show`, `free -h` — swap dropped 6.3G → 2G, disk 55% → 44%.
- All 5 services healthy post-change; no OOM traces in `dmesg`/journal; listening ports unchanged.
- `python3 -c "import yaml; yaml.safe_load(...)"` on `deploy.yml` → parses clean.
- Manual `--max-used-space 3G` prune run on the VPS reclaimed **2.9GB** (BuildKit 8.6GB → 5.6GB); disk settled at **37%**.
- Deploy run `#35526258373` succeeded with the new prune step; `/var/lib/docker` ~11G after (5.6GB BuildKit, 3.5GB images, 2G runtime).
- `git diff` reviewed: comment + one flag; no unrelated edits.

## Risks / Limitations

- `--max-used-space 3G` is a **soft** target — BuildKit trims to it on the next prune, it is not a hard disk quota. A pathological build burst can still exceed 3G until the next deploy runs the prune. Cache younger than 72h is never evicted by this step, so expect ~2GB of fresh cache to sit outside the ceiling until it ages.
- Cache pruning to 3G means more frequent partial rebuilds; accepted trade for a bounded disk footprint on a 1.7GB box.
- `/etc/fstab.bak` was removed from the VPS once the swap change proved stable across deploys — no stale copy left behind.
- **Not done (available, ~250M):** capping journald via `SystemMaxUse=256M` in `/etc/systemd/journald.conf`. Host-level change, no repo artifact — left out of this change to keep the diff to a single flag.
