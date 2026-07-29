# Linux Bash — Zero to Hero 🐧

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Modules](https://img.shields.io/badge/modules-19-blue.svg)](PROGRESS.md)
[![Capstone Projects](https://img.shields.io/badge/capstones-4-orange.svg)](#-capstone-projects)
[![Est. Time](https://img.shields.io/badge/est.%20time-~45h-informational.svg)](#-module-roadmap)
[![Level](https://img.shields.io/badge/level-beginner%20to%20advanced-brightgreen.svg)](#-prerequisites)
[![Shell](https://img.shields.io/badge/shell-bash%205.x-4EAA25.svg?logo=gnubash&logoColor=white)](https://www.gnu.org/software/bash/)
[![Platform](https://img.shields.io/badge/platform-Ubuntu%20%2F%20Debian-E95420.svg?logo=ubuntu&logoColor=white)](#%EF%B8%8F-environment-setup)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/wesamkiwan/Linux-bash-tutorials/pulls)
[![AI-Assisted](https://img.shields.io/badge/AI--assisted-Claude%20Code-D97757.svg?logo=anthropic&logoColor=white)](https://claude.com/claude-code)

A complete, production-ready learning path for Linux command-line and Bash scripting — from your very first `ls` to writing hardened, production-grade automation scripts. Examples use **Ubuntu/Debian** (`apt`), with notes where other distros differ.

## 🎯 What you'll be able to do when you finish

- Navigate, inspect, and manipulate any Linux filesystem confidently from the terminal
- Manage users, groups, and permissions the way a sysadmin does
- Read and write Bash scripts with variables, control flow, functions, and arrays
- Process and transform text at scale with `grep`, `sed`, and `awk`
- Manage processes, jobs, and system resources
- Automate recurring work with `cron` and `systemd` timers
- Write scripts that fail safely, log properly, and are safe to run in production
- Profile script performance and audit a system for security issues
- Write Bash-based Docker entrypoint and CI/CD deployment scripts
- Walk into a DevOps/SRE/sysadmin interview and answer both conceptual and hands-on questions

## ✅ Prerequisites

None. This course assumes **zero prior Linux/command-line experience**. If you've never opened a terminal, start at Module 1.

If you already know some basics, use the difficulty tags below to jump ahead — 🟢 Beginner, 🟡 Intermediate, 🔴 Advanced.

## 🖥️ Environment Setup

You need a real Linux shell to practice in. Pick one:

| Option | Best for | Setup |
|---|---|---|
| **WSL2 (Windows)** | Windows users (recommended if you're on Windows) | `wsl --install -d Ubuntu` in an elevated PowerShell, then restart |
| **Native Ubuntu/Debian VM** | Full isolation, closest to a real server | VirtualBox/VMware + [Ubuntu Desktop ISO](https://ubuntu.com/download/desktop) |
| **Cloud VM (free tier)** | Practicing on a "real" remote server + SSH | Oracle Cloud Free Tier / AWS Free Tier — spin up an Ubuntu instance |
| **Docker container** | Quick, disposable practice sandbox | `docker run -it --rm ubuntu:24.04 bash` |

Once you have a shell open, confirm you're ready:

```bash
whoami          # should print your username
bash --version  # should print GNU bash, version 5.x or later
```

💡 **Tip:** Keep a terminal open in a second window/pane while reading each module — type every example yourself. Reading without typing is the #1 reason people stall out on this path.

## 🗺️ Module Roadmap

| # | Module | Difficulty | Est. Time |
|---|--------|-----------|-----------|
| 1 | [Linux & Shell Fundamentals](module01-shell-fundamentals/) | 🟢 | 2h |
| 2 | [Filesystem Navigation & File Operations](module02-filesystem-navigation/) | 🟢 | 2.5h |
| 3 | [Viewing & Finding Files](module03-viewing-finding-files/) | 🟢 | 2h |
| 4 | [Permissions, Users & Ownership](module04-permissions-users/) | 🟡 | 2h |
| 5 | [I/O Redirection, Pipes & Filters](module05-io-redirection-pipes/) | 🟡 | 2h |
| 6 | [Bash Scripting Fundamentals](module06-scripting-fundamentals/) | 🟡 | 3h |
| 7 | [Control Flow](module07-control-flow/) | 🟡 | 2.5h |
| 8 | [Functions, Arrays & String Manipulation](module08-functions-arrays-strings/) | 🟡 | 2.5h |
| 9 | [Text Processing Power Tools (sed/awk/regex)](module09-sed-awk-regex/) | 🔴 | 3.5h |
| 10 | [Process Management & Job Control](module10-process-management/) | 🟡 | 2h |
| 11 | [Package Management & System Monitoring](module11-package-mgmt-monitoring/) | 🟢/🟡 | 2h |
| 12 | [Networking Basics](module12-networking-basics/) | 🟡 | 2h |
| 13 | [Terminal Productivity (tmux/dotfiles)](module13-terminal-productivity/) | 🟡 | 2h |
| 14 | [Error Handling, Traps & Debugging](module14-error-handling-debugging/) | 🔴 | 3h |
| 15 | [Automation & Scheduling (cron/systemd)](module15-automation-scheduling/) | 🔴 | 2h |
| 16 | [Production Scripting & Security Hardening](module16-production-security-hardening/) | 🔴 | 2.5h |
| 17 | [Performance Tuning & Profiling](module17-performance-tuning/) | 🔴 | 2h |
| 18 | [Security Auditing Scripts](module18-security-auditing/) | 🔴 | 2.5h |
| 19 | [Bash + Docker](module19-bash-docker/) | 🔴 | 2h |

**Total estimated time:** ~45 hours

## 🏆 Capstone Projects

- [Capstone 1: Automated Backup & Log-Rotation Toolkit](capstone1-backup-toolkit/) 🟡
- [Capstone 2: Server Monitoring & Alerting System](capstone2-monitoring-alerting/) 🔴
- [Capstone 3: Security Audit Automation Script](capstone3-security-audit-automation/) 🔴
- [Capstone 4: Docker Deployment Pipeline Script](capstone4-docker-deploy-pipeline/) 🔴

## 📚 Master References

- [master-cheatsheet.md](master-cheatsheet.md) — every command, pattern, and workflow in one scannable file
- [master-interview-prep.md](master-interview-prep.md) — every interview question across the whole course
- [master-references.md](master-references.md) — every curated learning resource across the whole course

## 📊 Track Your Progress

Open **[PROGRESS.md](PROGRESS.md)** and keep it updated as you go. It has a checklist for every module, a "you are here" pointer, and an overall completion percentage.

## 🧭 How to Use This Course

**The Daily Study Workflow — do this every session:**
1. Open `PROGRESS.md`, find your "you are here" pointer
2. Read the module's learning file(s) top to bottom, typing every example
3. Do the hands-on exercise yourself before peeking at the solution
4. Skim the module's cheat sheet — this is what you'll actually use on the job
5. Read the module's interview prep, out loud if possible
6. Tick the module's boxes in `PROGRESS.md`, move to the next module

Good luck — see you in [Module 1](module01-shell-fundamentals/).
