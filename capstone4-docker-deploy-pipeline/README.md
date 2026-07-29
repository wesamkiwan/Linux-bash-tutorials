# Capstone 4: Docker Deployment Pipeline Script 🔴

**Difficulty:** 🔴 Advanced | **Estimated Time:** 6-8h | **Prerequisites:** Modules 1-19 (especially 14, 16, 17, 19)

## Real-World Scenario

Your team doesn't run Kubernetes — just Docker on a single production host, deployed from CI over SSH. Right now, deploying means `docker stop old && docker run new`, which means a few seconds (sometimes longer, if the new image is slow to start) where the app is completely down. Worse, if the new version has a bug, someone has to notice, SSH in, and manually roll back while customers are affected.

Your manager wants zero-downtime deploys with automatic rollback — the same fundamental idea behind Kubernetes rolling updates or blue/green deployments, but built with nothing but Docker and bash, because that's what you actually have.

This is the capstone that ties the whole course together: Module 19's `exec`/healthcheck/PID-1 knowledge, Module 16's production hardening, Module 14's error handling, and Module 17's timeout discipline all show up here in one real deployment script.

## What You'll Build

A `deploy.sh` that builds a new container image, starts it alongside the still-running old one, waits for the new one to report healthy, cuts traffic over only if it does, and automatically rolls back — cleanly — if it doesn't. Plus a `rollback.sh` for reverting a deploy after the fact.

## Requirements

- [ ] `deploy.sh` builds/pulls a new image tagged with a version identifier (git SHA or timestamp)
- [ ] Starts the new container on a temporary name/port **alongside** the currently-running old container — the old one keeps serving traffic throughout
- [ ] Polls the new container's healthcheck (reusing the wait-loop pattern from Module 19) with a configurable timeout
- [ ] **On success**: stops and removes the old container, renames/repoints the new one to the production name/port
- [ ] **On healthcheck timeout (rollback)**: stops and removes the FAILED new container, leaves the old container completely untouched, logs clearly what happened, and exits non-zero so CI marks the deploy failed
- [ ] Full timestamped logging of every step: build, start, health-check polling, cutover or rollback decision
- [ ] `set -euo pipefail`, a `trap` that cleans up any dangling temporary container if the *script itself* crashes (not just a failed healthcheck — an actual bug or interrupted run), and a `die()` pattern
- [ ] Idempotent: if a leftover container from a previous failed deploy is still around, clean it up before starting a new attempt
- [ ] A `rollback.sh` (or `deploy.sh --rollback`) that can manually revert to the previous known-good image if a problem surfaces *after* cutover
- [ ] Show how this plugs into CI (a GitHub Actions snippet invoking the script over SSH or on a self-hosted runner)

## Starter Guidance

The trickiest part is making the `trap` smart enough to distinguish "something went wrong with my own script logic" (clean up the new container) from "the deploy is proceeding normally and the old container is *supposed* to get removed later" (don't touch anything). Structure it like this:

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_NAME="myapp"
IMAGE_REPO="myregistry/myapp"
VERSION="$(git rev-parse --short HEAD)"
NEW_CONTAINER="${APP_NAME}-new"
OLD_CONTAINER="${APP_NAME}"
HEALTH_TIMEOUT=60
HEALTH_INTERVAL=3
LOG_FILE="/var/log/${APP_NAME}-deploy.log"

DEPLOY_SUCCEEDED=0

log() { : ; }   # TODO
die() { : ; }   # TODO

cleanup() {
    # TODO: runs on trap EXIT. If $DEPLOY_SUCCEEDED is 0, the "-new" container
    # (if it exists) is left over from a failed/interrupted attempt — remove it.
    # NEVER touch $OLD_CONTAINER here — that's the production container's job
    # to remove only in the explicit cutover step below.
    :
}

remove_stale_new_container() {
    # TODO: idempotency — if $NEW_CONTAINER already exists from a prior failed
    # run, stop+remove it before starting a fresh attempt
    :
}

build_image() {
    # TODO: docker build -t "${IMAGE_REPO}:${VERSION}" .
    :
}

start_new_container() {
    # TODO: docker run -d --name "$NEW_CONTAINER" ... "${IMAGE_REPO}:${VERSION}"
    # bind it to a temporary host port, NOT the production port
    :
}

wait_for_healthy() {
    # TODO: poll `docker inspect --format '{{.State.Health.Status}}' $NEW_CONTAINER`
    # (or curl a healthcheck endpoint) every $HEALTH_INTERVAL seconds, up to
    # $HEALTH_TIMEOUT total — return 0 if healthy in time, 1 if it times out
    :
}

cutover() {
    # TODO: stop+remove $OLD_CONTAINER, rename $NEW_CONTAINER to $OLD_CONTAINER's
    # name/port. This is the ONLY place the old container should be removed.
    :
}

rollback_failed_new_container() {
    # TODO: stop+remove $NEW_CONTAINER, log clearly, exit 1. Old container is
    # never touched — it's still running and healthy.
    :
}

main() {
    trap cleanup EXIT
    log "Starting deploy of ${IMAGE_REPO}:${VERSION}"
    remove_stale_new_container
    build_image
    start_new_container

    if wait_for_healthy; then
        cutover
        DEPLOY_SUCCEEDED=1
        log "Deploy succeeded: ${IMAGE_REPO}:${VERSION} is now live"
    else
        rollback_failed_new_container
    fi
}

main "$@"
```

💡 **Hints:**
- Give your container a real `HEALTHCHECK` in its `Dockerfile` (from Module 19) so `docker inspect --format '{{.State.Health.Status}}'` returns `healthy`/`unhealthy`/`starting` — that's a cleaner signal than manually curling an endpoint yourself.
- The `cleanup` trap and `rollback_failed_new_container` overlap in what they do (remove the new container) — that's fine. `rollback_failed_new_container` handles the *expected* failure path with a clear message; `cleanup` is the safety net for *unexpected* crashes (e.g. the script gets killed mid-build).
- Keep track of the previous version's tag somewhere durable (a simple `last-good-version.txt` file) so `rollback.sh` has something to roll back *to*.

## Constraints & Assumptions

- Single Docker host, no orchestrator (Swarm/Kubernetes) — this is deliberately the "you don't have fancy infrastructure yet" scenario
- Assumes the container has a working healthcheck (built in Module 19) — this capstone is about the deployment orchestration, not about building the healthcheck itself
- A reverse proxy (nginx, Traefik) sitting in front and pointing at a stable port is implied but out of scope to configure here — the capstone focuses on the container swap itself

## Stretch Goals

- Keep the last N images locally so rollback doesn't require rebuilding or re-pulling
- Send a deploy success/failure notification via webhook (reuse Capstone 2's alerting pattern)
- Support deploying multiple services from one script (loop over a list of app configs)
- Add a `--dry-run` flag showing what would happen without actually touching containers

## 📋 How to Present This in a Portfolio/Interview

This is the project that signals genuine production/DevOps instinct, not just scripting ability. Zero-downtime deployment and automatic rollback are concepts most junior engineers can *describe* but haven't actually *built* — doing it in raw bash (rather than relying on a platform to do it for you) proves you understand the mechanics underneath the abstraction.

**Suggested portfolio description:**

> "Designed and implemented a zero-downtime Docker deployment pipeline in Bash for a single-host production environment without a container orchestrator. Uses a blue/green-style container swap with healthcheck-gated cutover and automatic rollback on failure, ensuring the previous version keeps serving traffic until the new version is verified healthy. Hardened with trap-based cleanup, idempotent re-run safety, and full deployment audit logging; integrated into CI via a GitHub Actions step."

Be ready to explain, out loud, exactly why the old container isn't removed until the new one proves healthy (the whole point of zero downtime), and how your trap logic tells apart "the deploy failed as expected" from "my script itself crashed" — that distinction is the crux of this entire capstone, and it's exactly what a strong interviewer will ask you to walk through.

---

**Reference solution:** [solution.md](solution.md)
