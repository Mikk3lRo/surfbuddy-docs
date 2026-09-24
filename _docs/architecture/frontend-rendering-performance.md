# Frontend rendering performance

The map and multi-model timeline contain wide or continuously animated rendering surfaces. Keep animation work isolated from OpenLayers and avoid creating compositor state per forecast cell.

## Timeline

- `TheTimeSliderMultiForecast.vue` keeps forecast data separate from row chrome. Row names scroll normally, while one shared icon/model rail and one shared unit rail provide the sticky columns.
- Ordinary rows are 20 px high, the weather-symbol row is 25 px, and compact comparison rows are 36 px. Transition height is calculated from these fixed values rather than DOM measurement.
- Normal and comparison rows coexist only while transition progress is between 0 and 1. At either endpoint, the inactive data rows and their rail content are unmounted.
- Opacity belongs on each complete row group and is present only during the transition. Never apply animated opacity or `will-change` to individual cells; doing so creates hundreds of compositor candidates and causes delayed painting during horizontal scroll.
- Forecast cells use `contain: layout paint` to limit layout and paint invalidation.
- The vertical mode gesture uses a direction lock. A gesture must move at least 10 px and its vertical distance must exceed its horizontal distance by a factor of 1.5 before it can control transition progress.

## Map sizing during timeline transitions

`.MainMap` remains the flexible layout element so map controls and forecast-run information follow the visible map area as the timeline changes height. The OpenLayers target itself has the fixed normal-view height `calc(100dvh - 190px)` and is vertically centered inside `.MainMap`. It can overflow equally above and below the flexible container in comparison mode.

Do not resize the OpenLayers target on every transition frame. Repeated map resizing forces expensive map layout and rendering even when the map content has not changed.

## Position markers

- The pulsing forecast-position ring is an OpenLayers `Overlay`, so it remains bound to its geographic coordinate while the map moves.
- The ring is a fixed 40 px SVG. Native SVG animations change only circle radius and opacity; CSS scaling or animated element dimensions produce visibly uneven motion in Firefox.
- The centered cross is an HTML overlay shown only during manual map movement. It represents the position that will be selected on release, while the geographic ring remains at the current forecast position.
- Clicking the map moves the ring to the selected coordinate and uses `View.animate()` to center the map.

Never drive marker animation by calling `map.render()` from an OpenLayers `postrender` handler. That creates a permanent full-map render loop. Marker animation must remain independent of map rendering, and render objects must not be allocated per animation frame.
