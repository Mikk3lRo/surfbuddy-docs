# DMI GRIB fetch

How `surfbuddy-v3` gets the HARMONIE_DINI_SF wind-speed/wind-dir grid for each forecast step, and why it doesn't use DMI's EDR API for this.

## Why not the EDR `/cube` endpoint

DMI retired `dmigw.govcloud.dk` on 2026-06-30 in favour of `opendataapi.dmi.dk`. Since that migration, the EDR `forecastedr/.../cube` endpoint has returned HTTP 429 ("Server is busy. Please try again later.") for a large share of requests — persistently, for the same request repeated minutes apart, regardless of request size (`/position` queries for a single point fail the same way) or client-side request rate (well under DMI's documented fair-use limit of 500 requests/5 seconds). This looks like backend capacity, not something fixable by throttling our own requests harder.

## What's used instead

Forecast grids are fetched from DMI's GRIB2 files instead:

1. `DMI::findGribFile()` queries the STAC `forecastdata` API (`/v1/forecastdata/collections/harmonie_dini_sf/items?modelRun=...&datetime=...`) for the file covering one forecast step. This is a separate, lightweight metadata endpoint from EDR and has not been observed to 429.
2. The GRIB file itself (~600MB, all parameters, full model domain — Northwestern Europe/North Atlantic, not just Denmark) is a static file on S3 (`dmi-opendata.s3.eu-north-1.amazonaws.com`), not served by DMI's own API gateway at all.
3. `gribReader::fetchIndex()` fetches the file's `.index` (a small wgrib-style text file with per-message byte offsets) and locates the `WIND` / `WDIR` messages at "10 m above ground".
4. `gribReader::fetchRange()` downloads only those two GRIB2 messages via HTTP Range requests (~15MB combined per forecast step, for the whole domain — there's no way to request a smaller geographic area at this stage, unlike EDR's `bbox` parameter).
5. `gribReader` decodes the two messages, and `DMI::fetchWindGrid()` crops the result down to the same default Denmark bounding box the EDR `/cube` call used to request server-side (crop now happens client-side, after decoding, since a GRIB2 message is one indivisible compressed block).

Station observations (`DMI::fetchObservations()`/`fetchStations()`) and point forecasts (`openMeteo.php`) are unaffected by any of this — they still use the EDR `metObs` endpoint and Open-Meteo respectively, neither of which has shown the same congestion.

## Format assumptions

`gribReader.php` is not a general GRIB2 library — it implements exactly the subset DMI's current HARMONIE_DINI_SF files use, and throws (`forecastException`) rather than silently producing wrong data if any of these don't hold:

- Grid Definition Template 3.30 (Lambert Conformal) only.
- Data Representation Template 0 (simple packing) only — no JPEG2000/PNG/complex packing.
- No bitmap section (every grid point has a value).
- Scanning mode `0x40` only (row-major, +i, +j — i.e. column index increases eastward, row index increases northward, matching `surfbuddy-v3`'s `xyMatrix`/`axesInfo` conventions used elsewhere in the pipeline).

All four were confirmed against DMI's actual files as of 2026-09-23. The grid's Lambert Conformal parameters (spherical earth, radius 6,371,229m; standard parallel 55.5°; origin longitude -8°) match `src/geometry/projection.php`'s `latLng2harmonieDini()`/`harmonieDini2latLng()` exactly, so that existing projection code is reused as-is to locate the grid origin and crop bounds — no separate Lambert Conformal math was implemented.

## Cost

Per forecast step: ~15MB downloaded (uncropped, full-domain WIND+WDIR messages), ~1.6s to decode both (3,061,036 points each), ~190MB peak PHP memory (`DMI::fetchWindGrid()` raises `memory_limit` to 512M for this, since the default 128M isn't enough). Negligible against the existing per-step `sleep(15)` pacing in `cronHandler::updateForecasts()`.

## Validated against

Decoded values were checked against DMI's own EDR `/position` endpoint for the same instance/point/time (when it wasn't 429'ing) — exact match to float precision. Also cross-checked against Open-Meteo's independently-hosted copy of the same HARMONIE model as an early sanity check, before EDR itself was confirmed reachable.
