# 📋 Module 19 Cheat Sheet — Bash + Docker (Entrypoints & CLI Scripting)

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Container** | A lightweight, isolated environment for running a process, packaged with its dependencies |
| **`ENTRYPOINT`** | The command that becomes a container's main process (PID 1) when it starts |
| **PID 1** | The first process in a container's namespace; gets no default signal handling from the kernel unless it registers a handler itself |
| **`exec`** | Bash builtin that replaces the current process with another command — same PID, no child process forked |
| **`SIGTERM`** | The signal `docker stop` sends first, asking a process to shut down gracefully |
| **`SIGKILL`** | The forced-kill signal sent after the grace period if the process hasn't exited; cannot be caught by anything |
| **`HEALTHCHECK`** | Dockerfile instruction defining a command Docker runs periodically to judge container health via its exit code |
| **Wait loop** | A bash loop that polls a dependency until it's ready or a timeout expires, before proceeding |

## `entrypoint.sh` Skeleton (copy-paste template)

```bash
#!/bin/bash
set -euo pipefail

die() {
    echo "ERROR: $1" >&2
    exit 1
}

# 1. Validate required env vars — fail fast
: "${REQUIRED_VAR:?REQUIRED_VAR must be set}"

# 2. Defaults for optional env vars
OPTIONAL_VAR="${OPTIONAL_VAR:-default-value}"

# 3. Wait for dependencies — ALWAYS with a timeout
elapsed=0; TIMEOUT="${WAIT_TIMEOUT:-30}"
until nc -z "${DEP_HOST:?DEP_HOST must be set}" "${DEP_PORT:-5432}" 2>/dev/null; do
    elapsed=$((elapsed + 1))
    [ "$elapsed" -ge "$TIMEOUT" ] && die "timed out waiting for dependency"
    sleep 1
done

# 4. Hand off to the real app — MUST be the last line
exec "$@"
```

## `exec` and Signal Forwarding — Without vs. With

| | Without `exec` (`"$@"`) | With `exec "$@"` |
|---|---|---|
| PID 1 inside container | bash | the application itself |
| `SIGTERM` from `docker stop` goes to | bash (which ignores it — no default handler on PID 1) | the application directly (which can run its own graceful-shutdown code) |
| Result of `docker stop` | full grace period wasted, then forced `SIGKILL` | clean, fast exit — no `SIGKILL` needed |
| App's own `echo $$` shows | a child PID (e.g. `8`) | `1` |
| Diagnose with | `docker exec <container> ps aux` — app not PID 1 | app shown as PID 1 |

✅ **Rule:** `exec "$@"` must be the **last line** of the entrypoint. Nothing meaningful can run after it in that script.

## Env Var Validation Pattern

| Syntax | Behavior | Use for |
|---|---|---|
| `${VAR:?message}` | Exit with `message` if `VAR` is unset **or empty** | Required config (DB URL, secrets, API keys) |
| `${VAR:-default}` | Silently substitute `default` if unset (or empty) | Optional config (ports, log levels, worker counts) |
| `: "${VAR:?msg}"` | Same as above; `:` is a no-op that forces the expansion to evaluate | Standard idiom for validating without printing the value |
| `command -v tool >/dev/null 2>&1 \|\| die "..."` | Checks a required binary exists in the image | Validating the image has what the entrypoint needs |

## Wait-Loop Pattern (always bounded by a timeout)

```bash
elapsed=0
until nc -z "$HOST" "$PORT" 2>/dev/null; do
    elapsed=$((elapsed + 1))
    [ "$elapsed" -ge "${TIMEOUT:-30}" ] && { echo "ERROR: timeout waiting for $HOST:$PORT" >&2; exit 1; }
    sleep 1
done
```

| Check tool | Use for | Notes |
|---|---|---|
| `nc -z HOST PORT` | "Is this TCP port open at all?" | Generic, works for almost any TCP service |
| `curl -fs URL` | "Does this HTTP endpoint respond successfully?" | Best for app-level readiness (`/health`, `/healthz`) |
| `pg_isready -h HOST -p PORT` | "Is Postgres specifically ready for connections?" | More precise than a bare port check for Postgres |
| `wait-for-it.sh host:port -t 30 -- cmd` | Off-the-shelf equivalent of the loop above | [vishnubob/wait-for-it](https://github.com/vishnubob/wait-for-it) |
| `dockerize -wait tcp://host:port -timeout 30s` | Off-the-shelf wait + config templating | Go binary, common in older images |

⚠️ **Never omit the timeout.** An unbounded wait loop hangs the container forever on a broken/mistyped dependency, with no error and no crash — just silence.

## Healthcheck Script Pattern

```bash
#!/bin/bash
set -euo pipefail
curl -fs -o /dev/null --max-time 2 "http://localhost:${PORT:-8080}/health" && exit 0 || exit 1
```

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD /usr/local/bin/healthcheck.sh
```

| Exit code | Docker interprets as |
|---|---|
| `0` | Healthy |
| `1` | Unhealthy |
| (anything else) | Reserved/undefined — always use exactly 0 or 1 |

## Docker CLI + Bash One-Liners

| Task | Command |
|---|---|
| Find a container by name | `docker ps --filter "name=web" --format '{{.ID}}'` |
| Find a container by label | `docker ps --filter "label=app=myapp" --format '{{.ID}}'` |
| Tail logs of a container found by name | `docker logs -f "$(docker ps --filter "name=web" -q)"` |
| Exec a check inside a running container | `docker exec my-container curl -fs localhost:8080/health` |
| Get a container's main process PID (verify `exec` worked) | `docker exec my-container ps aux` — app should be PID 1 |
| Loop: find + exec into every container matching a label | `for c in $(docker ps --filter "label=tier=web" -q); do docker exec "$c" some-check.sh; done` |
| Stop and check exit timing (diagnose missing `exec`) | `time docker stop my-container` — should be near-instant if `exec` is used correctly |

## CI/CD Bash Script Steps — Same Rules Apply

| Principle (from Modules 6/14/15) | Applies in CI as |
|---|---|
| `set -euo pipefail` at the top | Every nontrivial `script:`/`run:` step |
| `die()` / fail-fast on missing config | Missing secret/env var fails the pipeline immediately, not 3 steps later |
| Meaningful, distinct exit codes | CI reads the step's exit code to decide pass/fail — same as any unattended script |
| Unattended execution = no one watching live | A CI runner is exactly like a cron job (Module 15) — silent partial failures ship broken builds |

## 🔁 The New Container Entrypoint Workflow

Do this every time you write an `entrypoint.sh`:

1. **Validate required env vars first** — `${VAR:?message}` or `die()`, before anything else runs.
2. **Apply defaults for optional env vars** — `${VAR:-default}`.
3. **Wait for dependencies with a timeout** — never an unbounded loop.
4. **Log what you're about to do** — one line stating the app is starting, useful for debugging startup issues later.
5. **`exec "$@"` as the very last line** — no exceptions, nothing meaningful after it.
6. **Write a matching `healthcheck.sh`** — cheap, fast, translates a real readiness check into exit 0/1.
7. **Verify signal forwarding** — `docker exec <container> ps aux` (app should be PID 1) and `time docker stop <container>` (should be fast, not the full grace period).
