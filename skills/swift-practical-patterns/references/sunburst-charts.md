# Aligning rings in a sunburst chart

Use this for a small hierarchy whose children must occupy their parent's angular span. Requires Swift Charts on iOS/iPadOS 17+ or macOS 14+. For many categories or precise comparisons, prefer a bar chart or a hierarchical list; [Apple recommends few sectors and positive values](https://developer.apple.com/documentation/charts/sectormark).

## Allocate angles before creating marks

The [SectorMark initializer](https://developer.apple.com/documentation/charts/sectormark/init(angle:innerRadius:outerRadius:angularInset:)) accepts either a magnitude or an explicit range. Separate magnitude marks across multiple rings still participate in the chart's angular encoding; changing radii alone does not define parent-child alignment. Prepare one shared coordinate system, such as `0..<1`, and partition each parent's range among its children. Aggregate parent weights from the same leaves so each ring describes the same total. Keep a deterministic sibling order and stable IDs based on the full branch identity, rather than display names that can repeat.

Complete allocation helper, independent of SwiftUI. Empty input returns an empty partition; invalid weights, overflow, invalid parent bounds, or a share too small to represent return `nil`. Do not silently drop a child and then zip the shortened result against the original data. Handle failure before rendering, and omit or combine zero-valued data explicitly.

```swift
func sectorRanges(
    weights: [Double], within parent: Range<Double>
) -> [Range<Double>]? {
    guard parent.lowerBound.isFinite, parent.upperBound.isFinite,
          0 <= parent.lowerBound, parent.upperBound <= 1,
          parent.lowerBound < parent.upperBound,
          weights.allSatisfy({ $0.isFinite && $0 > 0 }) else { return nil }
    if weights.isEmpty { return [] }
    let total = weights.reduce(0, +)
    guard total.isFinite else { return nil }
    let width = parent.upperBound - parent.lowerBound
    var cumulative = 0.0
    var start = parent.lowerBound
    var result: [Range<Double>] = []
    for index in weights.indices {
        cumulative += weights[index]
        let end = index == weights.count - 1
            ? parent.upperBound
            : min(parent.upperBound, parent.lowerBound + width * (cumulative / total))
        guard start < end else { return nil }
        result.append(start..<end)
        start = end
    }
    return result
}
```

For example, partition `[3, 1]` within `0..<1`, then partition `[1, 2]` within the first returned range. Children occupy `0..<0.25` and `0.25..<0.75`; the other root occupies `0.75..<1`. Keep original weights for labels. Angular fractions are layout coordinates, not hours or measurements.

## Convert radial boundaries correctly

Apple defines `innerRadius` ratios relative to that sector's **outer radius**, while `outerRadius` ratios refer to the available plot area. For boundaries `a` and `b` measured as fractions of the full chart radius, supply `.ratio(a / b)` and `.ratio(b)`. Passing `.ratio(a)` as the inner radius multiplies it by `b` again.

Complete model and view definitions, not an app or a hierarchy builder. The caller supplies unique stable IDs, full-path labels, finite positive hours, ranges prepared above, and radial bounds satisfying `0 <= inner < outer <= 1`. The root ranges must collectively span `0..<1`; this example does not implement angular padding or domain pinning.

```swift
import SwiftUI
import Charts

struct SunburstSector: Identifiable {
    let id: String
    let label: String
    let category: String
    let hours: Double
    let angles: Range<Double>
    let inner: CGFloat
    let outer: CGFloat
}

@available(iOS 17, macOS 14, *)
struct SunburstChart: View {
    let sectors: [SunburstSector]

    var body: some View {
        Chart(sectors) { sector in
            SectorMark(
                angle: .value("Share", sector.angles),
                innerRadius: .ratio(sector.inner / sector.outer),
                outerRadius: .ratio(sector.outer)
            )
            .foregroundStyle(by: .value("Category", sector.category))
            .accessibilityLabel(sector.label)
            .accessibilityValue("\(sector.hours, format: .number) hours")
        }
    }
}
```

Angular gaps reduce visible widths, so they cannot be interpreted as exact proportions without accounting for that change. Padding every root range also removes the domain endpoints: check angular-domain inference before adopting that layout. Invisible domain marks, tiny annotation anchors, and depth-first input ordering are source techniques that still require target-runtime verification, including their effect on selection and accessibility. Do not promise VoiceOver traversal order from array order alone. Expose the hierarchy and original values in a readable list when the chart cannot communicate them clearly.

## Evidence and validation limits

Official documentation checked 2026-10-24. All Swift blocks passed Swift 6 typechecking with complete strict concurrency, nonisolated default isolation, warnings as errors, and no additional upcoming features, using Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), macOS 27.0 SDK targeting arm64 macOS 13, and iOS Simulator 27.1 SDK targeting arm64 iOS 16, preserving the view's availability annotations. These are deployment-target checks, not execution on older systems or historical toolchain validation.

On macOS 27.0.1 arm64, the allocator passed the nested example, 80 positive-weight partitions, empty input, and rejection checks for zero/negative/nonfinite weights, overflow, collapsed shares, and invalid bounds. A 400×400 ImageRenderer harness rendered the exact view with two roots and three children, using distinct fixture colors. Inspection confirmed the 3:1 root split and child boundaries; sampled pixels agreed with radial bands at fractions `0.2...0.5` and `0.55...0.8` of the 200-point available radius. This is a static local rendering check with no angular padding, annotations, or interaction. iOS/older runtimes, VoiceOver/Audio Graph, domain pinning, selection, animation, and performance remain unverified.

## Sources

- [Building a sunburst diagram in Swift Charts](https://nilcoalescing.com/blog/BuildingASunburstDiagramInSwiftCharts/) (2026-09-03)
