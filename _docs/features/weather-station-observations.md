# Weather Station Observations

Surfbuddy combines live observations from DMI, OpenWindMap and Netatmo in the same backend tables and frontend layer.

## Collection

- DMI station metadata is refreshed every 12 hours. Its latest wind direction, average speed, maximum speed, minimum speed, and 10-minute precipitation are fetched about every 110 seconds.
- `OpenWindMap::fetchObservations()` fetches the network-wide live response about every 110 seconds. Stations with a valid position inside Surfbuddy's practical Denmark bounding regions are discovered and stored during the same request.
- `Netatmo::fetchObservations()` fetches public Weathermap stations with wind data every 10 minutes. It retains only stations marked as Danish and positioned inside Surfbuddy's practical Denmark regions. Netatmo access is refreshed automatically from provisioned OAuth credentials.
- OpenWindMap wind speeds arrive in km/h and are converted to m/s before storage. Direction is stored in degrees without conversion.
- Netatmo wind and gust speeds also arrive in km/h and are converted to m/s. Netatmo does not provide a minimum wind value.
- Location IDs are namespaced as `dmi_*`, `openwindmap_*` and `netatmo_*`.
- Fetch transport failures are recorded in `errorsDatastore`; malformed successful responses fail loudly.

## Planned CWOP integration

TODO: Add Danish CWOP wind stations through a persistent APRS-IS listener managed by systemd on the VPS. Use NOAA's MADIS `APRSWXNETStation.txt` registry to discover Danish station IDs, then subscribe to those IDs directly instead of relying on APRS-IS geographic filters, which can miss positionless weather packets. Store valid wind observations through the existing observation pipeline and apply the same freshness rules as the other sources.

At the time of assessment, the registry contained 18 Danish CWOP stations; 16 had reported wind within 24 hours and 15 within one hour. Other Danish APRS weather stations may exist outside the CWOP registry and should be evaluated separately.

## Other source candidates

Open wind sources surveyed in September 2026 from public station lists (not tested for fresh data). "Extent" is the HARMONIE crop, lat 52–59, lon −1–25.

| Source | Active wind stations | In extent | In Denmark | New for us |
|---|---|---|---|---|
| DMI metObs (in use) | 398 with `wind_dir` | 258 | 258 | Already collected |
| DMI oceanObs | 0 (180 tide gauges) | 180 | 180 | No wind parameters |
| DWD (Germany) | 277 | 92 | 0 | ~92 |
| SMHI (Sweden) | 183 | 59 | 0 | ~59 |

- **DMI:** `opendataapi.dmi.dk/v2/oceanObs` needs no key but has only water level. The other 140 metObs stations with `wind_dir` are Greenland and the Faroe Islands.
- **DWD:** 10-minute wind files under `opendata.dwd.de/climate_environment/CDC/observations_germany/climate/10_minutes/wind/now/`, one zipped CSV per station; coordinates in `recent/zehn_min_ff_Beschreibung_Stationen.txt`. 48 of the 92 in-extent stations are north of lat 53.5, including Fehmarn, List auf Sylt, Flensburg, Schleswig, Kiel, Leck and Helgoland. Cadence fits the ~110 s fetch loop; the file format is more work than the JSON APIs used so far. Not yet checked: how many lie within e.g. 50 km of Denmark.
- **SMHI:** REST API at `opendata-download-metobs.smhi.se` (parameters 3 direction, 4 speed, 21 gust; station list has an `active` flag). Values are 10-minute averages published once per hour, so weak as live data. The 59 in-extent stations cover Skåne and the Øresund.
- **Ruled out:** Holfuy (API not open by default), WeatherFlow Tempest (multi-station use restricted), Windguru (no open station data), MET Norway Frost (needs client ID, rarely relevant). Vejdirektoratet (`api.vejdirektoratet.dk`) is unverified: unclear whether wind stations are open, likely needs a key.

## Freshness and retention

- `/observations/latest` returns only parameters measured within the last hour and only locations with wind direction data. It excludes Netatmo by default; `?experimental=netatmo` includes it. A known station therefore disappears from the map when its measurements become stale without being removed from the database.
- `/observations/location/{locationId}` returns the last 12 hours for the station graph.
- Stored observations are retained for three days and cleaned daily.

## Netatmo production readiness

**TODO: Do not expose Netatmo observations in production until their wind directions have been validated.**

The initial direction analysis covered only a few hours with relatively light wind. Many stations alternated between `359°` and other values, while comparisons with nearby DMI stations showed apparently stable station-specific offsets. This sample cannot establish whether `359°` represents an unavailable direction or whether the offsets come from installation errors, local turbulence, low-speed measurement uncertainty or insufficient variation in the prevailing wind.

Before enabling Netatmo in production:

- collect several days spanning stronger wind and materially different wind directions;
- compare only sufficiently strong, time-aligned observations with nearby DMI stations;
- require observations across multiple prevailing directions before classifying a station-specific offset;
- decide how to handle `359°`, unreliable directions and stations with persistent disagreement;
- verify that omitting an invalid direction cannot expose a stale older direction as current.

## Frontend

- The frontend refreshes `/observations/latest` every three minutes. Staging and local development opt into experimental Netatmo observations; production does not.
- Observation markers identify their source by background color: dark blue for DMI, dark red for OpenWindMap/Pioupiou and dark green for Netatmo.
- Map labels, graph-axis values, and graph tooltips display wind speed with one decimal while the stored values retain their original precision.
- Station graphs credit DMI, OpenWindMap or Netatmo according to the location ID prefix.
