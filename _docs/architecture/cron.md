# Cron

There is no cron daemon inside `surfbuddy-v3` — scheduling lives in `surfbuddy-server` and triggers `surfbuddy-v3` over plain HTTP:

1. `surfbuddy-server`'s `/etc/cron.d/cron` (deployed from `modified_files/etc/cron.d/cron`) runs `scripts/cron.php` every minute.
2. `cron.php` just does `file_get_contents('https://backend.surfbuddy.dk/cron')` and exits — no auth visible on that route.
3. That hits `surfbuddy-v3`'s `GET /cron` → `cronHandler::handleCron()`, which does table bootstrap, error-log emailing (PHPMailer), cache cleanup, refreshes DMI, OpenWindMap and configured Netatmo station observations, then calls `updateForecasts()` (DMI) and `updateEcmwfForecasts()` (ECMWF). See [weather station observations](../features/weather-station-observations.md) for observation timing and filtering.
4. `updateForecasts()` gets the available DMI steps from the forecastdata STAC catalog before calling `DMI::fetchAndWriteInstanceForecastTime()`. `updateEcmwfForecasts()` gets the published ECMWF steps from the run's file catalog before calling `ECMWF::fetchAndWriteInstanceStep()`. Neither forecast horizon is hardcoded. Both loops are `sleep()`-throttled and self-limited to roughly 45 seconds per invocation, so the next minute's hit continues partial work. Their fetch locks are cleared in `finally` and also expire after 15 minutes, so an externally terminated PHP process cannot stop later updates permanently. Replacing a stale lock is recorded in `errorsDatastore`. Old forecast-instance directories are kept for 48h.

The observed steps are stored per model and UTC run hour and supplemented from local forecast directories. Catalog failure therefore falls back to the last known schedule instead of blocking normal fetches. The first run establishes a baseline; a newly published step in a later run is recorded in `errorsDatastore` and the monitored PHP error log, so provider horizon or cadence expansions trigger an email without blocking automatic use of the new data.

`cron.php` also accepts a URL argument; `cron.d/cron` uses it to hit `/cron` on `docs.surfbuddy.dk` and `docsstaging.surfbuddy.dk` (Scan Organizer) every minute.

Net effect: forecast freshness is bounded by the 1-minute cron interval plus however many minutes `updateForecasts()`/`updateEcmwfForecasts()` need to catch up after a gap.
