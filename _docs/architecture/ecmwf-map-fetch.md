# ECMWF map fetch

How `surfbuddy-v3` extends the wind-speed map past HARMONIE's ~60h horizon using ECMWF's IFS open data, and why it needs a different pipeline from [DMI GRIB fetch](dmi-grib-fetch.md) despite fetching GRIB2 the same basic way (HTTP Range + an index file).

## Why ECMWF IFS, not GFS

Both were evaluated for a longer-range comparison forecast. ECMWF IFS was chosen over GFS on accuracy: ECMWF is generally the more accurate global model, and its horizon (~15 days) is close enough to GFS's (~16 days) that GFS's extra length isn't worth trading away.

## What's different from DMI's GRIB pipeline

- **Grid**: plain lat/lon (Grid Definition Template 0), not DMI's Lambert Conformal "Dini" grid — no reprojection math needed, grid x/y axes are directly lon/lat. `ECMWF::toLatLng()` is the (trivial) equivalent of `projection::harmonieDini2latLng()`.
- **Index format**: JSON-lines (one JSON object per GRIB message, with `param`/`_offset`/`_length`), not DMI's wgrib-style colon-delimited text. The index path replaces `.grib2` with `.index` — it is *not* `.grib2.index`.
- **No direct wind-speed/wind-dir field** — only `10u`/`10v` vector components exist. `ECMWF::fetchWindGrid()` combines them into speed/direction itself (`sqrt`/`atan2`) after decoding.
- **Data Representation is CCSDS/AEC compression** (`packingType: grid_ccsds`), not DMI's simple packing. `gribReader.php` only implements simple packing and can't decode this — decoding is delegated to `eccodes` via a small Python subprocess, `scripts/decode_grib.py` (uses `pygrib`), because there's no PHP GRIB2 extension and hand-rolling a CCSDS/AEC decoder in PHP isn't practical. The server packages providing this (`python3-grib`, `libeccodes-tools`) are provisioned in `surfbuddy-server`'s `install.php`.

`ECMWF::fetchWindGrid()` writes the two downloaded messages to temp files under `SBconfig::DIR_TEMP`, shells out to `/usr/bin/python3 scripts/decode_grib.py <lat1> <lat2> <lon1> <lon2> <10u file> <10v file>` (absolute path — PHP-FPM's `PATH` may not include `python3`), and parses the JSON array it prints (one object per input file: `shortName`, `nx`/`ny`, grid corner coordinates, and a flat row-major `values` array, south-to-north/west-to-east regardless of the source message's own scan direction).

## Tile generation

Shared with DMI via `mapTiles/tileGenerator.php`, which was factored out of `DMI.php` to take a `callable(xyPoint): latLngPoint $toLatLng` instead of hardcoding `projection::harmonieDini2latLng()`. DMI passes that function; ECMWF passes `ECMWF::toLatLng()` (a public method — PHP checks callable visibility against the invoking scope, which is `tileGenerator`, not `ECMWF`, so it can't be `private`).

ECMWF generates tiles at a **single fixed zoom level** (`ECMWF::TILE_ZOOM`, currently 3), not DMI's full/half/quarter-rez set (zoom 5/4/3) — its native ~0.25° grid doesn't carry enough detail to benefit from multiple resolutions. The frontend's `windspeedLayer.ts` is given `minZoom === maxZoom` for ECMWF's tiles; OpenLayers' `TileGrid.getZForResolution()` clamps every view zoom to that single level automatically (confirmed in `node_modules/ol/tilegrid/TileGrid.js`), so no custom over/under-zoom handling was needed.

The available forecast steps depend on the run: 00/12 UTC runs contain 3-hourly steps through 144h and 6-hourly steps through 240h, while 06/18 UTC runs contain 3-hourly steps through 90h. ECMWF publishes a run progressively. A 404 for the next valid step means it is not available yet, so the cron run stops there without recording an error or retrying immediately; the next cron invocation resumes from that step.

The crop bounding box is much larger than DMI's Denmark-only one — Europe + North Atlantic (lat 30–75, lon -40–40) — and still cheap: ECMWF's 0.25° grid over that whole area (~58k points) is a fraction of DMI's ~2km grid over just Denmark (~560k points).

## Instance discovery

`ECMWF::newestInstance()` is anchored on whatever instance is already on disk, not on guessing publish timing from "now": it checks exactly one candidate, the instance one cycle (6h) after the current local newest, since ECMWF's publish order is strictly sequential — if that one isn't out yet, no later one is either. This is normally a single HTTP request per call (cached 10 minutes). It only falls back to a short bounded search (up to 3 candidates, 6h apart) when there's no local instance at all yet (first ever run) — deliberately not trying to catch up in one go if far behind; a stale instance just advances one cycle every 10 minutes until it's current.

## Rate limiting and mirrors

ECMWF's data servers throttle aggressively and consistently (unlike DMI's EDR 429s, which were transient) — 429 responses carry no `Retry-After` header. `ECMWF::REQUEST_SLEEP_SECONDS` (5s) is used between all of ECMWF's own outbound requests as steady pacing, and `webRequest::get()`'s calls here always pass `retries: 0` — retrying a 429 immediately just hits the same wall again.

Requests go through `ECMWF::requestWithMirrorFallback()`, which tries a short list of mirrors in order (same files, byte-identical — confirmed by matching ETag/MD5):
- `https://data.ecmwf.int/forecasts` (primary)
- `https://storage.googleapis.com/ecmwf-open-data` (fallback)

AWS (`ecmwf-forecasts.s3.eu-central-1.amazonaws.com`) and Azure (`ai4edataeuwest.blob.core.windows.net/ecmwf`) are also official mirrors (per `ecmwf-opendata`'s `urls.py`) but aren't used yet — AWS returned its own S3 "SlowDown" throttling and Azure returned unexplained 409s when checked.

On a 429, that specific mirror is blocked for 300s (`settingsDatastore` key `ecmwfRateLimitedUntil_<md5(mirror)>`, so it persists across cron invocations) and the next mirror is tried; a non-429 error (e.g. 404 - not published yet) isn't mirror-specific and isn't retried on another mirror. A 429 is logged both to the timing log and to `errorsDatastore` (visible in the same error-count alerting DMI's issues use).

## Point forecasts

`apiHandlers/forecastHandler.php`'s `pointForecast()` appends ECMWF sections (via Open-Meteo's `ecmwf_ifs025` model, not `ecmwf_seamless`) after the HARMONIE section, thinned to every 3rd hour for the first ~5 days then every 6th hour.

The thinning boundary is aligned to ECMWF's actual UTC-anchored native GRIB steps (not round Danish local hours, unlike the rest of the point-forecast display) so that each shown ECMWF row can be matched to a real generated map tile — a `mapTime` field is attached when the matched tile's actual valid time differs from the row's own (Danish-local) timestamp, shown in `TheOlMapRaw.vue`'s time indicator.

The point-forecast endpoint accepts `comparison=1` and then returns its normal sections plus wind forecasts for eight Open-Meteo models. See [forecast model comparison](../features/forecast-model-comparison.md).

## Cost

~1.5MB per forecast step (10u+10v combined, CCSDS-compressed, whole global 0.25° field — Range requests fetch by GRIB message, not by geographic crop, same limitation DMI has). The two daily 00/12 UTC runs have 65 steps and the 06/18 UTC runs have 31 steps, for roughly 0.3GB/day or 9GB/month from ECMWF's servers.
