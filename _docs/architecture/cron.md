# Cron

There is no cron daemon inside `surfbuddy-v3` — scheduling lives in `surfbuddy-server` and triggers `surfbuddy-v3` over plain HTTP:

1. `surfbuddy-server`'s `/etc/cron.d/cron` (deployed from `modified_files/etc/cron.d/cron`) runs `scripts/cron.php` every minute.
2. `cron.php` just does `file_get_contents('https://backend.surfbuddy.dk/cron')` and exits — no auth visible on that route.
3. That hits `surfbuddy-v3`'s `GET /cron` → `cronHandler::handleCron()`, which does table bootstrap, error-log emailing (PHPMailer), cache cleanup, refreshes DMI station observations, then calls `updateForecasts()` (DMI) and `updateEcmwfForecasts()` (ECMWF).
4. `updateForecasts()` loops `DMI::newestInstance()` / `DMI::fetchAndWriteInstanceForecastTime()` for up to 60 forecast steps; `updateEcmwfForecasts()` loops `ECMWF::newestInstance()` / `ECMWF::fetchAndWriteInstanceStep()` for ECMWF's 85 native steps (3-hourly to 144h, 6-hourly to 360h) — see [ECMWF map fetch](ecmwf-map-fetch.md). Both are `sleep()`-throttled and self-limited to roughly 45 seconds per invocation (so a single minute's cron hit only makes partial progress; the next minute's hit picks up where it left off), and both guard their own "already fetching" lock flag (`isFetchingForecasts`/`isFetchingEcmwfForecasts`) with a `finally` block so an uncaught error can't leave it stuck forever. Old forecast-instance directories (either model) are cleaned up after, kept for 48h.

Net effect: forecast freshness is bounded by the 1-minute cron interval plus however many minutes `updateForecasts()`/`updateEcmwfForecasts()` need to catch up after a gap.
