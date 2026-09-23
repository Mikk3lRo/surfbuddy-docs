# Project Landscape

Surfbuddy is a wind forecast site for windsurfers, split across three independent repositories.

## Repositories

- [surfbuddy-server](surfbuddy-server/README.md) — provisioning code for the single VPS that hosts everything; also triggers the backend's cron.
- [surfbuddy-v3](surfbuddy-v3/README.md) — the API: fetches weather data and generates the wind-forecast map tiles.
- [surfbuddy-vue](surfbuddy-vue/README.md) — the frontend that renders those tiles on a map.

## How they connect

```
surfbuddy-server (cron, every minute)
        │  HTTP GET https://backend.surfbuddy.dk/cron
        ▼
surfbuddy-v3 (backend.surfbuddy.dk / api.surfbuddy.dk)
        ▲  HTTP (CORS, no auth)
        │  /forecasts, /tiles, /observations, /share
surfbuddy-vue (frontend.surfbuddy.dk / surfbuddy.dk / app.surfbuddy.dk)
```

`surfbuddy-server` also hosts vhosts for both the other repos' Apache/PHP-FPM setup, but doesn't call surfbuddy-vue directly — the browser does.

## Conventions

Read [_docs/README.md](../README.md) first — it lists the unconditional conventions that apply to every repo.
