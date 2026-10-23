# Alignment guides across nested views

Use this reference when a row must align an icon or sibling with a particular text line inside a nested stack. Alignment guides describe a relationship between content points; measuring frames into state or adding a visual offset does not express that relationship.

Apple documentation verified 2026-10-23. The alignment APIs used here require iOS 13 / macOS 10.15. The complete example uses iOS 14 / macOS 11 for `accessibilityHidden`; it was checked with Swift 6.4; validation details appear below.

## Select the guide before overriding it

An `HStack` selects a vertical guide, a `VStack` selects a horizontal guide, and a `ZStack` selects both. Set the container's alignment to the guide you intend to override. A child override for an inactive guide does not reposition that child in the stack. The returned value is a coordinate in the modified view's own dimensions, not a translation to apply to its frame. For example, moving a leading guide outward can move the view inward to keep the guides coincident. [Stack alignment](https://developer.apple.com/documentation/swiftui/aligning-views-within-a-stack), [ViewDimensions](https://developer.apple.com/documentation/swiftui/viewdimensions)

Use `.firstTextBaseline` or `.lastTextBaseline` for text of different sizes. A view without text uses its bottom for baseline alignment. Custom artwork can override that coordinate using its actual bounds; a fraction tuned for one asset is not a universal icon correction. SF Symbols already have typographic guides, so preserve those unless the design demonstrates a specific mismatch. [Baseline and symbol guidance](https://developer.apple.com/documentation/swiftui/aligning-views-within-a-stack)

## Give the nested target its own guide

Choose a custom `AlignmentID` when the desired point is inside a nested stack and the wrapper's built-in baseline identifies a different line. Define a fallback for views without an explicit value, select the custom guide on the common ancestor, and attach the explicit value to the one descendant that identifies the target. SwiftUI projects that value through the enclosing stack. Do not repeat the same override on the wrapper: that can replace the point you intended to propagate. [Cross-stack alignment](https://developer.apple.com/documentation/swiftui/aligning-views-across-stacks), [AlignmentID](https://developer.apple.com/documentation/swiftui/alignmentid)

This complete view definition aligns the time's baseline and marker's bottom with the nested title's baseline, rather than the category line or the whole column's center. The marker is decorative; the text carries the event information.

```swift
import SwiftUI

private struct EventTitleID: AlignmentID {
    nonisolated static func defaultValue(in dimensions: ViewDimensions) -> CGFloat {
        dimensions[VerticalAlignment.firstTextBaseline]
    }
}

extension VerticalAlignment {
    nonisolated static let eventTitle = VerticalAlignment(EventTitleID.self)
}

@available(iOS 14.0, macOS 11.0, *)
struct EventRow: View {
    let time: String
    let category: String
    let title: String
    let detail: String

    var body: some View {
        HStack(alignment: .eventTitle) {
            Text(verbatim: time)
            Circle()
                .fill(.blue)
                .frame(width: 10, height: 10)
                .accessibilityHidden(true)
            VStack(alignment: .leading) {
                Text(verbatim: category).font(.caption)
                Text(verbatim: title)
                    .font(.headline)
                    .alignmentGuide(.eventTitle) { dimensions in
                        dimensions[VerticalAlignment.firstTextBaseline]
                    }
                Text(verbatim: detail)
            }
        }
    }
}
```

The fallback uses the time's first baseline and the textless marker's bottom. The column supplies the explicit title guide. If several descendants provide explicit values for the same guide, they are combined; a custom guide does not select a unique descendant by name. For a single target, supply one explicit value per participating subtree. [Combining explicit values](https://developer.apple.com/documentation/swiftui/verticalalignment/combineexplicit(_:))

For horizontal guides, use leading/trailing semantics. SwiftUI mirrors horizontal guide coordinates in a right-to-left layout; do not add an unconditional manual sign flip. Check multiline text, larger Dynamic Type, and supported layout directions in the actual hierarchy. [Layout direction](https://developer.apple.com/documentation/swiftui/viewdimensions)

## Verification and remaining boundaries

The exact code block passed `swiftc -typecheck` with Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), the iPhoneOS 27.1 SDK targeting arm64 iOS 14, and the macOS 27.0 SDK targeting arm64 macOS 11. Checks used complete strict concurrency and warnings as errors: Swift 6 with both `nonisolated` and `MainActor` default isolation, and Swift 5 with `nonisolated` default isolation.

A separate Swift 6 ImageRenderer/custom-Layout probe ran on arm64 macOS 27.0.1: a nested explicit coordinate projected to 15 points in a 60-point column; a textless column without an explicit value fell back to its 30-point bottom; two explicit values (10 and 30) merged to 20. These are local layout/value checks, not a measurement of the sample row's text baselines. No visible row, Dynamic Type, right-to-left layout, accessibility, iOS execution, or older runtime was validated.

## Sources

- https://nilcoalescing.com/blog/AlignmentGuidesInSwiftUI/
