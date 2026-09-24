# Weather Station Observations

Surfbuddy combines live observations from DMI and OpenWindMap in the same backend tables and frontend layer.

## Collection

- DMI station metadata is refreshed every 12 hours. Its latest wind direction, average speed, maximum speed, minimum speed, and 10-minute precipitation are fetched about every 110 seconds.
- `OpenWindMap::fetchObservations()` fetches the network-wide live response about every 110 seconds. Stations with a valid position inside Surfbuddy's practical Denmark bounding regions are discovered and stored during the same request.
- OpenWindMap wind speeds arrive in km/h and are converted to m/s before storage. Direction is stored in degrees without conversion.
- Location IDs are namespaced as `dmi_*` and `openwindmap_*`.
- Fetch transport failures are recorded in `errorsDatastore`; malformed successful responses fail loudly.

## Freshness and retention

- `/observations/latest` returns only parameters measured within the last hour and only locations with wind direction data. A known station therefore disappears from the map when its measurements become stale without being removed from the database.
- `/observations/location/{locationId}` returns the last 12 hours for the station graph.
- Stored observations are retained for three days and cleaned daily.

## Frontend

- The frontend refreshes `/observations/latest` every three minutes.
- Map labels, graph-axis values, and graph tooltips display wind speed with one decimal while the stored values retain their original precision.
- Station graphs credit DMI or OpenWindMap according to the location ID prefix.

