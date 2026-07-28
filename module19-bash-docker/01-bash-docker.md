# Module 19: Bash + Docker (Entrypoints & CLI Scripting) 🔴

**Difficulty:** 🔴 Advanced
**Estimated Time:** 2 hours
**Prerequisites:** Modules 1-14 (Shell Fundamentals through Error Handling, Traps & Debugging). This module leans heavily on Module 6/14's `set -euo pipefail` and `die()` patterns, and Module 10's signal knowledge (`SIGTERM`, `SIGKILL`, PID basics).

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain why Bash is still everywhere in the container world — entrypoint scripts, init/setup logic, and CI/CD glue code — even though the app itself might be written in something else entirely
- [ ] Write a Docker `entrypoint.sh` that validates required environment variables, does setup work, and hands off cleanly to the real application
- [ ] Explain in depth what "PID 1" means inside a container, why signals like `SIGTERM` behave differently for PID 1, and how `exec "$@"` fixes the problem
- [ ] Use `${VAR:-default}` and explicit validation to build environment-variable configuration patterns that fail fast when something required is missing
- [ ] Write a bash healthcheck script suitable for a Dockerfile's `HEALTHCHECK CMD`, returning correct exit codes
- [ ] Write a "wait for a dependent service" loop with a timeout, and explain why a running container doesn't mean the service inside it is ready
- [ ] Script the Docker CLI from bash (`docker ps`, `docker logs`, `docker exec`) to build small operational tools
- [ ] Explain how the same bash discipline (`set -euo pipefail`, meaningful exit codes) applies directly to CI/CD pipeline scripts, and why a CI runner is just another unattended-execution environment like `cron`

---

## Module Goal

This is the final module of the course, and it's a deliberate bridge: everything you've learned about writing safe, disciplined Bash scripts (Modules 6-14) doesn't stop mattering once your app moves into a container. It matters *more*, because containers strip away a lot of the safety net a full Linux server normally gives you, and the small bash script that starts your container is often the only thing standing between "this deploys cleanly" and "this pages someone at 2 AM."

🎯 **On the job:** Picture two real, common incidents.

**Incident 1 — the deploy that never finishes.** A team containerizes their Node.js API. Every deploy, the orchestrator (Kubernetes, ECS, plain `docker-compose`, doesn't matter which) sends a polite `docker stop`, which is really just a `SIGTERM` sent to the container's main process, followed by a grace period, and then a hard `SIGKILL` if the container hasn't exited by then. Every single deploy, the container ignores the `SIGTERM` completely, sits there for the full grace period (often 10-30 seconds), and gets forcibly killed. In-flight requests get dropped mid-response. Nobody wrote code to *ignore* `SIGTERM` — the bug is entirely in a three-line `entrypoint.sh` that starts the app wrong, as you'll see below.

**Incident 2 — the crash-loop that "just needs a restart."** A team containerizes a web app that depends on a Postgres database, also running in its own container. Every time they redeploy the whole stack, the web app container starts a fraction of a second before the database container has finished initializing, tries to connect, gets a connection refused, and crashes. The orchestrator restarts it. It crashes again — the database is *still* starting up (Postgres itself takes a few seconds to run its own startup routine even after the container process has technically started). Eventually, on the third or fourth restart, the database is finally ready and the app boots fine. Nobody understands why "sometimes it just needs a minute," and everyone has quietly accepted a crash-loop as normal. It isn't. It's a missing wait-loop, also covered below.

Both incidents are entirely bash problems, fixable with small, disciplined scripts — not application code changes. That's exactly what this module builds.

---

## Core Concepts

### 1. Why bash still matters in a container world

A **container** is a lightweight, isolated environment for running a process — it packages an application with everything it needs (libraries, config, dependencies) so it runs the same way everywhere. **Docker** is the most common tool for building and running containers. None of that requires bash — you could containerize a Python app, a Go binary, a Java service, anything.

So why does a course on Bash have a module on Docker? Because almost every real container image still has a thin layer of bash wrapped around the "real" application, doing jobs that are awkward or impossible to do inside the application's own language:

- **Entrypoint scripts** — the very first thing that runs when a container starts, before the actual application. This is almost always bash (or `sh`), because it needs to run before anything else is set up.
- **Setup/init logic** — validating configuration, templating config files from environment variables, waiting for dependencies — done once, at startup, cheaply, in a language with no compile step.
- **CI/CD glue code** — the `script:` steps in a GitHub Actions workflow or GitLab CI pipeline are, under the hood, just shell commands. Building an image, running tests, pushing to a registry, deploying — all commonly bash.

💡 **Analogy:** Think of the application inside a container as an actor about to go on stage. Bash is the stage manager — it's not part of the performance, nobody in the audience is watching it, but it's the one making sure the lights are on, the props are in place, and the actor actually walks on stage at the right moment, instead of the curtain just opening on an empty set. When the stage manager does their job invisibly and correctly, nobody thinks about them at all. When they mess up cue timing, the whole show visibly breaks — and that's exactly what a broken entrypoint script looks like from the outside: a mysteriously flaky, crash-looping, un-stoppable container.

### 2. What an entrypoint script actually is

A Docker **`ENTRYPOINT`** is the command that runs when a container starts — it's *the* main process of the container. When that main process exits, the container exits. In a Dockerfile, you'll commonly see:

```dockerfile
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["node", "server.js"]
```

Here, `ENTRYPOINT` is the script that always runs, and `CMD` supplies the default arguments *to* that script — Docker runs them combined as `entrypoint.sh node server.js`. Inside `entrypoint.sh`, those arguments show up as `"$@"` (Module 8's positional-parameter array), which is the hook this module's central pattern depends on.

A minimal, well-behaved entrypoint script has three jobs, always in this order:

1. **Setup work** — validate required configuration, wait for dependencies, template config files.
2. **Hand off to the real application** — using `exec "$@"` (Concept 3, the heart of this module).
3. Nothing after that — once you `exec`, that script's own execution is over; there's no "step 3" that runs afterward in the same process.

### 3. PID 1, signals, and why `exec "$@"` matters

This is the single most important concept in this module, so we're going to build it up carefully, piece by piece.

**What is PID 1?** Every process on Linux has a **PID** (Process ID) — a number identifying it. **PID 1** is special: it's the very first process the kernel starts, and inside a container, whatever process Docker launches as the `ENTRYPOINT`/`CMD` *becomes* PID 1 of that container's own process namespace. This isn't a Docker-specific quirk — it's how Linux process namespaces work in general (touching on the same process-tree concepts from Module 10).

**Why does PID 1 matter for signals?** A **signal** (Module 10) is a small notification sent to a process — `SIGTERM` politely asks a process to shut down; `SIGKILL` forcibly terminates it with no chance to clean up. `docker stop` works by sending `SIGTERM` to the container's PID 1, waiting a grace period (10 seconds by default), and then sending `SIGKILL` if the process hasn't exited yet.

Here's the trap: **the Linux kernel treats PID 1 differently from every other process.** For an ordinary process, if you don't explicitly write a handler for a signal, the kernel applies a sensible default (for `SIGTERM`, the default is "terminate"). But PID 1 gets *no* default signal handling at all — if PID 1 doesn't explicitly set up a handler for a signal, that signal is simply ignored. This rule exists because PID 1 on a *real* Linux machine is `init`/`systemd`, and you really don't want a stray signal accidentally shutting down the entire machine's init system — so the kernel deliberately requires PID 1 to opt in to handling any signal.

Now connect this to bash: **plain Bash does not register a handler for `SIGTERM` by default.** So if bash itself is PID 1 inside your container — which happens whenever your entrypoint script runs your application as an ordinary foreground command instead of replacing itself — bash simply ignores the `SIGTERM` that `docker stop` sends it. Docker waits out the full grace period, gets no cooperation, and falls back to `SIGKILL`, which cannot be caught or ignored by anything (same rule from Module 14). Your application never even found out it was being shut down — it just got vaporized.

💡 **Analogy — handing off the steering wheel:** Imagine bash as a chauffeur who picks up a passenger (your application) and holds the actual controls the whole trip, just relaying "turn left," "turn right" between a backseat passenger and the car. If someone outside shouts "STOP THE CAR," they're shouting at the chauffeur — but if the chauffeur wasn't listening for that particular instruction, the passenger never hears it at all, no matter how loudly they shout, because the passenger was never actually driving. `exec` is the moment the chauffeur gets out and hands the passenger the wheel directly — now the person actually driving hears "STOP" themselves, immediately, with nothing standing in between.

**What does `exec` actually do?** The **`exec` builtin**, when given a command, **replaces the current process** with that command — same PID, same process, but now running different program code. It does not start a new child process; it *becomes* the new program. So `exec "$@"` at the end of an entrypoint script means: "stop being the bash script now, and become the actual application" — literally the same PID 1 slot, just running the app's binary instead of bash from that point forward. Because it's the *same* PID, and because most real applications (Node, Python, Java, Go binaries, `nginx`, `postgres`, etc.) *do* register their own `SIGTERM` handler for graceful shutdown, the signal now reaches code that actually knows what to do with it.

⚠️ **Without `exec`:** if your entrypoint script instead runs `"$@"` as an ordinary command (no `exec`), bash forks a **child process** to run the application and then just... waits for it, still sitting there as PID 1. `docker stop` sends `SIGTERM` to bash (PID 1), bash ignores it (no handler registered), the application child process never sees anything, and you're back to the full grace-period-then-`SIGKILL` outcome from Incident 1 in the Module Goal.

✅ **With `exec "$@"` as the last line:** bash replaces itself with the application. The application *is* PID 1. `docker stop` sends `SIGTERM` directly to it. If the application has its own graceful-shutdown handler (finishing in-flight requests, closing database connections cleanly), it runs immediately, and the container exits well within the grace period — no forced kill, no dropped requests.

### 4. Environment variable configuration patterns

Containers are configured almost entirely through **environment variables** — small key/value pairs passed in at `docker run` time (`-e KEY=value`) or via a `docker-compose.yml`/orchestrator config, rather than editing files inside the image. This is deliberate: the same image can run in dev, staging, and production just by changing which environment variables are supplied, with no rebuild.

Two patterns matter, and they map directly onto Module 6/14's `set -u` and `die()` patterns:

**Pattern A — fail fast on missing required variables.** If a variable is *required* for the app to function correctly (a database connection string, an API key), the entrypoint should check for it explicitly and refuse to start rather than let the application start up in a broken, half-configured state:

```bash
: "${DATABASE_URL:?ERROR: DATABASE_URL must be set}"
```

**Pattern B — sane defaults for optional variables**, using `${VAR:-default}` (introduced in Module 6, used constantly here):

```bash
PORT="${PORT:-8080}"
LOG_LEVEL="${LOG_LEVEL:-info}"
```

✅ **Best Practice:** Fail fast and loud on missing *required* configuration — at container startup, in the first few lines of the entrypoint, before any setup work happens — rather than letting the application start and fail confusingly three layers deep once it finally tries to use that missing value.

### 5. Healthchecks — is the app actually ready?

A Dockerfile's `HEALTHCHECK` instruction tells Docker to periodically run a command *inside* the running container to check whether the application is actually healthy — not just "the process exists," but "it's genuinely able to do its job" (e.g., a web server that responds to requests, not one that's stuck in an infinite loop or deadlocked while its process technically still runs).

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD /usr/local/bin/healthcheck.sh
```

The healthcheck command's **exit code** is what Docker reads: **`0` means healthy**, **`1` means unhealthy**. A small bash script is a natural fit here — it can run a lightweight check (`curl` a local endpoint, check a PID file, query a socket) and translate the result into the exit code Docker expects.

### 6. The "wait for a dependent service" pattern

A container **starting** and the service **inside** it being **ready** are two completely different moments in time. `docker run postgres` returns almost instantly and the container is "running" — but Postgres itself still needs to run its own internal startup sequence (initializing its data directory, replaying logs, opening its listening socket) before it can actually accept a connection. A container that depends on Postgres and tries to connect immediately at its own startup will very often lose that race.

The standard fix is a **wait loop**: before doing anything that depends on another service, poll it in a loop — using a tool appropriate to what's being checked (`nc` for "is this TCP port open," `curl` for "does this HTTP endpoint respond," `pg_isready` for "is Postgres specifically ready for connections") — with a short sleep between attempts, **and always a timeout**, so a genuinely broken dependency fails loudly instead of hanging forever.

💡 **Tip:** You don't have to hand-write this every time. Well-known existing tools do exactly this job: **[`wait-for-it.sh`](https://github.com/vishnubob/wait-for-it)** is a single, widely-used bash script that waits for a TCP host:port to become available with a configurable timeout; **`dockerize`** is a small Go binary that does similar waiting plus config templating. Either is a perfectly good production choice — the point of learning to write the loop yourself is understanding exactly what it's doing under the hood, so you can debug it (or a custom variant) when the generic tool doesn't fit your exact check.

### 7. Docker CLI scripting from bash

The `docker` CLI is just another command whose output you can parse and script around, exactly like any other command from Modules 2-9. Three subcommands come up constantly in operational scripts:

- **`docker ps`** — list running containers; `--filter` narrows results (e.g., `--filter "name=web"`), and `--format` controls output shape for easy parsing.
- **`docker logs`** — print (or `-f` to follow/stream) a container's stdout/stderr — the same output you'd see if you'd run it in the foreground.
- **`docker exec`** — run a command *inside* an already-running container's namespace, useful for ad-hoc checks (`docker exec my-container ps aux`) without needing a full shell session.

Combining these with familiar tools (`grep`, `awk` from Module 9; loops from Module 7) lets you build small, genuinely useful operational scripts — "find the container running my app by name and tail its logs" is a five-line script, not a special Docker-only skill.

### 8. CI/CD: bash as the glue, one more unattended environment

**CI/CD** (Continuous Integration / Continuous Deployment) systems — GitHub Actions, GitLab CI, and similar — run your code automatically on events like a `git push`. Under the hood, the `script:`/`run:` steps in their YAML configuration are, in the vast majority of real pipelines, just shell commands executed on a runner machine.

🎯 **On the job:** Tie this back to Module 15's automation content — a **CI runner is just another kind of unattended-execution environment, exactly like a cron job.** Nobody is watching a CI pipeline run interactively; if a step fails silently and the pipeline still reports green, a broken build gets shipped with nobody the wiser, for exactly the same reason a broken cron job goes unnoticed until someone asks "wait, why didn't the report generate last night?" All the same discipline applies directly, with no modification: `set -euo pipefail` at the top of any nontrivial script step, meaningful and distinct exit codes, and `die()`-style fail-fast checks for missing configuration (a missing `DOCKER_REGISTRY_TOKEN` secret should fail the pipeline immediately and legibly, not three steps later with a confusing authentication error).

---

## Detailed Explanations

### The PID 1 signal problem, before and after, concretely

Here is the *broken* pattern — deceptively simple-looking, and extremely common in entrypoint scripts written by people who haven't hit this problem yet:

```bash
#!/bin/bash
# entrypoint.sh — BROKEN: no exec
echo "Starting application..."
"$@"
echo "Application exited"
```

Walk through what actually happens when this runs as a container's `ENTRYPOINT`:

1. Docker starts this script. Bash becomes PID 1 inside the container.
2. `"$@"` (say, `node server.js`) runs as an ordinary foreground command — bash **forks a child process** for it and waits.
3. Bash, still PID 1, has registered no handler for `SIGTERM` (it never does, by default).
4. `docker stop` sends `SIGTERM` to PID 1 — which is bash, not `node`.
5. Bash ignores it (default PID-1 behavior per Concept 3). `node` never receives anything at all; it's still happily running, unaware anyone asked it to stop.
6. Docker waits out the full grace period (10 seconds by default), sees the container hasn't exited, and sends `SIGKILL` — to PID 1 (bash), which the kernel then also propagates to the process tree as the container is torn down, but with **no warning and no chance for `node` to finish in-flight work or close connections cleanly.**
7. `"Application exited"` never prints — the container was killed out from under the script, not exited normally.

Now the *fixed* version:

```bash
#!/bin/bash
# entrypoint.sh — FIXED: exec replaces bash with the app
echo "Starting application..."
exec "$@"
```

1. Docker starts this script. Bash is briefly PID 1.
2. `echo` runs and prints normally — ordinary bash behavior up to this point.
3. `exec "$@"` runs. Bash **replaces itself** with `node server.js` — no fork, no child process, the **same PID** now runs `node`'s code instead of bash's.
4. `node` is now PID 1. Node.js (like most real application runtimes) *does* register a handler for `SIGTERM` by default behavior in most frameworks, or the application code explicitly does.
5. `docker stop` sends `SIGTERM` to PID 1 — which is now genuinely `node`.
6. `node`'s handler runs: finish in-flight HTTP responses, close the database pool, log "shutting down," then exit cleanly — typically in well under a second, nowhere near the 10-second grace period.
7. Docker sees the container exit on its own and never needs to escalate to `SIGKILL` at all.

⚠️ **Warning:** `exec "$@"` only forwards signals correctly if `"$@"` is genuinely the last thing the script does, and nothing meaningful is expected to run afterward in that same script — because after `exec` succeeds, there *is* no "afterward" in that process anymore. Anything you need to do (logging, cleanup) has to happen **before** the `exec` line, not after it.

---

## Practical Examples

### Example 1 — Broken vs. fixed entrypoint, signal handling side by side

**Broken** (`entrypoint-broken.sh`):

```bash
#!/bin/bash
set -euo pipefail
echo "[entrypoint] setup complete, launching app (NO exec — broken)"
"$@"
```

**Fixed** (`entrypoint-fixed.sh`):

```bash
#!/bin/bash
set -euo pipefail
echo "[entrypoint] setup complete, launching app"
exec "$@"
```

Realistic behavior, testing with a tiny script standing in for "the app" that traps `SIGTERM`:

```bash
# app.sh — simulates a well-behaved application
#!/bin/bash
trap 'echo "[app] got SIGTERM, shutting down gracefully"; exit 0' TERM
echo "[app] running, PID $$"
sleep 3600
```

Running the broken version and sending it `SIGTERM` after a moment:
```
[entrypoint] setup complete, launching app (NO exec — broken)
[app] running, PID 8         <- note: PID 8, NOT PID 1 — it's a child of bash
(SIGTERM sent to PID 1 — bash — is silently ignored)
(10 seconds pass...)
(container force-killed by SIGKILL — "[app] got SIGTERM..." never prints)
```

Running the fixed version and sending it `SIGTERM`:
```
[entrypoint] setup complete, launching app
[app] running, PID 1         <- same PID exec replaced bash with
[app] got SIGTERM, shutting down gracefully
(container exits cleanly within a second — no SIGKILL needed)
```

Line-by-line:
- In the broken version, `app.sh`'s own `echo "PID $$"` prints something like `8`, not `1` — direct, observable proof that bash (PID 1) forked a child instead of handing over its own PID.
- In the fixed version, that same `echo` prints `1` — proof `exec` genuinely replaced the process rather than spawning a new one.
- 🎯 **On the job:** This PID number is the single fastest way to diagnose this exact class of bug on a real system — `docker exec <container> ps aux` and checking whether your application shows up as PID 1 tells you immediately whether your entrypoint is using `exec` correctly, with zero guessing.

### Example 2 — Environment variable validation with defaults

```bash
#!/bin/bash
# entrypoint.sh
set -euo pipefail

die() {
    echo "ERROR: $1" >&2
    exit 1
}

# Required — fail fast if missing (Module 6/14's set -u + die() pattern, applied to container config)
: "${DATABASE_URL:?DATABASE_URL must be set — e.g. postgres://user:pass@db:5432/appdb}"
: "${API_KEY:?API_KEY must be set}"

# Optional — sane defaults using ${VAR:-default}
PORT="${PORT:-8080}"
LOG_LEVEL="${LOG_LEVEL:-info}"
WORKERS="${WORKERS:-4}"

echo "[entrypoint] config: PORT=$PORT LOG_LEVEL=$LOG_LEVEL WORKERS=$WORKERS"
echo "[entrypoint] validated required environment variables"

exec "$@"
```

Realistic output when `DATABASE_URL` was never set:
```
/usr/local/bin/entrypoint.sh: line 9: DATABASE_URL: DATABASE_URL must be set — e.g. postgres://user:pass@db:5432/appdb
```

Realistic output when everything is set correctly:
```
[entrypoint] config: PORT=8080 LOG_LEVEL=info WORKERS=4
[entrypoint] validated required environment variables
```

Line-by-line:
- `: "${DATABASE_URL:?message}"` — the `:` is a no-op builtin that does nothing with its argument except force it to be evaluated; `${VAR:?message}` (Module 6/14's parameter-expansion family) exits the shell immediately with `message` printed to stderr if `VAR` is unset *or empty*. This is stricter than plain `set -u`, which only catches *unset*, not empty-but-blank values — an important distinction for container configs where an env var can easily be declared but left blank in a compose file.
- `${PORT:-8080}` supplies a default *without* erroring — the right choice for anything the application can reasonably run without, unlike the two required values above it.
- The validation and logging happen entirely **before** `exec "$@"` — exactly the ordering Concept 2 and the Detailed Explanations section require: all setup work first, `exec` last, nothing after it.

💡 **Tip:** `${VAR:?message}` and `${VAR:-default}` look almost identical but do opposite jobs — `:?` fails loudly on missing/empty, `:-` silently fills in a fallback. Mixing them up (using `:-` for something genuinely required) is how a missing database URL turns into a confusing empty-string connection error three layers deeper in the application instead of a clear message at startup.

### Example 3 — A healthcheck script

```bash
#!/bin/bash
# healthcheck.sh
set -euo pipefail

PORT="${PORT:-8080}"

# curl -f: fail (exit non-zero) on HTTP error codes, not just connection failures
# -s: silent (no progress bar), -o /dev/null: discard the response body, we only need the status
if curl -fs -o /dev/null --max-time 2 "http://localhost:${PORT}/health"; then
    exit 0   # healthy
else
    exit 1   # unhealthy
fi
```

Dockerfile wiring:
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD /usr/local/bin/healthcheck.sh
```

Realistic behavior, checking with `docker ps`:
```
CONTAINER ID   IMAGE     STATUS
a1b2c3d4e5f6   my-app    Up 2 minutes (healthy)
```

...or, if the app's `/health` endpoint stops responding:
```
CONTAINER ID   IMAGE     STATUS
a1b2c3d4e5f6   my-app    Up 5 minutes (unhealthy)
```

Line-by-line:
- `curl -fs -o /dev/null --max-time 2` — `-f` makes `curl` itself return a non-zero exit code on a 4xx/5xx HTTP response (without `-f`, `curl` would exit 0 even for a 500 error page, since it successfully "curled" *something*); `--max-time 2` caps how long any single check can take, so a hung application doesn't also hang the healthcheck itself indefinitely.
- The `if`/`else` directly maps `curl`'s success/failure into the exact `exit 0`/`exit 1` convention Docker's `HEALTHCHECK` expects (Concept 5) — this script's *only* job is that translation.
- `--interval=30s --timeout=3s --retries=3` in the Dockerfile means Docker calls this script every 30 seconds, gives it up to 3 seconds to finish, and only marks the container `unhealthy` after 3 consecutive failures — avoiding a single network blip from flapping the status.

⚠️ **Warning:** A healthcheck that's too expensive (querying a full database, running a heavy computation) run every 30 seconds adds real, continuous load — keep the check itself cheap and fast; check "is the app responsive," not "run a full integration test."

### Example 4 — Wait-for-Postgres loop with a timeout

```bash
#!/bin/bash
# wait-for-postgres.sh
set -euo pipefail

DB_HOST="${DB_HOST:?DB_HOST must be set}"
DB_PORT="${DB_PORT:-5432}"
TIMEOUT="${WAIT_TIMEOUT:-30}"

echo "[wait-for-postgres] waiting for ${DB_HOST}:${DB_PORT} (timeout ${TIMEOUT}s)..."

elapsed=0
until nc -z "$DB_HOST" "$DB_PORT" 2>/dev/null; do
    elapsed=$((elapsed + 1))
    if [ "$elapsed" -ge "$TIMEOUT" ]; then
        echo "ERROR: timed out after ${TIMEOUT}s waiting for ${DB_HOST}:${DB_PORT}" >&2
        exit 1
    fi
    sleep 1
done

echo "[wait-for-postgres] ${DB_HOST}:${DB_PORT} is accepting connections after ${elapsed}s"
exec "$@"
```

Realistic output — database becomes ready partway through:
```
[wait-for-postgres] waiting for db:5432 (timeout 30s)...
[wait-for-postgres] db:5432 is accepting connections after 4s
[entrypoint] config: PORT=8080 LOG_LEVEL=info WORKERS=4
```

Realistic output — database genuinely never comes up (e.g. wrong hostname):
```
[wait-for-postgres] waiting for db-typo:5432 (timeout 30s)...
ERROR: timed out after 30s waiting for db-typo:5432
```

Line-by-line:
- `nc -z "$DB_HOST" "$DB_PORT"` — `nc` (netcat) with `-z` performs a zero-I/O connection scan: it just checks whether the TCP port is open and accepting connections, without sending or expecting any actual data. This confirms "something is listening," which for Postgres specifically is a reasonable proxy for "ready" (a more precise check would use `pg_isready`, Postgres's own purpose-built readiness tool, when it's available in the image).
- The `until` loop (Module 7's control flow) keeps retrying once per second until either the check succeeds or `elapsed` reaches `TIMEOUT`.
- **The timeout check is not optional** — without it, a genuinely-broken or mistyped hostname makes this loop retry forever, hanging the container startup indefinitely with no error, no crash, just silence (see Common Pitfalls below).
- `exec "$@"` at the end means this script itself is designed to be chained *in front of* the real entrypoint/app — a common composition pattern: `wait-for-postgres.sh entrypoint.sh node server.js`, where each script does its one job and `exec`s into the next, so the final process is still genuinely PID 1 by the time the real app runs.

---

## Common Pitfalls & Best Practices

- **Forgetting `exec` before the final command.** This is the single most common and most damaging entrypoint bug — covered in full in the Detailed Explanations section. If your container ignores `docker stop` and always waits out the full grace period before being killed, this is almost always the cause. Check with `docker exec <container> ps aux` — if your app isn't PID 1, `exec` is missing somewhere in the chain.
- **A wait loop with no timeout.** `until nc -z "$HOST" "$PORT"; do sleep 1; done` with no exit condition looks harmless in a demo, but in production, a mistyped hostname, a firewall rule, or a genuinely dead dependency turns it into an infinite hang — the container never starts, never fails, and never logs anything useful; it just sits there, and whoever's debugging it has no idea whether it's "still starting" or "broken forever." Always cap the loop with a timeout and a clear, specific error message on expiry.
- **Healthchecks that are too slow or too expensive.** A `HEALTHCHECK` that runs a full database query or a multi-second computation, every 30 seconds, for the container's entire lifetime, adds continuous unnecessary load and risks the healthcheck itself timing out under normal load spikes, falsely marking a healthy container as unhealthy. Keep the check cheap: a lightweight `/health` endpoint, a port check, a PID-file check.
- **Hardcoding hostnames instead of using environment variables.** `curl http://prod-db-01.internal:5432` baked directly into a script breaks the instant that image is reused in staging, in a different region, or behind a different orchestrator's networking. Docker (and Docker Compose, Kubernetes) provide service discovery via DNS names passed as environment variables (`DB_HOST=db` inside a Compose network) specifically so the same image works unmodified across environments — always read the hostname from an env var with a sensible default, never hardcode it.
- **Doing setup work *after* `exec`.** Impossible by construction — once `exec` succeeds, the bash script's own code stops existing in that process. Any logging, cleanup, or additional setup needs to happen strictly before the `exec` line.
- **Treating a `docker ps` "running" status as "ready."** As Concept 6 covers, a running container and a ready service are different moments. Scripts that assume the former implies the latter are exactly how Incident 2 from the Module Goal happens.

✅ **Best Practice — the entrypoint mental model:** An entrypoint script's whole job is to get out of the way as fast as possible while still doing its setup correctly. Validate, wait, template config — then `exec`. The less time bash spends being PID 1, the less opportunity there is for a signal to arrive at the wrong moment and get silently swallowed.

---

## Hands-on Exercise

**Task:** You're containerizing a hypothetical web app, `webapp`, that:
- Requires `DATABASE_URL` and `SESSION_SECRET` environment variables (fail fast if missing).
- Has an optional `PORT` (default `3000`) and `LOG_LEVEL` (default `info`).
- Depends on a Postgres database reachable at `DB_HOST`/`DB_PORT` (default port `5432`), which may not be ready the instant the container starts.
- Exposes a `/healthz` endpoint once it's actually ready to serve traffic.

Write:
1. `entrypoint.sh` — validates required env vars, waits for the database with a timeout, then `exec`s the app.
2. `healthcheck.sh` — checks `/healthz` and returns the correct exit code for Docker's `HEALTHCHECK`.

Try writing both yourself before reading the solution.

### Solution

**`entrypoint.sh`:**

```bash
#!/bin/bash
set -euo pipefail

die() {
    echo "ERROR: $1" >&2
    exit 1
}

# --- 1. Validate required configuration, fail fast ---
: "${DATABASE_URL:?DATABASE_URL must be set}"
: "${SESSION_SECRET:?SESSION_SECRET must be set}"

# --- Sane defaults for optional configuration ---
PORT="${PORT:-3000}"
LOG_LEVEL="${LOG_LEVEL:-info}"
DB_HOST="${DB_HOST:?DB_HOST must be set}"
DB_PORT="${DB_PORT:-5432}"
WAIT_TIMEOUT="${WAIT_TIMEOUT:-30}"

echo "[entrypoint] PORT=$PORT LOG_LEVEL=$LOG_LEVEL DB_HOST=$DB_HOST DB_PORT=$DB_PORT"

# --- 2. Wait for the database dependency, with a timeout ---
echo "[entrypoint] waiting for database at ${DB_HOST}:${DB_PORT} (timeout ${WAIT_TIMEOUT}s)..."
elapsed=0
until nc -z "$DB_HOST" "$DB_PORT" 2>/dev/null; do
    elapsed=$((elapsed + 1))
    if [ "$elapsed" -ge "$WAIT_TIMEOUT" ]; then
        die "database at ${DB_HOST}:${DB_PORT} not reachable after ${WAIT_TIMEOUT}s"
    fi
    sleep 1
done
echo "[entrypoint] database is reachable after ${elapsed}s"

# --- 3. Hand off to the real application — same PID, signals forward correctly ---
echo "[entrypoint] starting application"
exec "$@"
```

**`healthcheck.sh`:**

```bash
#!/bin/bash
set -euo pipefail

PORT="${PORT:-3000}"

if curl -fs -o /dev/null --max-time 2 "http://localhost:${PORT}/healthz"; then
    exit 0
else
    exit 1
fi
```

**Dockerfile wiring:**

```dockerfile
COPY entrypoint.sh healthcheck.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/entrypoint.sh /usr/local/bin/healthcheck.sh

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["node", "server.js"]

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD /usr/local/bin/healthcheck.sh
```

Explanation of the design: every required variable is checked with `:?` **before** any other work happens, so a misconfigured deployment fails in under a second with a clear message rather than limping partway through startup. The wait loop is bounded by `WAIT_TIMEOUT`, so a genuinely broken database dependency produces a clear `die()` message instead of hanging the container forever — directly avoiding Incident 2 from the Module Goal. `exec "$@"` is the very last line, guaranteeing the Node process becomes PID 1 and receives `SIGTERM` directly from `docker stop` — directly avoiding Incident 1. The healthcheck script is deliberately tiny and fast, translating a single `curl` call into the exit code Docker expects, and reads `PORT` from the same environment variable the entrypoint uses, rather than a hardcoded value.

✅ Exercise complete — you've written a production-shaped entrypoint that fails fast on bad config, survives a slow-starting dependency, and shuts down gracefully under `docker stop`, plus a matching healthcheck.

---

## ✅ Module Completion Checklist

- [ ] I can explain why bash still matters in a container world — entrypoints, init logic, CI/CD glue
- [ ] I can write a Docker `entrypoint.sh` that validates env vars, does setup, and hands off to the app
- [ ] I can explain what PID 1 is, why signals behave differently for it, and how `exec "$@"` fixes the SIGTERM-forwarding problem
- [ ] I can use `${VAR:-default}` for optional config and `${VAR:?message}`/`die()` for required config
- [ ] I can write a bash healthcheck script that returns 0 for healthy and 1 for unhealthy
- [ ] I can write a wait-for-dependency loop with a mandatory timeout, and explain why "container running" != "service ready"
- [ ] I can script `docker ps`/`docker logs`/`docker exec` from bash for operational tasks
- [ ] I can explain why the same `set -euo pipefail`/exit-code discipline applies to CI/CD pipeline scripts

## Next Step

You've completed all 19 modules! Continue to the [Capstone Projects](../capstone1-backup-toolkit/) to build portfolio-worthy proof of your skills.
