# Dimming part of an area chart

Use this when an elapsed-time overlay should follow an area/line silhouette while leaving other chart content outside the mask. Requires Swift Charts on iOS/iPadOS 16+ or macOS 13+. This supplements basic mark, axis, and selection guidance.

## Keep the mask in chart coordinates

[ChartContent.mask](https://developer.apple.com/documentation/charts/chartcontent/mask(content:)) takes other `ChartContent` and uses its alpha. Apply it to the dimming `RectangleMark`, not to the whole `Chart` view. A SwiftUI view mask around the chart also affects axes and labels.

Build a [RectangleMark](https://developer.apple.com/documentation/charts/rectanglemark/init(xstart:xend:ystart:yend:)-6jeka) from the first timestamp to the desired cutoff, then mask it with the same samples and interpolation as the visible area and outline. Give mask marks a separate [series](https://developer.apple.com/documentation/charts/areamark/init(x:y:series:stacking:)) and use [.unstacked](https://developer.apple.com/documentation/charts/markstackingmethod/unstacked) for both areas. This keeps the duplicate mask geometry from representing an additional stacked data region. Keep mask alpha opaque; reusing the translucent visible fill would weaken the overlay. This is an implementation pattern derived from the APIs and source example, not a claim that arbitrary chart combinations have been visually verified.

Use [alignsMarkStylesWithPlotArea](https://developer.apple.com/documentation/charts/chartcontent/alignsmarkstyleswithplotarea(_:)) on both visible marks when their gradients should share plot-relative start/end points. Fix the Y domain when color positions should represent a stable numeric scale across updates. A decorative gradient is not a calibrated threshold scale; use explicit stops for actual thresholds.

Complete view and data definitions, not a standalone app. The caller supplies samples in ascending order with unique dates and finite levels in `0...10`; validate external data before using it. Empty/single-date input shows a fallback. The cutoff is clamped to the sample interval. Linear interpolation deliberately avoids implying a smoothed measurement between samples.

```swift
import SwiftUI
import Charts

struct LevelSample: Identifiable {
    let date: Date
    let level: Double
    var id: Date { date }
}

@available(iOS 16.0, macOS 13.0, *)
struct DimmingAreaChart: View {
    let samples: [LevelSample]
    let cutoff: Date

    var body: some View {
        if let first = samples.first, let last = samples.last,
           first.date < last.date {
            let end = min(max(cutoff, first.date), last.date)
            Chart {
                ForEach(samples) { sample in
                    AreaMark(
                        x: .value("Time", sample.date),
                        y: .value("Level", sample.level),
                        series: .value("Series", "Level"),
                        stacking: .unstacked
                    )
                    .interpolationMethod(.linear)
                    .foregroundStyle(.linearGradient(
                        colors: [.cyan, .blue],
                        startPoint: .bottom, endPoint: .top
                    ))
                    .alignsMarkStylesWithPlotArea()
                    .opacity(0.4)
                    .accessibilityHidden(true)

                    LineMark(
                        x: .value("Time", sample.date),
                        y: .value("Level", sample.level),
                        series: .value("Series", "Level")
                    )
                    .interpolationMethod(.linear)
                    .foregroundStyle(.linearGradient(
                        colors: [.cyan, .blue],
                        startPoint: .bottom, endPoint: .top
                    ))
                    .alignsMarkStylesWithPlotArea()
                    .lineStyle(StrokeStyle(lineWidth: 2))
                }

                RectangleMark(
                    xStart: .value("Start", first.date),
                    xEnd: .value("Cutoff", end)
                )
                .foregroundStyle(.black)
                .opacity(0.25)
                .mask {
                    ForEach(samples) { sample in
                        AreaMark(
                            x: .value("Time", sample.date),
                            y: .value("Level", sample.level),
                            series: .value("Series", "Mask"),
                            stacking: .unstacked
                        )
                        .interpolationMethod(.linear)
                        .foregroundStyle(.black)

                        LineMark(
                            x: .value("Time", sample.date),
                            y: .value("Level", sample.level),
                            series: .value("Series", "Mask")
                        )
                        .interpolationMethod(.linear)
                        .lineStyle(StrokeStyle(lineWidth: 2))
                        .foregroundStyle(.black)
                    }
                }
                .accessibilityHidden(true)

                RuleMark(x: .value("Cutoff", end))
                    .foregroundStyle(.secondary)
                    .accessibilityHidden(true)
            }
            .chartXScale(domain: first.date...last.date)
            .chartYScale(domain: 0...10, range: .plotDimension(padding: 1))
        } else {
            Text("At least two sample dates are required.")
        }
    }
}
```

## Preserve the meaning of the time and data

The source snaps its rule, point, and dimming boundary to the nearest sample. That timestamp can lie before or after the actual time. If the request is “through now,” supply the actual time as the cutoff; do not label the nearest sample's value as a current measurement without stating that approximation. The example above draws a rule at the clamped cutoff and does not invent a value there. Updating time is the caller's responsibility; accepting a `Date` does not schedule refreshes.

Place one-off dimming and rule marks outside the data loop. Draw any point emphasis after the dimming layer. If interpolation or line width changes, change both the visible and mask marks. The one-point plot-range padding above relates to the two-point stroke; it is not a universal clipping fix.

Hide duplicate area and decorative overlays from [chart accessibility](https://developer.apple.com/documentation/charts/chartcontent/accessibilityhidden(_:)), leaving the line's data exposed. If elapsed time is meaningful, expose a textual summary outside the chart as well. Hiding marks alone does not establish correct VoiceOver or Audio Graph output. Test inset axis labels, contrast, color differentiation, and material variants in the target interface.

## Evidence and validation limits

Official documentation checked 2026-10-23. The exact Swift block passed eight typecheck configurations using Xcode 27.1 (27A9275), Apple Swift 6.4, complete strict concurrency, and warnings as errors: Swift 5/6 × nonisolated/MainActor default isolation on arm64 macOS 12.0 (SDK 27.0) and arm64 iOS 15.0 (SDK 27.1), with availability annotations and no additional upcoming features. This does not verify historical compilers or SDKs.

On macOS 27.0.1 arm64, a temporary ImageRenderer harness rendered the exact view with five samples, a midpoint cutoff, a 640×320 chart, light appearance, and extra inset Y-axis labels. Compared with a zero-opacity overlay and an unmasked rectangle, the masked version changed pixels inside the left-hand area, preserved the sampled region above it, and preserved the post-cutoff region. Inspection confirmed the area/outline silhouette and visible labels in this case. Before/after-range cutoffs matched renders at the endpoints; empty and single-sample inputs rendered the fallback. Charts emitted an AxisValueLabel anchor diagnostic under the harness's extra inset-axis configuration; no general inset-axis compatibility claim follows from this check.

The source's UV dataset, health classifications, material appearance, cardinal interpolation, point selection, and screenshots are not reproduced or accepted as general guarantees. No iOS or older runtime was run. Dark appearance, other axis/annotation configurations, VoiceOver/Audio Graph, live time updates, scrolling, and performance remain untested.

## Sources

- [Area chart with a dimming layer up to the current point in time](https://nilcoalescing.com/blog/AreaChartWithADimmingLayer/) (2022-11-23)
