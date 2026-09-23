# surfbuddy-v3

The backend: a Slim 4 REST API (PHP, PSR-4 under `Mikk3lRo\SurfBuddy\`) that fetches raw weather data, turns it into wind-forecast map tiles, and serves both tiles and point forecasts. Entry point `public/index.php` → `src/api.php` (`api::init()`) builds the Slim app, adds CORS (`origin: "*"`, credentials on), and dispatches `src/apiRoutes.php`.

No README exists in the repo itself.

## Layout

- `src/apiHandlers/` — route controllers: `cronHandler` (see [cron](../../architecture/cron.md)), `forecastHandler`, `tilesHandler`, `observationsHandler`, `shareHandler`, `miscHandler`.
- `src/apiResponses/` — thin Slim response wrapper (`json`, `plainText`, cache-control helpers).
- `src/datastores/` — raw-SQL DB access per domain: `observationsDatastore`, `settingsDatastore`, `shareDatastore`, `errorsDatastore`.
- `src/integrations/` — external data sources: `DMI.php` (Danish Meteorological Institute Open Data API at `opendataapi.dmi.dk` — station observations via the EDR `metObs` endpoint; HARMONIE_DINI_SF forecast grids via the STAC `forecastdata` endpoint + `gribReader.php`, not the EDR `forecastedr`/`cube` endpoint — see [DMI GRIB fetch](../../architecture/dmi-grib-fetch.md)), `gribReader.php` (minimal GRIB2 parser used only for the Lambert Conformal + simple-packing subset DMI's HARMONIE files use — throws on anything else), `ECMWF.php` (ECMWF IFS open data — extends the map past HARMONIE's horizon; different GRIB grid/packing than DMI, decoded via a Python/eccodes subprocess rather than `gribReader.php` — see [ECMWF map fetch](../../architecture/ecmwf-map-fetch.md)), `openMeteo.php` (Open-Meteo.com, point forecasts for both DMI and ECMWF), `axesInfo.php`.
- `src/mapTiles/`, `src/marchingSquares/`, `src/raycasting/`, `src/geometry/` — the forecast-tile generation pipeline, see [forecast generation](../../architecture/forecast-generation.md). `mapTiles/tileGenerator.php` is the model-agnostic tile-writing code shared by DMI and ECMWF.
- `scripts/decode_grib.py` — decodes ECMWF's CCSDS/AEC-compressed GRIB2 messages via `pygrib`/eccodes, called as a subprocess from `ECMWF.php`. See [ECMWF map fetch](../../architecture/ecmwf-map-fetch.md).
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
