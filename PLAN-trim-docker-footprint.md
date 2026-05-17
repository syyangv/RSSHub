# PLAN: Trim RSSHub Docker Footprint

## Goal

Reduce RSSHub Docker disk usage while preserving currently used RSSHub functionality and keeping a documented path to restore browser-backed RSSHub routes when needed.

Current evidence shows RSSHub is only actively serving Spotify podcast routes:

| Feed                              | Route                                  |
| --------------------------------- | -------------------------------------- |
| JOKAKAKA                          | `/spotify/show/0JPRckNJLgiqogrrCvLjsI` |
| 随机波动 StochasticVolatility     | `/spotify/show/094kLIbO9UsVRz0kpz0gAy` |
| 波米和朋友们                      | `/spotify/show/6hDKSGvn1PYx8JN9cuaqri` |
| 猫挠夜总会                        | `/spotify/show/08dz9P84TfsHx73e7dwjLr` |
| 展开讲讲                          | `/spotify/show/5AyxeBKaGFxleNi47OvfEl` |
| No News Is Good News 今日无事发生 | `/spotify/show/2rJh3VkzJBvSsPe1KI5oqv` |
| 史蒂夫说                          | `/spotify/show/4rHsOhdvFMn94U3LSsfoT8` |

Spotify routes do not need Browserless or real-browser services.

## Current Docker footprint

Running containers observed before planning:

- `rsshub-rsshub-1`, `diygod/rsshub`, exposed on `localhost:1200`
- `rsshub-redis-1`, `redis:alpine`
- `rsshub-browserless-1`, `browserless/chrome`
- `rsshub-real-browser-1`, `ghcr.io/hyoban/puppeteer-real-browser-hono`, exposed on `localhost:3001`

Large removable images if browser routes are not needed by default:

- `browserless/chrome`, about 4.59 GB
- `ghcr.io/hyoban/puppeteer-real-browser-hono`, about 2.86 GB

Expected savings: about 7.45 GB if both browser images are removed after restore mode is documented and verified.

## Important review corrections

The initial plan only handled `PLAYWRIGHT_WS_ENDPOINT`. Actual RSSHub container environment also includes browser variables from `.env`:

- `PUPPETEER_WS_ENDPOINT=ws://browserless:3000`
- `PUPPETEER_REAL_BROWSER_SERVICE=http://real-browser:3000`

Therefore the execution must handle all browser-related variables, not just `PLAYWRIGHT_WS_ENDPOINT`.

`rsshub-real-browser-1` is currently an orphan relative to the visible `docker-compose.yml`, so execution must identify its source before deleting it.

## Constraints

- Do not remove Redis cache volume unless explicitly requested.
- Do not delete credentials or `.env`.
- Do not print or commit Spotify secrets from `.env`.
- Keep a browser-mode restore path before removing browser images.
- Make changes reversible.
- Avoid broad destructive Docker cleanup commands such as `docker system prune`.
- Expect brief RSSHub downtime during Compose restarts.

## Phase 1: Baseline and protect current feeds

Record current Docker usage:

```bash
docker system df
docker ps
docker images
docker volume ls
```

Verify all current Spotify feeds work before editing Compose:

```bash
curl -fsS http://localhost:1200/spotify/show/0JPRckNJLgiqogrrCvLjsI >/tmp/rsshub-jokakaka.xml
curl -fsS http://localhost:1200/spotify/show/094kLIbO9UsVRz0kpz0gAy >/tmp/rsshub-suijibodong.xml
curl -fsS http://localhost:1200/spotify/show/6hDKSGvn1PYx8JN9cuaqri >/tmp/rsshub-bomi.xml
curl -fsS http://localhost:1200/spotify/show/08dz9P84TfsHx73e7dwjLr >/tmp/rsshub-maonao.xml
curl -fsS http://localhost:1200/spotify/show/5AyxeBKaGFxleNi47OvfEl >/tmp/rsshub-zhankaijiangjiang.xml
curl -fsS http://localhost:1200/spotify/show/2rJh3VkzJBvSsPe1KI5oqv >/tmp/rsshub-nonews.xml
curl -fsS http://localhost:1200/spotify/show/4rHsOhdvFMn94U3LSsfoT8 >/tmp/rsshub-steve.xml
```

Acceptance:

- All seven feed requests return HTTP 200.
- Baseline Docker disk usage is recorded.
- Redis volume name is recorded.

## Phase 1.5: Discover real-browser ownership and active browser variables

Identify every Compose/config source that can activate browser services:

```bash
cd /Users/syang/Documents/GitHub/RSSHub
find . -maxdepth 3 -type f \( -name '*compose*.yml' -o -name '*compose*.yaml' -o -name '.env' -o -name '*.env' \) -print
```

Inspect the orphan real-browser container:

```bash
docker inspect rsshub-real-browser-1
```

Inspect current RSSHub env without printing secrets in the final report:

```bash
docker inspect rsshub-rsshub-1 --format '{{json .Config.Env}}'
```

Classify browser variables:

- Browserless variables:
    - `PLAYWRIGHT_WS_ENDPOINT`
    - `PUPPETEER_WS_ENDPOINT`
- Real-browser variables:
    - `PUPPETEER_REAL_BROWSER_SERVICE`

Acceptance:

- The source of `rsshub-real-browser-1` is known, or it is explicitly marked as an orphan.
- Browser-related variables are identified.
- Spotify credentials are not copied into the plan, logs, or final report.

## Phase 2: Make the default Compose stack slim

Edit `/Users/syang/Documents/GitHub/RSSHub/docker-compose.yml` so default mode starts only:

- `rsshub`
- `redis`

Remove browser-specific default configuration from the `rsshub` service:

- `PLAYWRIGHT_WS_ENDPOINT`
- `PUPPETEER_WS_ENDPOINT`
- `PUPPETEER_REAL_BROWSER_SERVICE`
- `depends_on: browserless`
- `depends_on: real-browser`, if present in any active config

Remove browser service definitions from default Compose:

- `browserless`
- `real-browser`, if present in any active config

If browser variables currently live in `.env`, do not delete `.env`. Instead:

1. Copy browser-only variables into override files in Phase 2.5.
2. Remove or comment only the browser-only lines in `.env` after overrides exist.
3. Leave Spotify credentials untouched.

Acceptance:

- `docker compose config` succeeds.
- Default Compose config includes only `rsshub` and `redis`.
- Default Compose config no longer references Browserless or real-browser endpoints.
- Redis remains configured.
- Spotify credentials remain in place but are not exposed.

## Phase 2.5: Preserve browser-mode restore path

Create `/Users/syang/Documents/GitHub/RSSHub/docker-compose.browser.yml` to restore both browser backends on demand:

```yaml
services:
    rsshub:
        environment:
            PLAYWRIGHT_WS_ENDPOINT: 'ws://browserless:3000'
            PUPPETEER_WS_ENDPOINT: 'ws://browserless:3000'
            PUPPETEER_REAL_BROWSER_SERVICE: 'http://real-browser:3000'
        depends_on:
            - browserless
            - real-browser

    browserless:
        image: browserless/chrome
        restart: unless-stopped
        ulimits:
            core:
                hard: 0
                soft: 0
        healthcheck:
            test: ['CMD', 'curl', '-f', 'http://localhost:3000/pressure']
            interval: 30s
            timeout: 10s
            retries: 3

    real-browser:
        image: ghcr.io/hyoban/puppeteer-real-browser-hono
        restart: unless-stopped
        ports:
            - '3001:3000'
```

Add local usage notes to `/Users/syang/Documents/GitHub/RSSHub/README-local-docker.md`:

```bash
# Slim mode, Spotify feeds only
docker compose up -d

# Temporary browser mode, for JS-heavy RSSHub routes
docker compose -f docker-compose.yml -f docker-compose.browser.yml up -d

# Return to slim mode
docker compose -f docker-compose.yml -f docker-compose.browser.yml down
docker compose up -d
```

Acceptance:

- Slim mode is documented.
- Browser restore mode is documented.
- Browser restore mode restores both Browserless and real-browser support.
- Browser restore mode can be enabled later without rediscovering config.

## Phase 3: Restart slim stack and verify feeds

This causes brief RSSHub downtime.

Run:

```bash
cd /Users/syang/Documents/GitHub/RSSHub
docker compose down --remove-orphans
docker compose up -d
```

Verify health:

```bash
curl -fsS http://localhost:1200/healthz
```

Verify all seven Spotify feeds again.

Acceptance:

- `docker ps` shows only `rsshub-rsshub-1` and `rsshub-redis-1` for this project.
- RSSHub health endpoint returns success.
- All seven Spotify feeds return HTTP 200.

## Phase 4: Verify browser restore mode before deleting images

Run:

```bash
cd /Users/syang/Documents/GitHub/RSSHub
docker compose -f docker-compose.yml -f docker-compose.browser.yml up -d
```

Verify:

```bash
docker ps
curl -fsS http://localhost:1200/healthz
```

Then return to slim mode:

```bash
docker compose -f docker-compose.yml -f docker-compose.browser.yml down
docker compose up -d
```

Acceptance:

- Browser mode starts successfully.
- Both `browserless` and `real-browser` services start in browser mode.
- Slim mode can be restored successfully.
- Current Spotify feeds still work after returning to slim mode.

## Phase 5: Remove unused browser containers and images

Only after Phase 4 passes, remove unused browser containers and images.

Target containers, if still present:

- `rsshub-browserless-1`
- `rsshub-real-browser-1`

Target images:

- `browserless/chrome`
- `ghcr.io/hyoban/puppeteer-real-browser-hono`

Use targeted deletion only. Do not use broad prune commands.

Example sequence:

```bash
cd /Users/syang/Documents/GitHub/RSSHub
docker compose down --remove-orphans
docker rm rsshub-browserless-1 rsshub-real-browser-1
docker image rm browserless/chrome
docker image rm ghcr.io/hyoban/puppeteer-real-browser-hono
docker compose up -d
```

If `docker rm` reports a container does not exist, continue.
If `docker image rm` reports an image is still used, inspect the referenced container before retrying.

Acceptance:

- Browser images no longer appear in `docker images`.
- Redis image and RSSHub image remain.
- Redis volume remains.
- RSSHub slim mode is running.
- All seven Spotify feeds still work.

## Phase 6: Confirm savings and document final state

Run:

```bash
docker system df
docker ps
docker images
docker volume ls
```

Document in `/Users/syang/Documents/GitHub/RSSHub/README-local-docker.md`:

- Final running containers.
- Final images.
- Disk saved.
- Slim mode command.
- Browser restore mode command.
- Known tradeoff: first browser-mode restore after image deletion will re-download large images.

## Done criteria

- Default RSSHub stack runs only RSSHub and Redis.
- All seven current Spotify feeds still work.
- Browser functionality has a documented override restore path for both Browserless and real-browser.
- Heavy browser images are removed only after restore mode is verified.
- Disk savings are confirmed.
- No secrets are printed, committed, or copied into docs.
