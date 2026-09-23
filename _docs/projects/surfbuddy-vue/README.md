# surfbuddy-vue

The frontend: Vue 3.5 + TypeScript, built with Vite 6. No state-management library (Pinia/Vuex) — state lives in component refs/composables. Also wrapped as a mobile app via Capacitor (`android/`, `ios/`) and installable as a PWA (`vite-plugin-pwa`). Package manager is pnpm.

README in the repo is the unmodified Vite/Vue scaffold text — no project-specific docs there.

## Layout

- `src/main.ts`, `src/App.vue` — entry point and root component.
- `src/router/index.ts` — 3 routes: `/` (home), `/:base64position` (a specific/shared map position, reuses `HomeView`), `/dev/GeneratePwaAssets` (dev utility).
- `src/views/HomeView.vue` — the main (only real) app view.
- `src/components/` — `TheOlMapRaw.vue` (core map), `TheTimeSlider.vue`, `ShareDialog.vue`, `Cell*.vue` (forecast value cells: wind/gust/direction/temperature/rain), `Button.vue`, `TheLogo.vue`.
- `src/classes/` — OpenLayers layer classes (`windspeedLayer.ts`, `winddirectionLayer.ts`, `crosshairLayer.ts`, `observationsLayer.ts`) and map helpers (`latlng.ts`, `axes.ts`, `windGrid.ts`).
- `src/api/` — `ApiRequests.ts` (fetch wrapper), `ApiForecasts.ts`, `ApiShare.ts`.

## Map rendering

OpenLayers (`ol` + `ol-ext`), not Leaflet/Mapbox. Wind speed comes from the backend as vector tiles (`{z}_{x}_{y}.json`) rendered via `VectorTileLayer` with a crossfade between instances; wind direction is drawn on a custom `CanvasLayer`. See [forecast generation](../../architecture/forecast-generation.md) for how the backend produces these tiles.

## Backend communication

`src/api/ApiRequests.ts` hardcodes the API base URL (`https://backend.surfbuddy.dk`) — no `.env`, no dev/staging switch, no Vite proxy config. Endpoints consumed: `/forecasts/{model}/latest`, `/forecasts/{model}/p/{lat}/{lng}`, `/observations/latest`, `/observations/location/{id}`, plus the share endpoint (`ApiShare.ts`) that returns a token used to build `https://surfbuddy.dk/s-{token}` links.

## No auth

No login/auth code exists anywhere in this repo. The only "token" concept is the share-link token, which is a content-sharing mechanism, not a user session.

## Build/deploy

`npm run build` type-checks then runs `vite build`. No Dockerfile, no CI. Deploy is manual: `deploy`/`stage` npm scripts `rsync`+`ssh` (via a bundled Windows `cwrsync`) straight to `frontend.surfbuddy.dk`/`staging.surfbuddy.dk`.
