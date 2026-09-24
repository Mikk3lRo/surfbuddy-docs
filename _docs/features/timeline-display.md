# Timeline display

Both forecast timelines show sticky day labels with abbreviated weekday and `d/M` date. The label text fades near the end of its day without painting over the timeline background. A dashed vertical divider marks each new day at the leading edge of its `00` cell; this is intentionally aligned to the cell boundary rather than the timestamp's visual center.

Night periods are calculated in the frontend for the selected coordinate and rendered as continuous background bands. Sunrise and sunset positions are interpolated between actual forecast timestamps, so the shading remains proportional across hourly, three-hourly and six-hourly sections.

Night shading stays behind row-level stacking contexts so text and colored forecast cells remain unaffected. Forecast cells must not receive individual `z-index` values; see [frontend rendering performance](../architecture/frontend-rendering-performance.md).

In landscape viewports below 500 px high, comparison mode uses 20 px model rows, hides gust values and accents, and omits temperature and rain. Normal and comparison content remain in one transition container so the vertical mode gesture does not trigger independent row animations.
