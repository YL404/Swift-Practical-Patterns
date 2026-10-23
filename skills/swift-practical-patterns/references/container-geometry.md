# Container sizing and geometry breakpoints

Use this reference when a card's size should follow a supported container, or a custom layout rule depends on resolved width and height rather than a size class. For system margin-guide alignment, see [layout-margins.md](layout-margins.md).

Official documentation and SDK declarations verified 2026-10-23. `containerRelativeFrame` requires iOS 17 / macOS 14; the single-new-value `onGeometryChange` overload below supports iOS 16 / macOS 13 with an SDK that declares it. A low deployment target does not make an older SDK recognize an API.

## Decide whether to size, select, or observe

Express a size relationship directly with `containerRelativeFrame`; do not round-trip the measured size through state just to set a frame. Use `ViewThatFits` when the requirement is to pick the first preferred alternative whose ideal size fits the selected axes. Use a custom `Layout` when children must be measured and placed together. Observe geometry when the interface needs a separate derived value or a geometric breakpoint that size classes cannot express. [Container sizing](https://developer.apple.com/documentation/swiftui/view/containerrelativeframe(_:alignment:_:)), [fit selection](https://developer.apple.com/documentation/swiftui/viewthatfits), [geometry observation](https://developer.apple.com/documentation/swiftui/view/ongeometrychange(for:of:action:))

## Size relative to the supported container

The container is the nearest supported ancestor, such as a scroll view, navigation stack, split-view column, tab, or presenting window. An ordinary `HStack` or `VStack` does not establish that reference size. The provided container length accounts for its safe-area insets; it is not the immediate parent's width or a process-global screen width. Only the requested axes receive this relative frame. [Container contract](https://developer.apple.com/documentation/swiftui/view/containerrelativeframe(_:alignment:_:))

Choose the overload by the relationship: the axes-only form matches a container dimension; `count` and `span` divide it into portions; the length closure computes a custom dimension from the provided length and axis. For example, a closure returning `length * 0.6` on `.vertical` sets a proportional height without prescribing the width.

For a count-and-span frame, `count` is the number of equal portions in the container, not the number of records. `span` is how many portions one item occupies. With the length supplied to the modifier L, portion count n, span k, and gap s, the item length is `(L - s * (n - 1)) / n * k + s * (k - 1)`. The spacing argument participates in this calculation; it does not insert gaps between siblings. Pass the same gap to the arranging stack. Use positive counts and spans that describe the intended layout, and check narrow proposals where the gaps could consume the available length. [Formula and overload](https://developer.apple.com/documentation/swiftui/view/containerrelativeframe(_:count:span:spacing:alignment:))

This complete view definition sets two-of-three card widths without prescribing a card height. The scroll view still needs a suitable height proposal from its parent. The constant range represents five fixed demo cards, not indices into a mutable model collection.

```swift
import SwiftUI

@available(iOS 17.0, macOS 14.0, *)
struct RelativeCards: View {
    private let spacing: CGFloat = 12

    var body: some View {
        ScrollView(.horizontal) {
            LazyHStack(spacing: spacing) {
                ForEach(0..<5) { day in
                    Text("Day \(day + 1)")
                        .padding()
                        .frame(maxWidth: .infinity)
                        .background(.blue.opacity(0.15), in: .rect(cornerRadius: 12))
                        .containerRelativeFrame(
                            .horizontal, count: 3, span: 2, spacing: spacing
                        )
                }
            }
        }
        .contentMargins(.horizontal, 20, for: .scrollContent)
    }
}
```

Scroll content margins inset the resting content separately from the viewport and scroll indicators. Keep them distinct from stack spacing and ordinary padding on the scroll view. Apple's scroll-view example combines content margins with container-relative card sizing. Do not manually subtract safe-area insets again. For system-provided margins rather than a chosen 20-point design inset, use the linked margin reference. [Content-margin composition](https://developer.apple.com/documentation/swiftui/view/contentmargins(_:_:for:)), [Beyond scroll views](https://developer.apple.com/videos/play/wwdc2023/10159/)

Sizing alone does not configure snapping or paging. The two-of-three example is not a full-width page; choose scroll-target behavior separately when the interface requires it.

## Derive a decision from a stable viewport

Return the smallest value needed by the response, such as a `Bool`, rather than storing every `CGSize`. SwiftUI may reevaluate the transform as geometry changes; the action responds to changes in the transformed value. This filters action calls, not geometry measurements or all layout work. In the verified SDK the transform is `@Sendable` and its result must be `Equatable & Sendable`; capture immutable inputs instead of reading mutable actor-isolated state inside the transform. [Transform and action contract](https://developer.apple.com/documentation/swiftui/view/ongeometrychange(for:of:action:))

This complete view definition uses a caller-chosen width threshold plus the viewport's aspect relationship. The threshold is a design input, not a device constant. Present it inside a parent that provides a finite viewport. `AnyLayout` keeps the same child hierarchy while changing its arrangement. [AnyLayout](https://developer.apple.com/documentation/swiftui/anylayout)

```swift
import SwiftUI

@available(iOS 16.0, macOS 13.0, *)
struct GeometryAdaptiveCard: View {
    let title: String
    let detail: String
    let widthThreshold: CGFloat
    @State private var usesVerticalLayout = true

    var body: some View {
        ScrollView {
            let layout = usesVerticalLayout
                ? AnyLayout(VStackLayout(alignment: .leading))
                : AnyLayout(HStackLayout(alignment: .top))
            layout {
                Text(verbatim: title).font(.headline)
                Text(verbatim: detail)
            }
            .padding()
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .onGeometryChange(for: Bool.self) { [widthThreshold] geometry in
            let size = geometry.size
            return size.height > size.width && size.width < widthThreshold
        } action: { newValue in
            usesVerticalLayout = newValue
        }
    }
}
```

Measure the externally constrained viewport, not the card whose own size changes when the action switches its layout. This is an implementation strategy for avoiding a measurement-state-layout feedback loop, not a promise that every enclosing hierarchy is independent of its content. A Boolean still permits oscillation if the state change moves the measured geometry back across the breakpoint. Check the actual hierarchy; adding an equality guard after the transform does not repair that dependency. The initial `true` is a provisional arrangement until geometry is delivered.

Use size classes when their broad categories already express the decision. Use geometric breakpoints for requirements that genuinely depend on dimensions or their relationship; do not infer device orientation from this Boolean. Content fit and Dynamic Type can require a different rule. For scroll offsets or scroll-content metrics, prefer `onScrollGeometryChange` when its deployment requirements are met. The old-and-new-value `onGeometryChange` overload requires iOS 18 / macOS 15 in the verified SDK; it does not share the single-value overload's older deployment floor. [Two-value overload](https://developer.apple.com/documentation/swiftui/view/ongeometrychange(for:of:action:)-36gt0)

## Verification and remaining boundaries

The two code blocks were extracted verbatim and passed `swiftc -typecheck` with Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), the iPhoneOS 27.1 SDK targeting arm64 iOS 16.0, and the macOS 27.0 SDK targeting arm64 macOS 13.0. Checks used complete strict concurrency and warnings as errors: Swift 6 with both `nonisolated` and `MainActor` default isolation, plus Swift 5 mode with `nonisolated` default isolation. The relative-sizing view retains its own iOS 17 / macOS 14 availability annotation.

Separate Swift 6 checks rejected the unguarded old-and-new-value geometry action at iOS 17 / macOS 14 and accepted it at iOS 18 / macOS 15. These are compiler availability checks with the current SDK, not execution on those older systems.

No visible layout, scrolling, resizing, older-system execution, Dynamic Type, or accessibility behavior has been validated for these examples. Content-margin/card-width composition, initial arrangement delivery, feedback-loop convergence, and layout performance still require checks in the target app; compilation does not establish those behaviors.

## Sources

- [Building adaptive SwiftUI layouts with containerRelativeFrame()](https://nilcoalescing.com/blog/BuildingAdaptiveSwiftUILayoutsWithContainerRelativeFrame/)
- [Responding to geometry changes in SwiftUI](https://nilcoalescing.com/blog/RespondingToGeometryChangesInSwiftUI/)
