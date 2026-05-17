# RSSHub local Docker

Default mode is slim and is intended for the currently tracked Spotify podcast feeds. It runs only RSSHub and Redis.

## Slim mode

```bash
docker compose up -d
```

Expected services:

- `rsshub`
- `redis`

## Browser mode

Use browser mode temporarily for RSSHub routes that need JavaScript rendering, Browserless, Puppeteer, or real-browser behavior.

```bash
docker compose -f docker-compose.yml -f docker-compose.browser.yml up -d
```

Browser mode adds:

- `browserless`, using `browserless/chrome`
- `real-browser`, using `ghcr.io/hyoban/puppeteer-real-browser-hono`

## Return to slim mode

```bash
docker compose -f docker-compose.yml -f docker-compose.browser.yml down
docker compose up -d
```

If the browser images were deleted to save disk, the next browser-mode start will re-download them.

## Current protected feeds

- `/spotify/show/0JPRckNJLgiqogrrCvLjsI`, JOKAKAKA
- `/spotify/show/094kLIbO9UsVRz0kpz0gAy`, 随机波动StochasticVolatility
- `/spotify/show/6hDKSGvn1PYx8JN9cuaqri`, 波米和朋友们
- `/spotify/show/08dz9P84TfsHx73e7dwjLr`, 猫挠夜总会
- `/spotify/show/5AyxeBKaGFxleNi47OvfEl`, 展开讲讲
- `/spotify/show/2rJh3VkzJBvSsPe1KI5oqv`, No News Is Good News 今日无事发生
- `/spotify/show/4rHsOhdvFMn94U3LSsfoT8`, 史蒂夫说

## Final slim state after cleanup

Baseline before cleanup:

- Images: 4, about 8.254 GB
- Containers: 4
- Local volumes: 1

Final slim state:

- Images: 2, about 799 MB
- Containers: 2
- Local volumes: 1

Remaining images:

- `diygod/rsshub:latest`
- `redis:alpine`

Removed browser images:

- `browserless/chrome:latest`
- `ghcr.io/hyoban/puppeteer-real-browser-hono:latest`

Approximate image-space reduction: 7.45 GB.
