# surfbuddy-vue

The frontend: Vue 3.5 + TypeScript, built with Vite 6. No state-management library (Pinia/Vuex) — state lives in component refs/composables. Also wrapped as a mobile app via Capacitor (`android/`, `ios/`) and installable as a PWA (`vite-plugin-pwa`). Package manager is pnpm.

README in the repo is the unmodified Vite/Vue scaffold text — no project-specific docs there.

## Layout

- `src/main.ts`, `src/App.vue` — entry point and root component.
- `src/router/index.ts` — 3 routes: `/` (home), `/:base64position` (a specific/shared map position, reuses `HomeView`), `/dev/GeneratePwaAssets` (dev utility).
- `src/views/HomeView.vue` — the main (only real) app view.
- `src/components/` — `TheOlMapRaw.vue` (core map), `TheTimeSlider.vue` (HARMONIE-only, used in production), `TheTimeSliderMultiForecast.vue` (adds the ECMWF section/section-labels/section-dividers on top of the same steps; staging/dev only, see below), `ShareDialog.vue`, `Cell*.vue` (forecast value cells: wind/gust/direction/symbol/temperature/rain), `Button.vue`, `TheLogo.vue`.
- `src/classes/` — OpenLayers layer classes (`windspeedLayer.ts`, `winddirectionLayer.ts`, `crosshairLayer.ts`, `observationsLayer.ts`) and map helpers (`latlng.ts`, `axes.ts`, `windGrid.ts`).
- `src/api/` — `ApiRequests.ts` (fetch wrapper), `ApiForecasts.ts`, `ApiShare.ts`.

## Map rendering

OpenLayers (`ol` + `ol-ext`), not Leaflet/Mapbox. Wind speed comes from the backend as vector tiles (`{z}_{x}_{y}.json`) rendered via `VectorTileLayer` (`windspeedLayer.ts`) with a crossfade between instances; wind direction is drawn on a custom `CanvasLayer`. `windspeedLayer.addLayer()` takes a `minZoom`/`maxZoom` per model (from the forecast step's `tileMinZoom`/`tileMaxZoom`, set by the backend) — HARMONIE has 3 pre-generated zoom levels, ECMWF has 1 (`minZoom === maxZoom`, which makes OpenLayers reuse that single tile at any view zoom automatically). See [forecast generation](../../architecture/forecast-generation.md) for how the backend produces these tiles.

## Staging-only features

`HomeView.vue` sets `showMultiForecastSlider` (and similar flags elsewhere, e.g. `ShareDialog.vue`'s dev-mode gate) from `import.meta.env.MODE === 'development' || window.location.host == 'staging.surfbuddy.dk'`, and swaps in the staging/dev-only component instead of the production one. This is a *display* gate only — the backend independently gates the underlying ECMWF point-forecast data server-side (by `Origin`/`Referer` header, since `backend.surfbuddy.dk` is shared by both the live and staging frontends), so the extra data isn't served to the live app regardless of what the frontend does. See [ECMWF map fetch](../../architecture/ecmwf-map-fetch.md#point-forecasts-staging-dev-only).

## Backend communication

`src/api/ApiRequests.ts` hardcodes the API base URL (`https://backend.surfbuddy.dk`) — no `.env`, no dev/staging switch, no Vite proxy config. Endpoints consumed: `/forecasts/{model}/latest`, `/forecasts/{model}/p/{lat}/{lng}`, `/observations/latest`, `/observations/location/{id}`, plus the share endpoint (`ApiShare.ts`) that returns a token used to build `https://surfbuddy.dk/s-{token}` links.

## No auth

No login/auth code exists anywhere in this repo. The only "token" concept is the share-link token, which is a content-sharing mechanism, not a user session.

## Build/deploy

`npm run build` type-checks then runs `vite build`. No Dockerfile, no CI. Deploy is manual: `deploy`/`stage` npm scripts `rsync`+`ssh` (via a bundled Windows `cwrsync`) straight to `frontend.surfbuddy.dk`/`staging.surfbuddy.dk`.
