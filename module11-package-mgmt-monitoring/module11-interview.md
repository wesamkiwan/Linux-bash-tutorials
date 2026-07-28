# 🎤 Module 11 Interview Prep — Package Management & System Monitoring

## Conceptual Questions

### 🟢 Beginner

**Q1: What is a package manager, and why not just download and compile software yourself?**

> "A package manager installs, updates, and removes software for you while automatically handling everything that software needs to actually run — its dependencies. Before package managers, you'd have to download source code, compile it, and manually track down every library it needed, often each with dependencies of its own — one wrong version and the build breaks, or it 'works' but destabilizes something else already installed. A package manager solves three things at once: dependency resolution, so it figures out and installs everything a package needs automatically; versioning, so you always know exactly what's installed; and security updates, so patches for your entire software stack flow through one consistent, trustworthy channel instead of you personally watching dozens of project release pages."

**Q2: What's the difference between `apt update` and `apt upgrade`? This trips a lot of people up.**

> "`apt update` refreshes your system's local package index — it contacts the repositories and downloads the current list of available packages and their versions. It installs nothing and changes no software at all; it only updates what your system *knows* is available. `apt upgrade` is the step that actually installs newer versions of anything installed, based on that index. The critical thing is `upgrade` only ever works off whatever the index currently says — if you never ran `update`, you could be upgrading against a stale, days-old picture of what's available, potentially missing real, available security patches. That's why the standard habit is always `sudo apt update && sudo apt upgrade` together, never `upgrade` alone."

**Q3: What does `apt install <package>` actually do behind the scenes?**

> "It looks up the package in the local index, works out its full dependency tree — everything else it needs that isn't already installed — downloads all of that from the configured repositories, and installs everything in the right order. You just ask for the one package by name; the dependency resolution happens automatically."

**Q4: How do you find out what a package is before installing it?**

> "`apt show <package>` gives you its version, size, dependency list, maintainer, and a fuller description, without installing anything. If I don't know the exact package name, I'd start with `apt search <keyword>` to search names and descriptions for something matching what I'm after."

**Q5: What does `uname -a` tell you, and how is it different from `lsb_release -a`?**

> "`uname -a` reports kernel-level information — kernel name, hostname, kernel release and version, and CPU architecture. `lsb_release -a` reports distribution-level information instead — which distro you're on, like Ubuntu, its release version, and its codename. They answer different questions: `uname` is 'what kernel is this,' `lsb_release` is 'what distro and release is this.'"

### 🟡 Intermediate

**Q6: Explain `apt remove` vs. `apt purge` — what's actually different, and when do you use each?**

> "`apt remove` uninstalls a package's program files but deliberately leaves its configuration files behind, usually under `/etc/`. The idea is that if you reinstall the same package later, your old settings are still there waiting for you. `apt purge` goes further and deletes those configuration files too, for a genuinely clean slate. I use `remove` by default, especially if there's any chance I'll reinstall the same software later and want to keep a hand-tuned config. I reach for `purge` specifically when I want zero trace left — for example, right before reinstalling the same package fresh with all-default settings, or when decommissioning something for good."

**Q7: How do you interpret load average? Walk me through it.**

> "Load average, from `uptime`, is three numbers — the average number of processes wanting CPU time over the last 1, 5, and 15 minutes. The number by itself means nothing without knowing how many CPU cores the machine has, since a single core can only actually run one thing at a time. I get the core count with `nproc`, then divide the load average by that number. A result at or below 1.0 means the system is keeping up with demand; a result well above 1.0 means processes are genuinely queuing for CPU time they can't immediately get. I also look at the trend across the three windows — if the 1-minute number is much higher than the 15-minute number, load spiked recently; if all three are elevated and climbing together, it's a sustained, worsening problem, not a brief blip."

**Q8: What's the difference between `apt` and `dpkg`? Why do both exist?**

> "`dpkg` is the lower-level tool that actually installs packages and tracks what's on disk — it's the foundation. `apt` is a friendlier layer built on top of it that adds repository management and automatic dependency resolution, which `dpkg` alone doesn't do — if you tried to install a `.deb` file directly with `dpkg` and it needed a library you didn't have, `dpkg` would just fail and tell you what's missing, rather than fetching it for you. In practice I use `apt` for almost everything — installing, updating, removing — and drop down to `dpkg` for two specific inspection tasks: `dpkg -L <package>` to see every file a package installed, and `dpkg -S <file>` to find which package owns a given file. Those two lookups are things `apt` doesn't do as directly."

**Q9: In `free -h` output, why might the "used" memory number look alarmingly high even on a perfectly healthy server?**

> "Because Linux treats idle RAM as wasted opportunity — it uses spare memory to cache recently-read disk data, shown in the `buff/cache` column, purely for performance. That cache memory looks 'used' in a naive reading, but it's not locked away at all — the kernel reclaims it instantly the moment an actual application needs memory it doesn't have elsewhere. On a server that's been running a while, most memory naturally ends up absorbed into cache, which is a sign of healthy resource use, not a problem. The number that actually reflects what's realistically available to a new process is the `available` column, which already accounts for that reclaimable cache — that's the one I actually watch, along with whether `Swap` usage is meaningfully non-zero and growing, which is the real memory-pressure warning sign."

**Q10: A colleague adds a PPA and installs something from it without telling you. What's your concern?**

> "Ubuntu's official repositories are vetted by Canonical and Debian maintainers, but a PPA is published by whoever created it — anyone can host one. Adding a PPA means trusting that individual or team's build pipeline with root-level access to install software on the machine. My concern isn't that PPAs are inherently bad — plenty are legitimate and widely trusted — it's that adding one without checking who publishes it and why skips the trust evaluation that official repos already did for you. I'd want to know: is this a well-known, recognized maintainer, is there a legitimate reason we needed something outside the default repos, and is it actually still needed, since PPAs also need to be tracked and eventually cleaned up like any other dependency."

### 🔴 Advanced

**Q11: You're asked to safely patch a fleet of production servers for a security vulnerability. Walk me through your approach.**

> "First, I'd confirm exactly which package and version the fix applies to, rather than blanket-upgrading everything blindly. On each server (or ideally via a controlled rollout, one batch at a time rather than the whole fleet simultaneously), I'd run `sudo apt update` to refresh the index, then check `apt list --upgradable` or `apt show <package>` specifically to confirm the fixed version is actually available before proceeding. I'd apply the upgrade for that specific package, or run a full `apt upgrade` if the security bulletin affects multiple packages, then verify the fix actually landed — checking the installed version against what the bulletin specifies. I'd roll this out in batches rather than to every server at once, so if something unexpected breaks — a service that depended on old behavior — the blast radius is contained and I can catch it before it hits the whole fleet. Finally, I'd document what was patched, when, and confirm monitoring shows affected services came back up healthy afterward."

**Q12: `df -h` shows a filesystem at 97% full, but `du -sh` on every visible directory under it only adds up to a fraction of that. What's likely going on, and how do you investigate further?**

> "A classic cause is a large file that's been deleted but is still held open by a running process — on Linux, deleting a file only removes its directory entry; if a process still has it open, the disk space isn't actually released until that process closes the file or exits, so `du` (which walks the directory tree) won't see it at all, while `df` (which asks the filesystem directly how much space is actually allocated) still reports it as used. I'd look for processes with large deleted-but-open file handles — for example, checking `lsof` for entries marked `(deleted)` — often a long-running service that's been writing to a huge log file that got deleted out from under it without the process being restarted or told to reopen it. The fix is usually restarting that process (or asking it to reopen its log handles) so the kernel can finally reclaim the space."

**Q13: How would you explain to a junior engineer why `curl https://something.sh | sudo bash` is risky, without just saying 'don't do that'?**

> "I'd walk through what actually happens: that command downloads a script from a remote server and immediately executes it with full root privileges, without you ever seeing what's in it first. You're trusting three things simultaneously and blindly: that the server hasn't been compromised, that the script's author has no malicious intent, and that the script hasn't changed since whoever recommended it last checked it — since it fetches live, it could be a completely different script tomorrow than what you tested today. Compare that to the `apt`/PPA/Snap path: those go through a package format with a maintainer, a version you can pin, and in the case of official repos, actual review. My advice isn't just 'never run install scripts' — sometimes there's no packaged alternative — it's: download the script first, actually read it, and only then decide whether to run it, rather than piping it straight into a root shell sight-unseen."

---

## Practical/Coding Questions

**Q1: Write the command sequence to safely update all packages on an Ubuntu server, then clean up afterward.**

Solution:
```bash
sudo apt update
apt list --upgradable
sudo apt upgrade -y
sudo apt autoremove -y
sudo apt clean
```
Explanation: `update` refreshes the index first — mandatory before trusting `upgrade`. `apt list --upgradable` is a quick sanity check of what's about to change (skip it in a fully unattended script, but good practice interactively). `upgrade -y` applies the updates, auto-confirming since this is routine maintenance. `autoremove` clears now-orphaned dependency packages, and `clean` clears the downloaded `.deb` cache in `/var/cache/apt/archives`, which is always safe since `apt` can re-fetch those files if ever needed again.

**Q2: A file `/usr/local/bin/mystery-script` exists on a server you just inherited and nobody knows what installed it. Write the command to find out.**

Solution:
```bash
dpkg -S /usr/local/bin/mystery-script
```
Explanation: `dpkg -S <file>` performs the reverse lookup — given a file path, it reports which installed package owns it. If it returns nothing, that's itself informative: the file wasn't installed by any tracked package at all (it was placed there manually, by a script, or by something outside `apt`/`dpkg` entirely), which is a different — and often more concerning — situation than an untracked file that turns out to belong to a known package.

**Q3: Write the commands to determine whether a server's load average genuinely indicates a CPU capacity problem right now.**

Solution:
```bash
nproc
uptime
```
Explanation: `nproc` establishes the baseline — how many CPU cores this specific machine actually has. `uptime`'s load average is meaningless without that context. Divide each of the three load-average numbers by the core count from `nproc`: a result at or below roughly 1.0 means the system is keeping up; well above 1.0 across all three windows means it's genuinely, and likely persistently, over capacity — not just a brief spike.

**Q4: Write a one-liner to find the five largest subdirectories under `/var`, to investigate a nearly-full filesystem.**

Solution:
```bash
du -sh /var/*/ 2>/dev/null | sort -rh | head -5
```
Explanation: `du -sh /var/*/` summarizes the total size of each immediate subdirectory under `/var` (the trailing slash restricts the glob to directories). `2>/dev/null` suppresses "permission denied" noise from directories you can't read. `sort -rh` sorts human-readable sizes correctly in reverse (largest first) — plain `sort` would sort `"1.1G"` before `"900M"` alphabetically, which is wrong; `-h` specifically tells `sort` to interpret human-readable suffixes numerically. `head -5` keeps just the top five, the most likely culprits.

---

## Gotcha Questions

**Q1: "I ran `apt upgrade` this morning and it said nothing to upgrade, but a critical CVE patch was announced yesterday for a package I have installed. What went wrong?"**

> Trap: The candidate needs to catch that `apt upgrade` only ever consults the *local* index, and that index is only as fresh as the last `apt update` — if `apt update` hasn't been run since before the patch was published upstream, `upgrade` has no way of knowing the fix exists yet, and correctly (from its own point of view) reports nothing to do. This is exactly the update-before-upgrade distinction the module drills — `upgrade` never fails safe by silently refreshing the index itself first; that's a separate, deliberate step you must always run.

**Q2: "`free -h` shows only 200Mi 'free' out of 8Gi total RAM on a server that's been up for months. Should I be worried?"**

> Trap: A small raw "free" number on a long-uptime server is completely normal and, on its own, tells you nothing bad — Linux deliberately uses idle RAM for disk cache (`buff/cache`) rather than leaving it empty, since cache is instantly reclaimable the moment an application actually needs the memory. The candidate should redirect to the `available` column, which already accounts for that reclaimable cache, and to whether `Swap` usage is meaningfully non-zero and growing — that's the real signal of genuine memory pressure, not the raw "free"/"used" split.

**Q3: "`df -h` shows my root filesystem at 95% full, but I deleted a 10GB log file an hour ago and the number hasn't moved. Did the delete not work?"**

> Trap: The delete very likely did work at the filesystem level — but if some running process still has that file open (a common pattern: a service writing continuously to a log file that then gets deleted out from under it by a cleanup script), the kernel won't actually reclaim the underlying disk blocks until every process holding it open closes it or exits. `du` won't show it (it's not in any directory listing anymore), but `df` correctly reports the space as still allocated, because it genuinely is, from the filesystem's point of view, until that last reference is released. The fix is finding and restarting (or signaling) the process still holding it open — checking `lsof` for `(deleted)` entries is the standard way to find the culprit.

---

## Quick-Fire Rapid Review

- **Q: What does `apt update` actually change on disk?** A: Nothing — it only refreshes the local package index.
- **Q: What command actually installs new package versions?** A: `apt upgrade`.
- **Q: What's the standard safe combo, in order?** A: `sudo apt update && sudo apt upgrade`.
- **Q: Does `apt remove` delete config files?** A: No — `apt purge` does.
- **Q: What command cleans up orphaned dependency packages?** A: `apt autoremove`.
- **Q: What tool does `apt` sit on top of?** A: `dpkg`.
- **Q: How do you find which package owns a file?** A: `dpkg -S <file>`.
- **Q: How do you list every file a package installed?** A: `dpkg -L <package>`.
- **Q: What's a PPA?** A: A third-party, individually-published `apt` repository.
- **Q: What must you check before trusting a load average number?** A: The CPU core count (`nproc`) — divide load average by it.
- **Q: In `free -h`, which column reflects realistic available memory?** A: `available`, not raw `free` or `used`.
- **Q: What's the real memory-pressure red flag in `free -h`?** A: Meaningful, growing `Swap` usage.
- **Q: What command shows disk space per mounted filesystem?** A: `df -h`.
- **Q: What command shows the size of one specific directory?** A: `du -sh <directory>`.
- **Q: Name two deeper sysstat-package monitoring tools.** A: `vmstat` and `iostat`.
- **Q: Why is `curl | sudo bash` risky?** A: It runs unread, remotely-fetched code with root privileges, with no guarantee it hasn't changed or been compromised.
