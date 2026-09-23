# surfbuddy-v3

The backend: a Slim 4 REST API (PHP, PSR-4 under `Mikk3lRo\SurfBuddy\`) that fetches raw weather data, turns it into wind-forecast map tiles, and serves both tiles and point forecasts. Entry point `public/index.php` → `src/api.php` (`api::init()`) builds the Slim app, adds CORS (`origin: "*"`, credentials on), and dispatches `src/apiRoutes.php`.

No README exists in the repo itself.

## Layout

- `src/apiHandlers/` — route controllers: `cronHandler` (see [cron](../../architecture/cron.md)), `forecastHandler`, `tilesHandler`, `observationsHandler`, `shareHandler`, `miscHandler`.
- `src/apiResponses/` — thin Slim response wrapper (`json`, `plainText`, cache-control helpers).
- `src/datastores/` — raw-SQL DB access per domain: `observationsDatastore`, `settingsDatastore`, `shareDatastore`, `errorsDatastore`.
- `src/integrations/` — external data sources: `DMI.php` (Danish Meteorological Institute govcloud API, HARMONIE_DINI_SF model — forecast grids + station observations), `openMeteo.php` (Open-Meteo.com, point forecasts), `axesInfo.php`.
- `src/mapTiles/`, `src/marchingSquares/`, `src/raycasting/`, `src/geometry/` — the forecast-tile generation pipeline, see [forecast generation](../../architecture/forecast-generation.md).
- `src/middlewares/` — `jsonMiddleware` (default JSON content negotiation).
- `src/SBconfig.php` — static config class: paths, DMI API key, DB credentials, `isMainServer()` host check. Not loaded from `.env` — see Secrets below.
- `public/svg-map/` — empty in the working tree; runtime-generated tile output directory.
- `_dev/` — dev/test scripts and art assets, not part of the served app. Ignore.
- `public/strobe/` — ignore

## Database

MySQL/MariaDB (database `SB`) via the author's own `mikk3lro/atomix-databases` PDO wrapper, raw SQL (no ORM). See [database](../../architecture/database.md).

## How it relates to the other repos

- Serves the frontend (`surfbuddy-vue`) directly over HTTP/CORS: `/forecasts/...`, `/tiles/...`, `/observations/...`, `/share` (see frontend's `src/api/`).
- Its `/cron` route is invoked every minute by `surfbuddy-server`'s system cron, not scheduled by this repo itself.

## Secrets committed in this repo

Not read in detail, not to be exposed — `src/SBconfig.php` hardcodes DB credentials and the DMI API key in plaintext, committed to the repo. No `.env`/`.env.example` exists.
