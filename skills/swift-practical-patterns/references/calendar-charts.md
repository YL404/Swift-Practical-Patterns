# Calendar chart coordinates and accessibility

Use this when a heatmap needs weekday columns and week rows while retaining the actual date of every observation. Swift Charts requires iOS/iPadOS 16+ or macOS 13+. This is guidance for coordinate preparation and accessibility decisions, not a complete calendar widget.

## Keep display coordinates separate from date identity

A [date PlottableValue](https://developer.apple.com/documentation/charts/plottablevalue/value(_:_:unit:calendar:)-2r0fo) can specify a calendar unit and calendar. A weekday unit does not itself strip the rest of the timestamp. For repeated weekday columns, map each observation into the same reference week for X and retain the real date for Y, identity, labels, and selection results.

Choose one calendar and time zone for aggregation, coordinate preparation, mark encodings, and labels. In a SwiftUI view, read `calendar`, `timeZone`, and `locale` from the environment; copy the time zone into the calendar used for these calculations. Rebuild coordinates when that configuration changes. Group observations into one value per actual local day first; reject or combine duplicate days deliberately. Missing data is not a zero measurement.

Complete Foundation helper. It intentionally maps day-level values to the start of a reference day, discarding the observation's time of day. The reference date selects only the display week; never use the returned value as record identity. Calendar failures return `nil`, not a fallback date that would silently move an observation into the wrong column. [dateInterval(of:for:)](https://developer.apple.com/documentation/foundation/calendar/dateinterval(of:for:)) and [calendar day addition](https://developer.apple.com/documentation/foundation/calendar/date(byadding:value:to:wrappingcomponents:)) avoid assuming that every local day lasts 86,400 seconds.

```swift
import Foundation

func weekdayCoordinate(
    for date: Date, referenceDate: Date, calendar: Calendar
) -> Date? {
    guard let week = calendar.dateInterval(of: .weekOfYear, for: referenceDate)
    else { return nil }
    let weekday = calendar.component(.weekday, from: date)
    let offset = (weekday - calendar.firstWeekday + 7) % 7
    guard let projected = calendar.date(byAdding: .day, value: offset, to: week.start),
          projected < week.end,
          calendar.component(.weekday, from: projected) == weekday else { return nil }
    return projected
}
```

After handling a failed projection, use the returned date in the X `.value` with `unit: .day` and the same `calendar`. Use the real observation date in Y with `unit: .weekOfYear`. Set the X domain to that entire reference week, including absent weekdays, rather than allowing a partial dataset to choose the column span. Preserve gaps in missing weeks as well. If exact equal-height rows matter across daylight-saving changes, use explicit ordinal week rows or verify the temporal scale's geometry; a week is not always a fixed number of seconds.

For position-based `RectangleMark` dimensions, [.inset(1)](https://developer.apple.com/documentation/charts/markdimension/inset(_:)) removes one screen-space point on each side. It changes the visible tile, not the date or measurement. Apply a `7 / rowCount` aspect ratio to the **plot area** with [chartPlotStyle](https://developer.apple.com/documentation/swiftui/view/chartplotstyle(content:)) when using seven equal-width columns and equal-height rows. Applying it to a chart that also contains axes or a legend does not by itself establish square tiles. Pixel sizes, text fit, and month-boundary offsets need layout checks at the actual available width.

## Decide whether native accessibility matches the data

Give every data mark its real date and value. Test repeated weekday columns with VoiceOver: the source reports column grouping on iOS, but that is an observed configuration, not a guarantee for every release.

If the native hierarchy merges days incorrectly, [accessibilityChildren](https://developer.apple.com/documentation/swiftui/view/accessibilitychildren(children:)) replaces the element's children with synthetic elements; SwiftUI does not display their views. Applied inside `chartPlotStyle`, the replacement concerns the plot-area content rather than the entire chart. This API contract does not establish that a particular synthetic grid has correct focus frames, traversal order, scrolling, selection, or Audio Graph behavior.

Build any synthetic layout from the same complete calendar slots as the visual plot. Padding just the beginning and end of a sparse sample array shifts all dates after an internal missing day. Keep a slot for every absent local day, hide its synthetic element, and use stable date identities for real data. Include the year where a range crosses years. Month headings can improve navigation, but their presence and focus geometry still require assistive-technology checks. Until that replacement is verified, provide an accessible dated list of the original values instead of claiming that marks alone make the calendar accessible.

## Evidence and validation limits

Official documentation checked 2026-10-24. The exact helper passed Swift 6 typechecking with complete strict concurrency, nonisolated default isolation, warnings as errors, and no additional upcoming features, using Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), macOS 27.0 SDK targeting arm64 macOS 13, and iOS Simulator 27.1 SDK targeting arm64 iOS 16. This does not verify historical toolchains or execution on those older deployment targets.

On macOS 27.0.1 arm64, 720 projections passed across Gregorian/ISO 8601 calendars, Sunday/Monday first weekdays, UTC/Los Angeles/Berlin/Shanghai time zones, and dates around spring/autumn changes and year boundaries. Checks preserved weekdays and repeated-week coordinates; separate checks covered a 23-hour local day and a missing internal observation without collapsing its coordinate gap. Other calendars and unusual reference weeks need their own checks.

A separate ImageRenderer/ChartProxy probe used 21 consecutive UTC dates and weekday-unit X encodings in a 420×180 chart. Three Mondays had X positions `0, 140, 280` with original dates and approximately `60, 60, 60` after projection. Static images showed the staircase and aligned columns respectively. The day-unit encoding and explicit reference-week domain described above were typechecked; that additional configuration was not rendered. No synthetic accessibility grid was installed. Square tiles, month offsets, VoiceOver/Audio Graph, iOS execution, scrolling, selection, live environment changes, and performance remain unverified.

## Sources

- [Building an accessible calendar chart with Swift Charts](https://nilcoalescing.com/blog/BuildingAnAccessibleCalendarInSwiftCharts/) (2026-08-27)
