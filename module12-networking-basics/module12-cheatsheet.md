# 📋 Module 12 Cheat Sheet — Networking Basics

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Port** | A numbered "door" (0-65535) into a machine, identifying which service traffic is meant for |
| **HTTP method** | The verb of a web request — GET (retrieve), POST (create/submit), PUT (replace), DELETE (remove) |
| **Header** | Metadata attached to an HTTP request/response, separate from the body (e.g. `Content-Type`) |
| **SSH** | Secure Shell — an encrypted protocol for logging into a remote machine's command line |
| **Private key** | Your secret half of a key pair; never leaves your machine, never shared |
| **Public key** | The shareable half of a key pair; safe to install on any server you want access to |
| **`known_hosts`** | Local file recording the host key fingerprint of every server you've connected to and trusted |
| **ICMP** | The lightweight protocol `ping` uses to test basic reachability |
| **DNS** | Domain Name System — translates hostnames (example.com) into IP addresses |

## `curl` Flags Reference

| Flag | Purpose |
|---|---|
| *(none)* | Sends a GET request, prints response body to terminal |
| `-X <METHOD>` | Set the HTTP method (`-X POST`, `-X DELETE`, etc.) |
| `-d '<data>'` / `--data` | Attach a request body; implies POST if `-X` not given |
| `-H "Name: value"` | Add a custom header; repeatable |
| `-o <file>` | Save response to a named file |
| `-O` | Save response using the URL's own filename |
| `-L` | Follow redirects automatically |
| `-I` | Send a HEAD request — headers only, no body |
| `-v` | Verbose mode — show the full request/response conversation (`>` sent, `<` received) |
| `-s` | Silent mode — suppress the progress meter (useful in scripts) |
| `-i` | Include response headers along with the body |

**Common recipe:**
```bash
curl -X POST https://api.example.com/v1/things \
  -H "Content-Type: application/json" \
  -d '{"key":"value"}'
```

## `wget` Quick Reference

| Command | Effect |
|---|---|
| `wget <url>` | Download, saved as the URL's own filename |
| `wget -O <name> <url>` | Download, saved under a chosen filename |
| `wget -c <url>` | Resume/continue a partially-downloaded file |
| `wget -r <url>` | Recursive download (whole directory tree) |

**`curl` vs. `wget` — when to use which:**

| Situation | Use |
|---|---|
| Testing/calling a JSON API | `curl` |
| Custom HTTP method, headers, or request body | `curl` |
| Downloading one large file, possibly resumable | `wget` |
| Recursively grabbing a directory of files | `wget` |
| Piping response into another command/script | `curl` |

## SSH / SCP / Rsync Command Reference

| Command | Effect |
|---|---|
| `ssh user@host` | Connect to a remote host as `user` |
| `ssh -p 2222 user@host` | Connect on a non-default port |
| `ssh host-alias` | Connect using a `~/.ssh/config` `Host` alias |
| `ssh-keygen -t ed25519 -C "comment"` | Generate a new key pair (modern default type) |
| `ssh-keygen -f ~/.ssh/name -t ed25519` | Generate a key pair with a custom filename |
| `ssh-copy-id user@host` | Install your public key on a remote server's `authorized_keys` |
| `ssh-keygen -R hostname` | Remove a specific stale entry from `known_hosts` |
| `scp file user@host:/path/` | Copy a local file to a remote server |
| `scp user@host:/path/file .` | Copy a remote file down to the current directory |
| `scp -r dir/ user@host:/path/` | Copy a directory recursively |
| `rsync -avz src/ user@host:/path/` | Sync a directory: archive mode, verbose, compressed |
| `rsync -avz --delete src/ user@host:/path/` | Same, also removing files at destination not present in source |
| `rsync -avz -e "ssh -p 2222" src/ host:/path/` | Rsync over SSH on a non-default port |

## `~/.ssh/config` Example Block

```
Host prod-web
    HostName 203.0.113.10
    User weki
    Port 22
    IdentityFile ~/.ssh/id_ed25519_prod

Host staging
    HostName 203.0.113.20
    User deploy
    IdentityFile ~/.ssh/id_ed25519_staging
```

Then simply: `ssh prod-web`, `scp file.txt prod-web:/tmp/`, `rsync -avz build/ staging:/var/www/`

## `ss` Flags Reference

| Flag | Meaning |
|---|---|
| `-t` | Show TCP sockets |
| `-u` | Show UDP sockets |
| `-l` | Show only listening sockets |
| `-p` | Show owning process name/PID (may need `sudo`) |
| `-n` | Show numeric ports/addresses, skip name resolution |
| `-a` | Show all sockets (listening + established) |

**Most-used combo:** `ss -tulpn` — every listening TCP/UDP socket, with process info, numeric.

`netstat` (legacy equivalent, install via `sudo apt install net-tools` if missing): `netstat -tulpn`

## Common Ports Reference

| Port | Service |
|---|---|
| 22 | SSH |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL / MariaDB |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 8080 | HTTP alternate (common for dev servers, proxies) |
| 27017 | MongoDB |

## Reachability & DNS

| Command | Purpose |
|---|---|
| `ping -c 4 host` | Send 4 ICMP echo requests, then stop |
| `dig example.com` | Detailed DNS lookup |
| `nslookup example.com` | Simple, older DNS lookup |
| `host example.com` | Terse, one-line DNS lookup |
| `/etc/hosts` | Local file mapping hostnames to IPs, checked before DNS |

## 🔁 The New Server SSH Setup Workflow

Do this every time you get access to a brand-new server:

1. **First connection** — `ssh user@new-host`. Verify the host key fingerprint shown looks legitimate (compare against one provided out-of-band if possible), then accept it — this is saved to `~/.ssh/known_hosts`.
2. **Generate a key pair** if you don't already have one for this purpose — `ssh-keygen -t ed25519 -C "purpose"`.
3. **Install your public key** — `ssh-copy-id user@new-host`.
4. **Add a `Host` alias** in `~/.ssh/config` so you never retype connection details again.
5. **Confirm passwordless login** — `ssh alias-name` should drop you straight in, no password prompt.
6. **(Server-side, once confirmed working)** disable password authentication in `/etc/ssh/sshd_config` so keys are the only way in.

## 🔁 The "Can't Connect" Troubleshooting Workflow

Do this every time something "can't reach" something else:

1. **Is the host reachable at all?** `ping -c 4 <host>` — no reply / 100% packet loss means a network-level problem before anything else matters.
2. **Does DNS resolve correctly?** `dig <hostname>` — confirm it resolves to the IP you expect.
3. **Is the right port even open on the target?** `ss -tulpn` on the target machine (if you have access) — confirm the service is `LISTEN`ing where you expect, and not bound only to `127.0.0.1` if remote access is needed.
4. **Does the application itself respond correctly?** `curl -v <url>` — read the full request/response conversation to see exactly where it breaks.
