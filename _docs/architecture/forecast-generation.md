# Forecast generation

All in `surfbuddy-v3`, triggered by [cron](cron.md).

1. **Fetch** — `src/integrations/DMI.php` pulls forecast grids (wind-speed + wind-dir, HARMONIE_DINI_SF model) and station observations (wind dir/speed/gusts, precipitation) from the Danish Meteorological Institute's Open Data API (`opendataapi.dmi.dk`). Forecast grids are fetched as GRIB2 files (via `gribReader.php`), not through DMI's EDR JSON API — see [DMI GRIB fetch](dmi-grib-fetch.md) for why and how. `src/integrations/openMeteo.php` separately serves point forecasts via Open-Meteo.com. Raw fetched data is written under `SBconfig::DIR_FORECASTDATA`.
2. **Contour generation** — `src/marchingSquares/` runs the marching-squares algorithm (`marchingSquaresOptimized`, with `mipmap`/`mipmapLevel` for zoom levels) over the gridded wind values (`xyMatrix`) to produce contour polygons (`xyMultiPolygon`) — this is what turns raw grid data into the shaded wind-speed regions on the map.
3. **Coastline clipping** — `src/raycasting/` does point-in-polygon raycasting (`raymap`, `raymapLevel`, mip-mapped for speed) to clip wind data against land/sea outlines, using outline polygons from `src/mapTiles/mapOutline`.
4. **Tiling** — `src/mapTiles/forecastTiles` finds the newest available forecast instance/time on disk and computes tile availability; `apiHandlers/tilesHandler` serves the result as SVG or GeoJSON vector tiles at `/tiles/{model}/{instance}/{forecastTime}/{z}_{x}_{y}.svg|json` (plus `/tiles/map/{z}_{x}_{y}.svg|json` for the static coastline outline, and a `winddir.json` variant for direction).
5. **Rendering** — `surfbuddy-vue` consumes the wind-speed tiles via OpenLayers `VectorTileLayer` (crossfading between instances as new data arrives) and draws wind direction separately on a custom `CanvasLayer`. See [surfbuddy-vue](../projects/surfbuddy-vue/README.md).

`src/geometry/` (points, polygons, rectangles, bounds, projections, tile math) is shared plumbing used by steps 2-4.

Point forecasts (a single lat/lng rather than a tile) go through `apiHandlers/forecastHandler` instead, using `openMeteo.php` directly — they don't go through the marching-squares/raycasting pipeline.
