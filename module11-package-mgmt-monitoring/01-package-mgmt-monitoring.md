# Module 11: Package Management & System Monitoring 🟢🟡

**Difficulty:** 🟢 Beginner / 🟡 Intermediate
**Estimated Time:** 2 hours
**Prerequisites:** Modules 1-10 (Shell Fundamentals through Process Management & Job Control)

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain what a package manager is and why it beats manually downloading and compiling software (dependency resolution, versioning, security updates)
- [ ] Explain the crucial difference between `apt update` (refresh the package index) and `apt upgrade` (actually install new versions), and never confuse the two again
- [ ] Install, search for, inspect, and remove packages with `apt install`, `apt search`, `apt show`, `apt remove`, `apt purge`, `apt autoremove`, and `apt list --installed`
- [ ] Use `dpkg` — the lower-level tool `apt` is built on — to list installed packages, list the files a package owns (`dpkg -L`), and find which package owns a given file (`dpkg -S`)
- [ ] Describe what a PPA and a Snap package are, why Ubuntu offers them alongside `apt`, and the trust/security trade-off of adding third-party sources
- [ ] Identify your system and kernel with `uname -a`, `lsb_release -a`, `/etc/os-release`, and `hostnamectl`
- [ ] Read `uptime` and correctly interpret load average numbers relative to your CPU core count
- [ ] Diagnose "why is this server slow" or "why is this server out of disk space" using `free -h`, `df -h`, and `du -sh`

---

## Module Goal

By the end of this module, you'll be able to keep a Linux server's software up to date safely, and diagnose its health — CPU load, memory, and disk space — the same way an operations engineer does during a normal Tuesday and during a 2 a.m. incident.

🎯 **On the job:** Picture two extremely common scenarios. First: you're responsible for a small fleet of Ubuntu servers, and a security bulletin just went out about a vulnerability in a library half of them use. You need to patch every box safely, without breaking anything that depends on an older version, and without pulling in unrelated risky changes at the same time. Second: someone messages you "the app server feels really slow" or "we can't write logs anymore, disk full." You SSH in with zero prior context and, in under two minutes, need to know: is the CPU pegged? Is memory exhausted and swapping? Is a disk actually full? Which directory ate all the space? This module gives you both skill sets — patch software responsibly, and read a server's vital signs on demand.

---

## Core Concepts

### 1. What is a package manager, and why not just download software manually?

A **package** is a bundle containing a piece of software — its compiled program files, configuration templates, documentation, and metadata about what else it needs to run. A **package manager** is a tool that installs, updates, configures, and removes these packages for you, while tracking exactly what's installed and what depends on what.

Before package managers, installing software on Linux meant downloading source code, compiling it yourself, and manually hunting down every library it needed (its **dependencies**) — often each with its *own* dependencies. Get one version wrong and the build fails, or worse, it "works" but breaks something else already installed.

A package manager solves three problems at once:

- **Dependency resolution** — if you install a package that needs three other libraries, the package manager figures that out and installs all three automatically, at compatible versions.
- **Versioning** — it tracks exactly which version of everything is installed, so you can update deliberately instead of guessing.
- **Security updates** — vulnerability patches for the entire software ecosystem you use flow through one consistent update mechanism, instead of you personally tracking dozens of individual projects' release pages.

💡 **Analogy — an app store with dependency tracking:** Think of a package manager like a phone's app store, but smarter. Installing "Photo Editor" on your phone doesn't ask you to separately go find and install some image-processing library it needs — the app store handles that silently. A Linux package manager does the same thing for system software: ask for `nginx` (a web server), and it quietns pulls in every library `nginx` needs to actually run, at versions known to work together, and it remembers all of it so it can cleanly remove or update later.

### 2. `apt` — Ubuntu/Debian's main package manager

**`apt`** ("Advanced Package Tool") is the primary command-line package manager on Ubuntu, Debian, and every Debian-based distribution. It talks to a network of **repositories** — servers hosting collections of packages — and handles downloading, dependency resolution, and installation for you.

⚠️ **Distro awareness:** This module focuses on `apt` because it's what Ubuntu/Debian use. If you ever work on a Red Hat, Fedora, CentOS, or Rocky/AlmaLinux server, the equivalent tools are `yum` (older) or `dnf` (its modern replacement) — same *concept* (repositories, dependency resolution, install/remove/update), different command syntax. Knowing `apt` well transfers conceptually; you'd just need to learn the RHEL-family command names when the day comes.

### 3. The index vs. the packages themselves

This is the single most important mental model in this whole module, so read it twice.

Your system keeps a local **package index** — essentially a cached list of "here's every package available in each configured repository, and which version is currently the latest." That index is just a local file cache; it is **not** the software itself, and it does **not** update automatically just because a new version was released somewhere upstream.

- **`sudo apt update`** — refreshes that local index by contacting the repositories and downloading the latest list of available packages and their version numbers. **It installs nothing.** It changes zero software on your machine. It only updates your system's *knowledge* of what's available.
- **`sudo apt upgrade`** — looks at that (now-refreshed) index, compares it against what's actually installed, and downloads + installs newer versions of anything that has one.

💡 **Analogy — a store catalog vs. actually shopping:** `apt update` is like refreshing a store's paper catalog to see this week's stock and prices. `apt upgrade` is you actually walking in and buying the items that are now on a newer/better version than what you have at home. Flipping through the catalog (`update`) never puts anything in your cart — you still have to go shopping (`upgrade`) afterward.

⚠️ **The single most common beginner mistake in this entire module:** Running `apt upgrade` without running `apt update` first. If you skip `update`, `upgrade` is working from a stale, possibly days- or weeks-old index — it might install nothing when patches are actually available, or it might miss a critical security fix entirely because your system doesn't even know it exists yet. **Always run `apt update` immediately before `apt upgrade`.** This is so standard that most people chain them: `sudo apt update && sudo apt upgrade`.

### 4. Installing, searching, and inspecting packages

- **`sudo apt install <package>`** — installs a package (and any dependencies it needs) using whatever version is currently in your local index.
- **`apt search <keyword>`** — searches package *names and short descriptions* in the index for a keyword — useful when you know roughly what you want ("a text editor") but not the exact package name.
- **`apt show <package>`** — shows detailed metadata about a specific package: its version, size, dependencies, maintainer, and a longer description — without installing it. Great for confirming "is this really the package I think it is" before installing.
- **`apt list --installed`** — lists every package currently installed on the system, with its version.

### 5. Removing packages: `remove` vs. `purge`, and `autoremove`

This is the second core distinction this module drills into you.

- **`sudo apt remove <package>`** — uninstalls the package's program files, but **deliberately leaves behind its configuration files** (typically under `/etc/`). The reasoning: if you reinstall the same package later, your old settings are still sitting there waiting for you.
- **`sudo apt purge <package>`** — uninstalls the package **and** deletes its configuration files too. Use this when you genuinely never want a trace of that software's settings left behind — for example, before installing a completely fresh, default configuration of the same package.
- **`sudo apt autoremove`** — cleans up dependency packages that were pulled in automatically for something else, but are now orphaned because nothing installed still needs them. Run this periodically (and especially after a `remove`/`purge`) to avoid slowly accumulating unused cruft.

💡 **Analogy:** `remove` is like uninstalling an app from your phone but leaving your saved preferences file behind in case you reinstall it later. `purge` is a full factory-reset of that one app — settings and all, gone. `autoremove` is emptying the "downloaded for that app but not used by anything else" leftovers afterward.

### 6. `dpkg` — the lower-level tool underneath `apt`

**`dpkg`** ("Debian package") is the lower-level tool that actually installs and tracks packages on disk — `apt` is a friendlier layer built *on top of* `dpkg` that adds repository management and automatic dependency resolution. You rarely need `dpkg` to install things directly (that's what `apt` is for), but it's the right tool for two specific inspection tasks `apt` doesn't do as directly:

- **`dpkg -l`** — lists all packages `dpkg` knows about and their status (a terser, lower-level cousin of `apt list --installed`).
- **`dpkg -L <package>`** — lists every file a specific installed package put on your filesystem. Useful when you want to know exactly what a package touched.
- **`dpkg -S <file>`** — the reverse lookup: given a file path, tells you which installed package owns it. Extremely useful when you find a mystery file or binary and want to know "what package is this actually part of?"

🎯 **On the job:** You find a config file, say `/etc/nginx/nginx.conf`, and want to know exactly which package installed it and what other files came with it. `dpkg -S /etc/nginx/nginx.conf` tells you the owning package; `dpkg -L nginx` then shows you every other file that package placed on the system — man pages, binaries, default configs, everything.

### 7. Alternative distribution channels: PPAs and Snap

`apt`'s official Ubuntu repositories are curated and reviewed, but sometimes you need software that isn't in them, or you need a newer version than Ubuntu's stable release ships.

- **PPA (Personal Package Archive)** — a repository hosted by an individual or team (commonly on Launchpad), added to your system with `sudo add-apt-repository ppa:<name>/<ppa-name>`, after which it behaves just like any other `apt` repository — you still `apt update` and `apt install` from it. PPAs let you get newer or specialized software `apt`'s default repos don't carry.
- **Snap** — a completely separate packaging system that ships pre-installed on modern Ubuntu. Snap packages (called "snaps") bundle an application together with all its dependencies in a self-contained, sandboxed format, installed with `sudo snap install <name>`. This avoids dependency conflicts with the rest of your system, at the cost of larger download sizes and, for some snaps, slower startup.

⚠️ **Warning — trust matters here:** Ubuntu's official repositories are vetted by Canonical/Debian maintainers. A PPA is run by whoever created it — anyone can publish one. Adding a PPA means trusting that person's build pipeline with root-level software on your machine. Only add PPAs from sources you actually trust (well-known projects, recognized maintainers), and never blindly copy-paste `add-apt-repository` commands from random forum posts without checking what you're actually adding. The same caution applies doubly to piping random install scripts straight into a shell (covered in Common Pitfalls, below) — a compromised or malicious script has the same access you do, including `sudo`.

### 8. Identifying your system

Before troubleshooting anything, it helps to know exactly what you're troubleshooting. Four commands answer "what is this machine, exactly?":

- **`uname -a`** — prints kernel information: kernel name, hostname, kernel release/version, architecture. Useful for confirming exactly which kernel build you're on.
- **`lsb_release -a`** — prints Linux Standard Base info: distributor (e.g. Ubuntu), release version, codename (e.g. "jammy"). (May need `sudo apt install lsb-release` on minimal images.)
- **`/etc/os-release`** — a plain text file (view with `cat`) containing the same kind of distro identification, in a machine-parseable `KEY=value` format — scripts often read this file directly instead of shelling out to `lsb_release`.
- **`hostnamectl`** — shows the system's hostname alongside OS, kernel, and virtualization details in one combined summary; on `systemd`-based systems (which you met in Module 10) this is often the single fastest overview command.

### 9. `uptime` and load average

**`uptime`** shows how long the system has been running, how many users are logged in, and — most importantly for diagnostics — the **load average**: three numbers representing the average number of processes wanting CPU time, averaged over the last 1, 5, and 15 minutes respectively.

Here's the part that trips people up: **a load average number is not a percentage, and it means nothing on its own — it only makes sense relative to how many CPU cores the machine has.**

- A load average of `1.00` on a **1-core** machine means that core was, on average, fully busy — 100% utilized, with nothing idle but also nothing queuing up waiting.
- That same `1.00` on a **4-core** machine means only about a quarter of the available CPU capacity was in use — plenty of headroom.
- A load average of `8.00` on a 4-core machine means, on average, twice as many processes wanted CPU time as the machine had cores available — processes were genuinely queuing and waiting their turn.

💡 **The rule of thumb:** Divide the load average by your number of CPU cores. A result **at or below 1.0** means the system is keeping up. A result **well above 1.0** means processes are queuing for CPU time — the system is overloaded relative to its capacity.

The three numbers (1/5/15-minute averages) also tell a story by themselves: if the 1-minute number is much higher than the 15-minute number, load just spiked recently. If all three are climbing together, load has been building steadily — a trend worth investigating before it gets worse.

🎯 **On the job:** find your core count with `nproc`, then run `uptime`. A load average of `6.00` sounds alarming in isolation — but on a 16-core box it's nothing to worry about, while that same `6.00` on a 2-core box is a genuine, active problem.

### 10. Memory: `free -h`

**`free -h`** shows memory usage in human-readable units (the `-h` flag — same idea as `ls -h` and `df -h` elsewhere). Key columns:

- **Mem (total/used/free/shared/buff-cache/available)** — physical RAM.
- **Swap (total/used/free)** — disk-backed overflow space used when RAM is exhausted.

The two columns that confuse almost everyone: **`buff/cache`** and **`available`**.

- **`buff/cache`** is memory the kernel is using to cache recently-read disk data and filesystem metadata, purely for performance — reading the same file twice is much faster if it's still in the page cache from last time. Critically, this memory is **not locked away** — the kernel will instantly reclaim it and hand it to an application the moment that application actually needs it.
- **`available`** is the number that actually matters when asking "how much memory can a new process realistically get right now?" — it accounts for that reclaimable cache, so it's usually a much healthier, more honest number than "free" (which only counts memory that's *completely* untouched, and looks artificially low on a system that's been running a while).

✅ **Best Practice:** Don't panic at a small "free" number — a Linux box that's been up for a while will naturally show most of its RAM absorbed into `buff/cache`, and that's a sign of a healthy system using its resources well, not a problem. Look at **`available`**, and watch the **swap** row — meaningful, growing swap *usage* (not just swap existing) is the real memory-pressure red flag, since it means the system ran out of actual RAM and started spilling to much-slower disk.

### 11. Disk space: `df -h` and `du -sh`

- **`df -h`** ("disk free") shows space used/available **per mounted filesystem** — e.g. your root filesystem `/`, a separate `/boot` partition, a mounted data volume. This answers "which filesystem is running out of room?"
- **`du -sh <directory>`** ("disk usage") shows the total size of a specific directory (and everything inside it) — the `-s` flag summarizes to one total instead of listing every subdirectory, and `-h` again means human-readable. This answers "which directory is actually eating the space `df` just told me is running low?"

🎯 **On the job:** `df -h` tells you `/var` is 95% full. It does *not* tell you why. That's when you `du -sh /var/*` (or drill deeper directory by directory) to hunt down the actual culprit — commonly runaway log files, old Docker images, or a forgotten backup directory.

### 12. Deeper monitoring tools (brief mention)

`uptime`, `free -h`, and `df -h` cover the vast majority of "is this server okay" questions. When you need more depth — moment-to-moment CPU/memory/swap activity, or per-disk I/O throughput and latency — two tools from the `sysstat` package go further:

- **`vmstat`** — reports virtual memory, process, CPU, and I/O statistics as a repeating stream of snapshots, useful for watching trends over a short window live.
- **`iostat`** — reports per-device disk I/O statistics (throughput, wait times), useful when `df` shows plenty of free space but something still feels slow — the disk itself might be the bottleneck, not its capacity.

```bash
sudo apt install sysstat
```

You won't need these for most day-to-day checks, but knowing they exist means you have somewhere deeper to go when `uptime`/`free`/`df` aren't enough to explain what you're seeing.

---

## Detailed Explanations

### Why `apt update` and `apt upgrade` are two separate commands, not one

It would be simpler, in theory, for `apt` to just have one "get me the latest software" command. It's deliberately split in two because they answer genuinely different questions with genuinely different levels of risk: *"what's available?"* (`update`) is a completely safe, read-only, no-changes operation you can run as often as you like. *"Actually change my installed software"* (`upgrade`) is a real, potentially disruptive action — installing new binaries, restarting services, possibly changing behavior. Keeping them separate means you can refresh your knowledge of what's out there constantly and cheaply, while deliberately choosing the moment you actually let changes happen — valuable when, say, you want to check what a security patch would change before committing a whole fleet of production servers to it.

### Why load average needs to be divided by core count to mean anything

A single CPU core can only genuinely execute one thing at a time. Load average measures, roughly, "how many processes wanted a share of CPU time, on average, over this window" — including ones that got a turn and ones still waiting their turn. On a machine with only one core, any load average above `1.0` directly means processes were waiting in a queue for their turn, because there was only one line to stand in. On a machine with eight cores, there are effectively eight parallel queues — a load average of `6.0` there means those eight lines, on average, only had six people total across all of them: comfortably under capacity. The raw number by itself carries no information about how much capacity exists to serve it — that's exactly why it must always be read next to `nproc`'s core count, never alone.

### Why `buff/cache` memory isn't really "used" the way you'd assume

Modern Linux treats unused RAM as wasted opportunity — it would rather use idle memory to cache disk reads (speeding up future access to the same files) than let it sit empty doing nothing. That cached memory sits in a middle state: it's currently holding useful data, but it is instantly, cheaply reclaimable the moment a real application asks for memory the kernel doesn't have anywhere else to give. This is why `free`'s raw "free" column can look alarmingly low on a perfectly healthy, long-uptime server — most of what looks "used" is actually this soft, reclaimable cache — and why the `available` column (which accounts for that reclaimability) is the number worth actually paying attention to.

---

## Practical Examples

### Example 1 — The safe update/upgrade/install/remove sequence

```bash
sudo apt update
```

Realistic output:
```
Hit:1 http://archive.ubuntu.com/ubuntu jammy InRelease
Get:2 http://archive.ubuntu.com/ubuntu jammy-updates InRelease [119 kB]
Get:3 http://archive.ubuntu.com/ubuntu jammy-security InRelease [110 kB]
Fetched 229 kB in 1s (198 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
14 packages can be upgraded. Run 'apt list --upgradable' to see them.
```

```bash
apt list --upgradable
```
```
Listing... Done
curl/jammy-updates 7.81.0-1ubuntu1.15 amd64 [upgradable from: 7.81.0-1ubuntu1.14]
openssl/jammy-updates 3.0.2-0ubuntu1.12 amd64 [upgradable from: 3.0.2-0ubuntu1.11]
...
```

```bash
sudo apt upgrade
```
```
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Calculating upgrade... Done
The following packages will be upgraded:
  curl libcurl4 openssl ...
14 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
Do you want to continue? [Y/n] y
...
Setting up openssl (3.0.2-0ubuntu1.12) ...
```

```bash
sudo apt install tree
```
```
Reading package lists... Done
Building dependency tree... Done
The following NEW packages will be installed:
  tree
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 47.1 kB of archives.
...
Setting up tree (2.0.2-1) ...
```

```bash
sudo apt remove tree
```
```
The following packages will be REMOVED:
  tree
0 upgraded, 0 newly installed, 1 to remove and 0 not upgraded.
...
```

```bash
sudo apt autoremove
```
```
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

Line-by-line:
- `apt update` contacts the configured repositories (`archive.ubuntu.com` here) and refreshes the local index — notice it explicitly reports "14 packages can be upgraded" but installs nothing itself.
- `apt list --upgradable` shows exactly which packages have a newer version waiting, and what version you're moving from/to — a good habit before blindly upgrading everything.
- `apt upgrade` is the step that actually downloads and installs those 14 updates, and it asks for confirmation (`[Y/n]`) before touching anything.
- `apt install tree` pulls in a small, harmless utility (`tree`, which prints directory structures visually) as an example of installing something new.
- `apt remove tree` uninstalls `tree`'s program files but (per Concept 5) would leave any config behind if `tree` had any.
- `apt autoremove` checks for now-orphaned dependencies; here there are none, so it reports nothing to do — normal and expected most of the time.

💡 **Tip:** Chain the first two steps together as a habit: `sudo apt update && sudo apt upgrade -y` (the `-y` auto-confirms — use it in scripts, but consider leaving it off interactively the first few times so you get used to reading what's about to change).

### Example 2 — `remove` vs. `purge`, seeing the actual difference

```bash
sudo apt install nginx-light
```
```
...
Setting up nginx-light (1.18.0-6ubuntu14.4) ...
```

```bash
sudo apt remove nginx-light
```
```
The following packages will be REMOVED:
  nginx-light
0 upgraded, 0 newly installed, 1 to remove and 0 not upgraded.
...
```

```bash
ls /etc/nginx/
```
```
mime.types  modules-enabled  nginx.conf  sites-available  sites-enabled  ...
```

```bash
sudo apt purge nginx-light
```
```
The following packages will be REMOVED:
  nginx-light*
0 upgraded, 0 newly installed, 1 to remove and 0 not upgraded.
...
(Reading database ... files and directories currently installed.)
Purging configuration files for nginx-light (1.18.0-6ubuntu14.4) ...
```

```bash
ls /etc/nginx/ 2>&1
```
```
ls: cannot access '/etc/nginx/': No such file or directory
```

Line-by-line:
- After `apt remove nginx-light`, the program itself is gone, but `/etc/nginx/` — its configuration directory — is still sitting there fully intact, exactly as Concept 5 explained.
- `apt purge nginx-light` (note the `*` next to the package name in its output, `apt`'s own visual marker that this is a purge, not a plain remove) goes the extra step and deletes `/etc/nginx/` entirely — confirmed by the follow-up `ls` failing with "No such file or directory."

⚠️ **Warning:** If you plan to reinstall the same package with the same custom configuration later, `remove` (not `purge`) preserves that work for you. Only `purge` when you genuinely want a clean slate.

### Example 3 — Finding which package owns a file with `dpkg -S`

```bash
dpkg -S /bin/ls
```
```
coreutils: /bin/ls
```

```bash
dpkg -L coreutils | head -8
```
```
/.
/usr
/usr/bin
/usr/bin/ls
/usr/bin/cat
/usr/bin/cp
/usr/bin/mv
/usr/bin/rm
```

Line-by-line:
- `dpkg -S /bin/ls` answers "who owns this file?" — here, the humble `ls` command turns out to belong to the `coreutils` package, the bundle of fundamental Linux utilities you met back in Module 1.
- `dpkg -L coreutils` then flips the question around — "what else did this package install?" — listing every file `coreutils` placed on the filesystem, confirming `ls` is just one of many core tools bundled together in it.

🎯 **On the job:** You stumble across a binary or config file you don't recognize on a server you've inherited. `dpkg -S <path>` instantly tells you what package it belongs to, so you can look that package up (`apt show`) instead of guessing or, worse, deleting something you shouldn't have.

### Example 4 — Diagnosing a slow, disk-tight server: `free -h` + `df -h` + `du -sh`

```bash
free -h
```
```
               total        used        free      shared  buff/cache   available
Mem:            7.6Gi       2.1Gi       412Mi        89Mi       5.1Gi       5.2Gi
Swap:           2.0Gi       1.3Gi       712Mi
```

```bash
df -h
```
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        40G   38G  1.2G   97% /
/dev/sdb1       200G  140G   50G   70% /data
tmpfs           3.8G     0  3.8G    0% /dev/shm
```

```bash
du -sh /var/log/* | sort -rh | head -5
```
```
2.3G    /var/log/journal
1.1G    /var/log/nginx
340M    /var/log/apt
88M     /var/log/syslog
12M     /var/log/dpkg.log
```

Line-by-line:
- `free -h`: total RAM is 7.6 GiB. `buff/cache` (5.1 GiB) is large but reclaimable (Concept 10) — the number to actually worry about is `available`, which at 5.2 GiB is healthy. The real red flag here is **`Swap: used 1.3Gi`** — real, meaningful swap usage means this system genuinely ran out of physical RAM at some point and had to spill to disk, which is drastically slower — a strong lead for "why does this feel sluggish."
- `df -h`: the root filesystem `/dev/sda1` is at **97% used**, with only 1.2 GiB free — this is the "out of disk space" half of the problem, and it's specifically the `/` filesystem, not `/data` (which is fine at 70%).
- `du -sh /var/log/* | sort -rh | head -5` drills into the likely offender directory (`/var/log`, a classic space-eater) and sorts by size descending (`sort -rh` — reverse, human-numeric-aware sort) — revealing `/var/log/journal` (systemd's own logging, met in Module 10) as the single biggest consumer at 2.3 GiB.

✅ **Best Practice:** Always pair `df -h` (which filesystem is full) with `du -sh` on the likely directories (what's actually filling it) — `df` alone never tells you *what* to delete, only *where* the problem is.

### Example 5 — Interpreting load average against core count

```bash
nproc
```
```
4
```

```bash
uptime
```
```
14:32:07 up 21 days,  4:12,  3 users,  load average: 7.85, 6.20, 3.10
```

Line-by-line:
- `nproc` reports this machine has **4 CPU cores**.
- `uptime`'s load average shows `7.85` (last 1 min), `6.20` (last 5 min), `3.10` (last 15 min).
- Applying the rule from Concept 9: divide each by 4 cores → roughly `1.96`, `1.55`, `0.78`. The 1-minute figure is nearly **double** the machine's capacity right now, the 5-minute figure is also over capacity, and the 15-minute figure (0.78) shows the system was comfortably under capacity a little while ago.
- **Interpretation:** this is a load spike that started recently and is actively getting worse (1-min > 5-min > 15-min, all rising) — this machine, right now, has more work queued up than its 4 cores can keep up with. This is exactly the moment to reach back to Module 10's tools (`ps aux --sort=-%cpu`, `top`) to find out *which* process is driving it.

---

## Common Pitfalls & Best Practices

- **Running `apt upgrade` without `apt update` first.** As covered in Concept 3, `upgrade` only ever acts on your local, possibly stale index. Always run `sudo apt update && sudo apt upgrade` together, in that order — never `upgrade` alone if you haven't refreshed the index recently.
- **Confusing `remove` with `purge`.** Reach for `remove` when you might reinstall the same package later and want to keep its configuration; reach for `purge` when you genuinely want the configuration gone too. Purging by habit, without thinking about it, has cost people carefully hand-tuned config files they assumed "uninstall" would leave alone.
- **Piping a random install script straight into `sudo bash`.** You'll see instructions online like `curl https://example.com/install.sh | sudo bash`. This runs a script you have not read, with full root privileges, straight from the network — if that server is compromised, or the script is malicious, or it's simply just been silently updated since someone recommended it, you've handed over your entire machine sight-unseen. ✅ **Better:** download the script first (`curl -O https://example.com/install.sh`), actually read it, and only then run it — or better yet, prefer official `apt`/PPA/Snap packages whenever one exists.
- **Ignoring disk space until it's critical.** A filesystem that quietly crept from 80% to 90% to 97% full rarely announces itself until something actually fails — a database that can't write, a log that can't rotate, a deploy that can't unpack. ✅ **Best Practice:** check `df -h` periodically (or better, have monitoring alert on a threshold, previewed in a later module) rather than waiting for something to break first.
- **Adding PPAs or installing Snaps without checking who publishes them.** Concept 7's warning bears repeating here as a habit, not just a fact: verify the publisher of a PPA or snap before trusting it with system-level access, the same way you'd think twice before installing an unfamiliar app on your phone.
- **Reading "used" memory in `free -h` at face value.** A large "used" number that's actually mostly `buff/cache` is not a problem — it's Linux using spare RAM productively. Read `available` and the `Swap` row instead of panicking at "used."
- **Reading load average without checking core count first.** A load average of `4.0` is either "totally fine" (on a 16-core box) or "seriously overloaded" (on a single-core box) — the number is meaningless without `nproc` next to it.

✅ **Best Practice — The "confirm before you act" habit:** Before installing (`apt show` first), before removing (double-check `remove` vs. `purge` is really what you want), and before running any install script from the internet (read it first) — pause and verify. It costs seconds and prevents the expensive kind of mistake.

---

## Hands-on Exercise

**Task — "This server feels slow and we're getting disk-space warnings":**

1. Check how many CPU cores the machine has, then check the current load average and interpret it relative to that core count.
2. Check memory usage and determine whether the system is under real memory pressure (hint: don't just look at "used").
3. Check disk space across all mounted filesystems and identify which one is critically full.
4. Drill into the most-likely directory to find out what's actually consuming the space.
5. Once you've diagnosed the issue, safely refresh and apply any pending package updates, then clean up any orphaned packages — as routine maintenance that's often overdue on a neglected server.

Try this yourself in a real terminal (or reason through it based on the example outputs above) before reading the solution.

### Solution

```bash
# 1. Core count and load average
nproc
```
```
4
```
```bash
uptime
```
```
15:10:44 up 45 days,  9:02,  2 users,  load average: 5.40, 5.10, 4.95
```
*Interpretation: 5.40 / 4 cores ≈ 1.35 — the system has been running noticeably over capacity, consistently (all three windows are close together and all above 1x core count), not just a brief spike. This alone explains "feels slow."*

```bash
# 2. Memory pressure check
free -h
```
```
               total        used        free      shared  buff/cache   available
Mem:            3.8Gi       1.2Gi       180Mi        45Mi       2.4Gi       2.3Gi
Swap:           1.0Gi       640Mi       384Mi
```
*Interpretation: `available` (2.3Gi) is actually reasonably healthy, so this isn't primarily a memory story — but `Swap: used 640Mi` is non-trivial and worth keeping an eye on if it keeps climbing.*

```bash
# 3. Disk space per filesystem
df -h
```
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        20G   19G  512M   98%  /
tmpfs           1.9G     0  1.9G    0% /dev/shm
```
*Interpretation: root filesystem `/` is at 98% — this is the "disk-space warnings" half of the ticket, and it's urgent.*

```bash
# 4. Find what's actually filling it
du -sh /var/log/* /var/cache/* 2>/dev/null | sort -rh | head -5
```
```
1.8G    /var/log/journal
610M    /var/cache/apt/archives
300M    /var/log/nginx
90M     /var/log/syslog
40M     /var/log/dpkg.log
```
*Interpretation: `/var/log/journal` and a large `/var/cache/apt/archives` (old downloaded `.deb` package files `apt` doesn't automatically delete) are the two biggest, safest cleanup targets.*

```bash
# 5. Safe update, upgrade, and cleanup
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
sudo apt clean
```
```
...
14 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
...
Removing unused dependencies...
0 upgraded, 0 newly installed, 3 to remove and 0 not upgraded.
```

Explanation: I never guessed — every conclusion came from a command's actual output. `nproc` + `uptime` together (not `uptime` alone) confirmed the load really was high relative to this machine's specific capacity, and that it was sustained rather than a momentary spike. `free -h`'s `available` figure ruled memory *out* as the primary cause, while flagging swap usage as something to watch. `df -h` pinpointed exactly which filesystem was actually critical (98% on `/`, not just "disk usage exists somewhere"), and `du -sh` on the likely directories found the concrete, actionable culprits instead of guessing at what to delete. Only after diagnosing did I move to remediation: `apt update` before `apt upgrade` (never the other way around), `-y` to auto-confirm since this is routine maintenance, `autoremove` to clear orphaned dependencies, and `apt clean` to clear out `/var/cache/apt/archives` — the downloaded package cache found in step 4, which is always safe to delete since `apt` can re-download those files if it ever needs them again.

✅ Exercise complete — you diagnosed both a CPU-load complaint and a disk-space warning using only read-only commands first, then applied a safe, standard update-and-cleanup sequence once the diagnosis was clear.

---

## ✅ Module Completion Checklist

- [ ] I can explain what a package manager is and why it beats manually downloading and compiling software (dependency resolution, versioning, security updates)
- [ ] I can explain the crucial difference between `apt update` (refresh the package index) and `apt upgrade` (actually install new versions)
- [ ] I can install, search for, inspect, and remove packages with `apt install`, `apt search`, `apt show`, `apt remove`, `apt purge`, `apt autoremove`, and `apt list --installed`
- [ ] I can use `dpkg -l`, `dpkg -L`, and `dpkg -S` to inspect installed packages and the files they own
- [ ] I can describe what a PPA and a Snap package are, and why third-party sources require extra trust
- [ ] I can identify a system with `uname -a`, `lsb_release -a`, `/etc/os-release`, and `hostnamectl`
- [ ] I can read `uptime` and correctly interpret load average numbers relative to CPU core count
- [ ] I can diagnose a slow or disk-full server using `free -h`, `df -h`, and `du -sh`

## Next Step

Continue to [Module 12: Networking Basics](../module12-networking-basics/)
