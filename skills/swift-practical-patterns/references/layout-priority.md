# Controlling container sizing with `layoutPriority(_:)`

Use this reference when deciding how [`layoutPriority(_:)`](https://developer.apple.com/documentation/swiftui/view/layoutpriority(_:)) changes space allocation in `HStack`/`VStack`, container sizing in `ZStack` and multi-view `overlay`/`background`, or subview handling in a custom `Layout`—and when to avoid relying on it in containers that ignore priority. For replacing bounded proposals with ideal sizes instead of prioritizing within available space, see [fixed-size.md](fixed-size.md). For `SplitArrangement` ratio and point tie-breaking on iPhone Duo, see [duo-layout-details.md](duo-layout-details.md).

Apple documentation and Xcode 27.2 SDK interfaces verified 2026-11-24. [`View.layoutPriority(_:)`](https://developer.apple.com/documentation/swiftui/view/layoutpriority(_:)) is available from iOS 13 / macOS 10.15 / tvOS 13 / watchOS 6 / visionOS 1; [`LayoutSubview.priority`](https://developer.apple.com/documentation/swiftui/layoutsubview/priority) requires iOS 16 / macOS 13 / tvOS 16 / watchOS 9 / visionOS 1.

## Match the container's priority policy

`layoutPriority(_:)` stores a `Double` trait on a view (default `0`; negative values are valid). SwiftUI does not enforce one universal behavior—each container applies its own policy:

| Container | How it uses `layoutPriority(_:)` |
|---|---|
| `HStack` / `VStack` | Allocates primary-axis space from highest to lowest priority group after reserving spacing and lower-priority minimums |
| `ZStack` | Computes the stack's own size from the highest-priority subview(s), then proposes that size to lower-priority subviews |
| Multi-view `overlay` / `background` | Applies the same implicit `ZStack` sizing rule among overlay/background children; the modified base view keeps its own size |
| Custom `Layout` | No automatic behavior; the layout must read [`LayoutSubview.priority`](https://developer.apple.com/documentation/swiftui/layoutsubview/priority) and implement its own rule |
| `LazyHStack` / `LazyVStack` | No visible effect along the stack's primary axis (children receive an unspecified proposal and adopt their ideal size) |
| `List` / `Form` | No cross-row allocation effect (rows are sized independently) |
| [`ViewThatFits`](https://developer.apple.com/documentation/swiftui/viewthatfits) | Ignored; evaluates candidate views strictly in declaration order |

Only the relative order of priority values matters in built-in stacks (`1` and `100` produce the same layout). Use negative values when one view should take space only after default-priority (`0`) siblings.

## Allocate bounded space in `HStack` and `VStack`

Along their primary axis, `HStack` and `VStack` process subviews in descending priority groups. For each group, the stack subtracts inter-item spacing and reserves the minimum space needed by all lower-priority subviews, then offers the remaining space to the current group. Any unused space passes down to the next lower priority group. [layoutPriority(_:)](https://developer.apple.com/documentation/swiftui/view/layoutpriority(_:))

- **Flexible vs. bounded prioritized views:** a prioritized flexible view (`Shape`, `Color`, `Spacer`, or `.frame(maxWidth: .infinity)`) accepts all offered space, clamping lower-priority flexible siblings to their minimum size (or `0` if unconstrained). A bounded view such as short `Text` stops at its preferred size and passes the leftover space to lower-priority siblings.
- **Prefer `layoutPriority(_:)` over `fixedSize()` inside bounded rows:** raising a `Text` label's priority lets it resist truncation ahead of lower-priority siblings while staying inside the stack's proposed width and preserving sibling minimums. By contrast, `.fixedSize()` replaces the parent proposal with `nil` and can overflow the container or collapse siblings to zero; see [fixed-size.md](fixed-size.md).

## Size `ZStack` and multi-view `overlay` / `background` from one layer

`ZStack` does not partition space among siblings. When all children share the same priority, every child contributes to the stack's size—so a flexible `Shape` or `Color` expands the whole `ZStack` to fill its parent. Raising one child's priority makes only the highest-priority child(ren) determine the `ZStack`'s size, which is then proposed to lower-priority children.

The same rule governs the implicit `ZStack` formed when multiple views are passed to [`overlay(alignment:content:)`](https://developer.apple.com/documentation/swiftui/view/overlay(alignment:content:)) or [`background(alignment:content:)`](https://developer.apple.com/documentation/swiftui/view/background(alignment:content:)): raising the priority of a badge label shrinks a flexible backing shape inside the overlay to hug that label, while the modified base view retains its own size.

Complete view definition for iOS 15+ / macOS 12+ (required for `overlay(alignment:content:)` and `.tint`/`.quaternary`; `layoutPriority(_:)` itself starts at iOS 13 / macOS 10.15). When a single view simply needs a background shape behind it, attach `.background(...)` directly to that view instead of wrapping both in a `ZStack` just to set `layoutPriority`.

```swift
import SwiftUI

@available(iOS 15.0, macOS 12.0, *)
struct PrioritizedBadgeOverlay: View {
    let badgeText: String

    var body: some View {
        RoundedRectangle(cornerRadius: 16)
            .fill(.quaternary)
            .frame(width: 240, height: 120)
            .overlay {
                Capsule()
                    .fill(.tint)
                Text(verbatim: badgeText)
                    .padding(.horizontal, 12)
                    .padding(.vertical, 6)
                    .layoutPriority(1)
            }
    }
}
```

## Implement priority explicitly in custom `Layout` types

SwiftUI forwards `layoutPriority(_:)` to custom [`Layout`](https://developer.apple.com/documentation/swiftui/layout) implementations through `subview.priority`, without applying any default sizing or ordering policy. If your custom layout should respect priority, read `subview.priority` in `sizeThatFits` and/or `placeSubviews`, guard against empty `subviews`, and preserve declaration order on ties (Swift's `sorted(by:)` is guaranteed stable). [LayoutSubview.priority](https://developer.apple.com/documentation/swiftui/layoutsubview/priority)

Complete `Layout` definition for iOS 16+ / macOS 13+. This layout divides available width evenly and orders subviews from highest to lowest priority while keeping declaration order for equal priorities.

```swift
import SwiftUI

@available(iOS 16.0, macOS 13.0, tvOS 16.0, watchOS 9.0, *)
struct PriorityOrderedRow: Layout {
    var spacing: CGFloat = 8

    func sizeThatFits(
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout ()
    ) -> CGSize {
        proposal.replacingUnspecifiedDimensions()
    }

    func placeSubviews(
        in bounds: CGRect,
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout ()
    ) {
        guard !subviews.isEmpty else { return }
        let count = CGFloat(subviews.count)
        let width = max(0, (bounds.width - spacing * (count - 1)) / count)
        var x = bounds.minX

        for subview in subviews.sorted(by: { $0.priority > $1.priority }) {
            subview.place(
                at: CGPoint(x: x, y: bounds.minY),
                proposal: ProposedViewSize(width: width, height: bounds.height)
            )
            x += width + spacing
        }
    }
}
```

## Verification and remaining boundaries

Both examples above were extracted verbatim and passed `swiftc -typecheck` with Xcode 27.2 (27B52), Apple Swift 6.4.2 (swiftlang-6.4.2.1.9), targeting arm64 iOS 16.0 (`iPhoneOS27.2.sdk`) and arm64 macOS 13.0 (`MacOSX27.2.sdk`) under Swift 6 (`nonisolated` and `MainActor` default isolation) and Swift 5 (`nonisolated` default isolation), with complete strict concurrency and warnings as errors (`PrioritizedBadgeOverlay` also passed at arm64 iOS 15.0 and macOS 12.0).

A separate local AppKit/SwiftUI `NSHostingView` measurement probe ran on arm64 macOS 27.2 (Xcode 27.2):

- `HStack(spacing: 8)` in `340 × 120` with three `minWidth: 60` flexible views: equal priority gave `108.0` each; raising the middle child to `layoutPriority(1)` or `layoutPriority(100)` gave identical results (`60.0`, `204.0`, `60.0`); lowering the third child to `layoutPriority(-1)` gave `132.0`, `132.0`, `60.0`.
- `HStack(spacing: 8)` in `340 × 120` with prioritized short `Text("Short")` (`layoutPriority(1)`) and a flexible sibling: text stayed `34.0 × 16.0` and passed the remaining `298.0` width to the sibling.
- `HStack(spacing: 10)` in `300 × 80` with three tiers (`priority(2)` fixed `100`, `priority(1)` flexible, `priority(0)` `minWidth: 40`) resolved to `100.0`, `140.0`, `40.0`.
- `HStack(spacing: 8)` in `120 × 40` with long single-line `Text` (`layoutPriority(1)`) and a `minWidth: 30` sibling clamped text to `79.0` without overflowing `120.0`, leaving `33.0` for the sibling.
- `ZStack` in `340 × 120` with a flexible rectangle and padded `Text("Hello, world!")`: equal priority sized the `ZStack` and rectangle to `340.0 × 120.0`; adding `.layoutPriority(1)` to the text shrank both the `ZStack` and rectangle to `100.0 × 40.0`.
- Multi-view `.overlay` on a `340 × 120` base view with a flexible rectangle and padded `Text("Badge")`: equal priority stretched the overlay rectangle to `340.0 × 120.0`; adding `.layoutPriority(1)` to the overlay text shrank the overlay rectangle to `55.0 × 32.0` while the base view stayed `340.0 × 120.0`.
- `PriorityOrderedRow(spacing: 8)` in `340 × 120` with the third child at `layoutPriority(1)` placed the third child at `x = 0.0`, followed by the first at `x = 116.0` and the second at `x = 232.0` (`108.0` wide each).
- `ViewThatFits` in `200 × 40` chose the first fitting child (`80.0 × 20.0`, priority `0`) over a second fitting child (`120.0 × 20.0`) with `layoutPriority(10)`.
- `LazyHStack(spacing: 8)` with three `minWidth: 60` flexible views (middle `layoutPriority(1)`) resolved all three to `60.0` wide both inside a horizontal `ScrollView` and directly inside a `300 × 60` frame.

iOS runtime execution, right-to-left layout, Dynamic Type scaling, `List`/`Form` runtime checks, and `Grid` multi-row priority edge cases were not exercised locally.

## Sources

- [The role of layout priority in SwiftUI containers](https://nilcoalescing.com/blog/TheRoleOfLayoutPriorityInSwiftUIContainers)
