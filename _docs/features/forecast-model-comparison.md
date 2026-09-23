# Forecast model comparison

The staging/development timeline can switch from separate wind, gust and direction rows to one compact row per forecast model. Each compact cell shows wind speed, gust and direction together.

## Models and labels

| Open-Meteo model | Full label | Sticky label |
| --- | --- | --- |
| `dmi_harmonie_arome_europe` | HARMONIE | DMI |
| `metno_nordic` | MET Nordic | YR |
| `icon_d2` | ICON D2 | D2 |
| `arpege_world` | ARPEGE | ARP |
| `icon_eu` | ICON EU | EU |
| `icon_global` | ICON Global | GLB |
| `ecmwf_ifs025` | ECMWF IFS | IFS |
| `ecmwf_aifs025_single` | ECMWF AIFS | AI |

The full labels occupy a 90 px column. The short labels replace the wind icons in the 30 px sticky column so the active model remains identifiable while scrolling horizontally. Forecast-section labels are hidden in comparison mode, while the section dividers remain to show the changes between hourly, three-hourly and six-hourly resolution.

## Data flow

- `TheOlMapRaw.vue` exposes the comparison toggle when `HomeView.vue` enables the multi-forecast timeline.
- `TheTimeSliderMultiForecast.vue` owns the comparison display and requests data only when comparison mode is opened.
- `ApiForecasts.getPointForecastComparison()` calls `GET /forecasts/comparison/p/{lat}/{lng}` and converts timestamps to Copenhagen time.
- `forecastHandler::pointForecastComparison()` requests all models together from Open-Meteo. The endpoint is restricted by the same staging/development origin check as the extended ECMWF point forecast.
- A coordinate change reloads comparison data while comparison mode is open. Existing rows remain visible until the replacement response arrives.

Rows follow the main timeline's timestamps. They remain hourly through HARMONIE's available period, then use three-hourly points followed by six-hourly points at the shared long-range boundary. A missing model keeps an empty row so the order and layout remain stable. AIFS has no gust values through Open-Meteo, so its compact cells show a dash and the neutral gust accent.
