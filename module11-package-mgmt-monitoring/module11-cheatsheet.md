# 📋 Module 11 Cheat Sheet — Package Management & System Monitoring

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Package** | A bundle of a program's files, config templates, and metadata about its dependencies |
| **Package manager** | Tool that installs/updates/removes packages and resolves dependencies automatically |
| **Repository** | A server hosting a collection of packages that `apt` can download from |
| **Package index** | The local cached list of what's available in each repository and at what version |
| **Dependency** | Another package a given package needs installed in order to run |
| **PPA** | Personal Package Archive — a third-party `apt` repository, usually on Launchpad |
| **Snap** | A self-contained, sandboxed package format shipped with modern Ubuntu |
| **Load average** | The 1/5/15-min average number of processes wanting CPU time |

## `apt` Command Reference (Ubuntu/Debian)

| Command | What it does |
|---|---|
| `sudo apt update` | Refreshes the **local index** only — installs/changes nothing |
| `sudo apt upgrade` | Installs newer versions of anything the (refreshed) index shows as outdated |
| `sudo apt install <pkg>` | Installs a package + its dependencies |
| `sudo apt remove <pkg>` | Uninstalls program files; **keeps** config files (`/etc/...`) |
| `sudo apt purge <pkg>` | Uninstalls program files **and** deletes config files |
| `sudo apt autoremove` | Removes orphaned dependency packages nothing needs anymore |
| `sudo apt clean` | Clears the downloaded `.deb` cache in `/var/cache/apt/archives` |
| `apt search <keyword>` | Searches package names/descriptions in the index |
| `apt show <pkg>` | Shows version, size, dependencies, description — no install |
| `apt list --installed` | Lists every installed package + version |
| `apt list --upgradable` | Lists packages with a newer version available |

⚠️ **Golden rule:** always `sudo apt update && sudo apt upgrade` — never `upgrade` alone.

## `dpkg` Command Reference (lower-level, apt is built on this)

| Command | What it does |
|---|---|
| `dpkg -l` | Lists all packages `dpkg` knows about + status |
| `dpkg -l \| grep <pkg>` | Check if a specific package is installed |
| `dpkg -L <pkg>` | Lists every file a specific installed package put on disk |
| `dpkg -S <file>` | **Reverse lookup** — which package owns this file? |
| `dpkg -s <pkg>` | Shows detailed status info for one package |

## RHEL-Family Awareness (yum / dnf)

| Distro family | Package manager | Notes |
|---|---|---|
| Ubuntu / Debian | `apt` (built on `dpkg`) | This module's focus |
| RHEL / CentOS (older) | `yum` | Same concepts, different syntax |
| Fedora / RHEL 8+ / Rocky / Alma | `dnf` | Modern `yum` replacement |

## Alternative Distribution Channels

| Tool | Command | Trust note |
|---|---|---|
| PPA | `sudo add-apt-repository ppa:<user>/<name>` then `apt update` | Third-party — vet the publisher first |
| Snap | `sudo snap install <name>` | Sandboxed, self-contained, larger downloads |

## System Identification

| Command | Shows |
|---|---|
| `uname -a` | Kernel name, hostname, kernel version, architecture |
| `lsb_release -a` | Distributor, release version, codename |
| `cat /etc/os-release` | Machine-parseable `KEY=value` distro info |
| `hostnamectl` | Combined hostname + OS + kernel + virtualization summary |

## System Monitoring

| Command | Shows | Key thing to check |
|---|---|---|
| `uptime` | Uptime, users, **load average** (1/5/15 min) | Divide by `nproc` core count |
| `nproc` | Number of CPU cores | Baseline for reading load average |
| `free -h` | RAM + swap, human-readable | Watch **available** & **Swap used**, not raw "used" |
| `df -h` | Disk space per mounted filesystem | Which filesystem is near 100% |
| `du -sh <dir>` | Total size of one directory | Drill down after `df -h` flags a filesystem |
| `du -sh /path/* \| sort -rh \| head` | Biggest subdirectories, sorted | Find the actual space-hog |
| `vmstat` | Live CPU/memory/IO snapshots (needs `sysstat`) | Deeper trend-watching over time |
| `iostat` | Per-disk I/O throughput/latency (needs `sysstat`) | Disk bottleneck vs. capacity issue |

### `free -h` Columns Explained

| Column | Meaning |
|---|---|
| `total` | Total physical RAM |
| `used` | Currently allocated (can look high but include reclaimable cache) |
| `free` | Completely untouched RAM (often misleadingly small — ignore in isolation) |
| `buff/cache` | Disk cache — reclaimable instantly when an app needs it; **not a problem** |
| `available` | **The number that matters** — realistic estimate of what a new process could get |
| `Swap used` | Disk-backed overflow RAM in active use — meaningful/growing usage = red flag |

### `df -h` Columns Explained

| Column | Meaning |
|---|---|
| `Filesystem` | The device/partition |
| `Size` | Total size of that filesystem |
| `Used` / `Avail` | Space used / still free |
| `Use%` | Percentage full — the number to watch, alert around 80-90%+ |
| `Mounted on` | Where in the directory tree it's mounted |

### Load Average Interpretation

| Load average | 4-core machine | 1-core machine |
|---|---|---|
| `1.0` | 25% of capacity — fine | 100% of capacity — fully busy, no queue |
| `4.0` | 100% of capacity — fully busy | 400% — 4x overloaded, heavy queuing |
| `8.0` | 200% — 2x overloaded, queuing | 800% — severely overloaded |

**Rule:** `load average ÷ nproc`. Result ≤ 1.0 = keeping up. Result >> 1.0 = queuing/overloaded.

## 🔁 The Safe System Update Workflow

Do this every time you patch a server:

1. **Refresh the index first** — `sudo apt update`. Never skip this.
2. **Review what's pending** — `apt list --upgradable`, so you know what's about to change.
3. **Apply updates** — `sudo apt upgrade` (add `-y` only for scripted/routine runs).
4. **Clean up afterward** — `sudo apt autoremove` then `sudo apt clean`.
5. **Verify** — spot-check that critical services still run as expected.

## 🔁 The "Is This Server Healthy" Diagnostic Workflow

Do this any time someone says "the server feels slow" or "we're out of disk space":

1. **Check CPU pressure** — `nproc` for core count, then `uptime` for load average. Divide the two.
2. **Check memory pressure** — `free -h`. Read `available`, not raw `used`. Check if `Swap` usage is meaningfully non-zero.
3. **Check disk space** — `df -h` across all mounted filesystems. Note anything near 90%+.
4. **Drill into the culprit** — `du -sh <suspect-dir>/* | sort -rh | head` to find exactly what's consuming space.
5. **Go deeper only if needed** — `top`/`htop` (Module 10) for the specific process; `vmstat`/`iostat` for sustained trend data.
6. **Remediate, don't just observe** — apply the Safe System Update Workflow above, clear stale logs/caches, or escalate if the fix is bigger than routine maintenance.
