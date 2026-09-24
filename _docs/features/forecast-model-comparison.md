# Forecast model comparison

The timeline can switch from separate wind, gust and direction rows to one compact row per forecast model. Each compact cell normally shows wind speed, gust and direction together. In landscape viewports below 500 px high, rows show only wind speed and direction; temperature and rain are also omitted from comparison mode.

The two row groups are mounted together at their natural size during the transition, with the comparison rows positioned behind the normal rows. At either endpoint, only the active group remains mounted. A shared progress value expands the clipped containing height, fades out the normal rows and fades in the comparison rows. A direction-locked vertical touch gesture controls progress directly; releasing it selects the nearest mode. The map button is hidden on coarse-pointer devices and drives the same progress with a 0.5-second animation on desktop.

Forecast cells and row chrome are separate layers. Row names remain part of the horizontally moving content, while one shared icon/model rail sticks to the left and one unit rail sticks to the right. The three layers use the same row definitions and transition height.

The transition unmounts inactive rows at its endpoints, applies opacity only to complete row groups during animation, and isolates cell painting. See [frontend rendering performance](../architecture/frontend-rendering-performance.md) for the constraints behind this structure.

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
- `HomeView.vue` owns both the ordinary and comparison forecast data. It calls the normal point-forecast endpoint with `comparison=1` when comparison data is needed.
- A comparison response contains the normal timeline sections and all model rows. The backend derives both from one combined Open-Meteo response.
- Switching from normal to comparison mode starts an uncached request immediately. Requests caused by map movement remain debounced in both modes.
- Existing rows remain visible beneath the loading overlay until the replacement response arrives.

Rows follow the main timeline's timestamps. They remain hourly through HARMONIE's available period, then use three-hourly points followed by six-hourly points at the shared long-range boundary. A missing model keeps an empty row so the order and layout remain stable. AIFS has no gust values through Open-Meteo, so its compact cells show a dash and the neutral gust accent.
