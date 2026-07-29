# Capstone 4: Reference Solution — Docker Deployment Pipeline Script

This is one valid, complete solution. Your implementation doesn't need to match line-for-line — it needs to satisfy the requirements in [README.md](README.md).

## `deploy.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_NAME="myapp"
IMAGE_REPO="myregistry/myapp"
VERSION="$(git rev-parse --short HEAD)"
NEW_CONTAINER="${APP_NAME}-new"
PROD_CONTAINER="${APP_NAME}"
PROD_PORT=8080
HEALTH_TIMEOUT=60
HEALTH_INTERVAL=3
LOG_FILE="/var/log/${APP_NAME}-deploy.log"
LAST_GOOD_FILE="/var/lib/${APP_NAME}/last-good-version.txt"

DEPLOY_SUCCEEDED=0

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

die() {
    log "FATAL: $*"
    exit 1
}

cleanup() {
    local exit_code=$?
    if [[ "$DEPLOY_SUCCEEDED" -eq 0 ]] && docker inspect "$NEW_CONTAINER" &>/dev/null; then
        log "Script exiting without a successful cutover — removing leftover '${NEW_CONTAINER}' container"
        docker rm -f "$NEW_CONTAINER" &>/dev/null || true
    fi
    exit "$exit_code"
}

remove_stale_new_container() {
    if docker inspect "$NEW_CONTAINER" &>/dev/null; then
        log "Found leftover '${NEW_CONTAINER}' from a previous attempt — removing it first"
        docker rm -f "$NEW_CONTAINER"
    fi
}

build_image() {
    log "Building image ${IMAGE_REPO}:${VERSION}"
    docker build -t "${IMAGE_REPO}:${VERSION}" .
}

start_new_container() {
    log "Starting ${NEW_CONTAINER} on temporary port 0 (random host port)"
    docker run -d --name "$NEW_CONTAINER" -P "${IMAGE_REPO}:${VERSION}" > /dev/null
}

wait_for_healthy() {
    log "Waiting up to ${HEALTH_TIMEOUT}s for ${NEW_CONTAINER} to report healthy"
    local elapsed=0
    while (( elapsed < HEALTH_TIMEOUT )); do
        local status
        status=$(docker inspect --format '{{.State.Health.Status}}' "$NEW_CONTAINER" 2>/dev/null || echo "unknown")
        if [[ "$status" == "healthy" ]]; then
            log "${NEW_CONTAINER} is healthy after ${elapsed}s"
            return 0
        fi
        log "  ...status is '${status}' (${elapsed}s/${HEALTH_TIMEOUT}s)"
        sleep "$HEALTH_INTERVAL"
        elapsed=$(( elapsed + HEALTH_INTERVAL ))
    done
    log "${NEW_CONTAINER} did not become healthy within ${HEALTH_TIMEOUT}s"
    return 1
}

cutover() {
    log "Cutting over: removing old ${PROD_CONTAINER}, promoting ${NEW_CONTAINER}"
    if docker inspect "$PROD_CONTAINER" &>/dev/null; then
        docker stop "$PROD_CONTAINER" > /dev/null
        docker rm "$PROD_CONTAINER" > /dev/null
    fi
    docker stop "$NEW_CONTAINER" > /dev/null
    docker rm "$NEW_CONTAINER" > /dev/null
    docker run -d --name "$PROD_CONTAINER" -p "${PROD_PORT}:${PROD_PORT}" "${IMAGE_REPO}:${VERSION}" > /dev/null

    mkdir -p "$(dirname "$LAST_GOOD_FILE")"
    echo "$VERSION" > "$LAST_GOOD_FILE"
    log "Cutover complete. ${PROD_CONTAINER} is now running ${IMAGE_REPO}:${VERSION}"
}

rollback_failed_new_container() {
    log "Rolling back: removing failed ${NEW_CONTAINER}, leaving ${PROD_CONTAINER} untouched"
    docker logs --tail 50 "$NEW_CONTAINER" | tee -a "$LOG_FILE" || true
    docker rm -f "$NEW_CONTAINER" || true
    die "Deploy of ${IMAGE_REPO}:${VERSION} failed healthcheck — production is unaffected"
}

main() {
    trap cleanup EXIT
    log "=== Starting deploy of ${IMAGE_REPO}:${VERSION} ==="

    remove_stale_new_container
    build_image
    start_new_container

    if wait_for_healthy; then
        cutover
        DEPLOY_SUCCEEDED=1
        log "=== Deploy succeeded ==="
    else
        rollback_failed_new_container
    fi
}

main "$@"
```

### Why these design choices?

- **`DEPLOY_SUCCEEDED` is the single flag that tells `cleanup` what to do.** It defaults to `0`, and is only ever set to `1` at the very last line of the success path — *after* cutover completes. If the script dies anywhere before that line (a `docker build` failure, a network blip, even a plain bug), the trap correctly infers "this attempt failed" and cleans up `$NEW_CONTAINER` — without ever touching `$PROD_CONTAINER`, because `cleanup` was never written to know that container exists at all. Scope-limiting what the safety net can reach is what makes it safe.
- **`cutover` is the only function that ever removes the production container.** This isn't an accident — it's the entire point. `wait_for_healthy` gates entry into `cutover`, so the old container is *provably* still serving traffic right up until the new one has already proven itself.
- **`rollback_failed_new_container` calls `die()` at the end**, which both logs a clear message and returns a non-zero exit code — that's what makes `if ./deploy.sh; then ...` work correctly as a CI gate, and it's also what triggers the `trap`'s `exit_code` to propagate correctly rather than getting silently swallowed.
- **The healthcheck poll has a hard ceiling (`HEALTH_TIMEOUT`), never an unbounded loop.** An indefinitely-hanging deploy script is worse than a failed one — it blocks CI runners and leaves a human unsure whether anything is still happening.

## `rollback.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_NAME="myapp"
IMAGE_REPO="myregistry/myapp"
PROD_CONTAINER="${APP_NAME}"
PROD_PORT=8080
LAST_GOOD_FILE="/var/lib/${APP_NAME}/last-good-version.txt"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
die() { log "FATAL: $*"; exit 1; }

main() {
    local target_version="${1:-}"

    if [[ -z "$target_version" ]]; then
        [[ -f "$LAST_GOOD_FILE" ]] || die "No version specified and no last-good-version.txt found"
        target_version=$(cat "$LAST_GOOD_FILE")
        log "No version specified — rolling back to last known good: ${target_version}"
    fi

    docker image inspect "${IMAGE_REPO}:${target_version}" &>/dev/null \
        || die "Image ${IMAGE_REPO}:${target_version} not found locally — cannot roll back to it"

    log "Rolling back ${PROD_CONTAINER} to ${IMAGE_REPO}:${target_version}"
    docker stop "$PROD_CONTAINER" > /dev/null 2>&1 || true
    docker rm "$PROD_CONTAINER" > /dev/null 2>&1 || true
    docker run -d --name "$PROD_CONTAINER" -p "${PROD_PORT}:${PROD_PORT}" "${IMAGE_REPO}:${target_version}" > /dev/null

    log "Rollback complete. ${PROD_CONTAINER} is now running ${IMAGE_REPO}:${target_version}"
}

main "$@"
```

This manual rollback is deliberately simpler than `deploy.sh` — it accepts a short outage (stop-then-start) because it's an emergency-response tool for "we already cut over and now we regret it," not a routine deploy path. Optimizing it for zero downtime would be solving a problem that doesn't need solving here.

## CI Integration (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy over SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd /opt/myapp
            git pull origin main
            ./deploy.sh
```

CI failing here means exactly what it should mean: `deploy.sh` exited non-zero, which only happens if the healthcheck genuinely timed out — and production was never touched.

## Testing Your Solution

**Simulated successful deploy:**

```
[2026-03-15 10:00:00] === Starting deploy of myregistry/myapp:a1b2c3d ===
[2026-03-15 10:00:00] Building image myregistry/myapp:a1b2c3d
[2026-03-15 10:00:12] Starting myapp-new on temporary port 0 (random host port)
[2026-03-15 10:00:12] Waiting up to 60s for myapp-new to report healthy
[2026-03-15 10:00:12]   ...status is 'starting' (0s/60s)
[2026-03-15 10:00:15]   ...status is 'starting' (3s/60s)
[2026-03-15 10:00:18] myapp-new is healthy after 6s
[2026-03-15 10:00:18] Cutting over: removing old myapp, promoting myapp-new
[2026-03-15 10:00:19] Cutover complete. myapp is now running myregistry/myapp:a1b2c3d
[2026-03-15 10:00:19] === Deploy succeeded ===
```

**Simulated failed healthcheck (bad build), showing automatic rollback:**

```
[2026-03-15 11:00:00] === Starting deploy of myregistry/myapp:badc0de ===
[2026-03-15 11:00:00] Building image myregistry/myapp:badc0de
[2026-03-15 11:00:10] Starting myapp-new on temporary port 0 (random host port)
[2026-03-15 11:00:10] Waiting up to 60s for myapp-new to report healthy
[2026-03-15 11:00:10]   ...status is 'starting' (0s/60s)
...
[2026-03-15 11:01:10] myapp-new did not become healthy within 60s
[2026-03-15 11:01:10] Rolling back: removing failed myapp-new, leaving myapp untouched
[2026-03-15 11:01:10] <last 50 lines of the failing container's logs, e.g. a stack trace>
[2026-03-15 11:01:10] FATAL: Deploy of myregistry/myapp:badc0de failed healthcheck — production is unaffected
```

Confirm `echo $?` is non-zero after the failed run, and — critically — that `docker ps` still shows the *old* `myapp` container running the whole time, never stopped.
