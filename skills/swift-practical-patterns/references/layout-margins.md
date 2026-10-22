# System margins and content alignment

Use this guidance when custom SwiftUI content should align with system container margins, a carousel should scroll to the edge while resting at those margins, or UIKit content needs margin-guide constraints. Intentional design spacing is not a bug merely because it differs from system margins.

Official documentation verified 2026-10-22. The SwiftUI guide APIs below require the Xcode 27.1 SDK and iOS/iPadOS 27.1 at runtime; Apple also lists aligned 27.1 releases on other platforms. Examples target iOS. A runtime availability check does not make these declarations available to an older SDK.

## Choose the source and destination of spacing

| Need | Use | Minimum iOS |
|---|---|---|
| Align content with the margins of its current container | `contentMargins(for: .container, edges: .horizontal)` | 27.1 |
| Read the current container's margins for drawing or internal scroll insets | `GeometryProxy.contentMargins(for:edges:)` | 27.1 |
| Set chosen margins on scroll content or indicators | `contentMargins(_:_:for:)` with `ContentMarginPlacement` | 17 |
| Constrain UIKit content to its view's margins | `UIView.layoutMarginsGuide` | 9 |

`ContentMarginGuide.container` is a **source** of system metrics, not a `ContentMarginPlacement`. Its values depend on the platform, container size, and the view's position inside the container. On iPhone Duo the two sides may differ next to a vertical bar. Do not substitute a constant, mirror one edge to the other, or choose margins by device name. [Container guide](https://developer.apple.com/documentation/swiftui/contentmarginguide/container), [Scroll content margins](https://developer.apple.com/documentation/swiftui/view/contentmargins(_:_:for:))

Margins do not replace safe areas or fold avoidance. Keep readable and interactive content within the system's safe-area layout; an intentional full-bleed background can extend beyond it. A system margin guide is not a query for a fold's reserved region. For the narrower Duo sizing and presentation decisions, see [duo-layout-details.md](duo-layout-details.md).

## Apply the guide rather than guessing its value

`contentMargins(for:edges:alignment:)` insets content at placement time while leaving the proposed size unchanged. Edges omitted from `edges` stay unchanged. `alignment` positions a view that does not fill its proposed size inside the inset bounds; it usually has no apparent effect on a view that already fills them. [Modifier contract](https://developer.apple.com/documentation/swiftui/view/contentmargins(for:edges:alignment:))

This is a complete view definition. A containing app supplies the text and presents it. The older-system branch retains readable content with ordinary padding; it does **not** reproduce system margin alignment exactly.

```swift
import SwiftUI

@available(iOS 16.0, *)
struct MarginAlignedText: View {
    let text: String

    var body: some View {
        NavigationStack {
            ScrollView {
                if #available(iOS 27.1, *) {
                    paragraph
                        .contentMargins(for: .container, edges: .horizontal)
                } else {
                    paragraph
                        .padding(.horizontal)
                }
            }
            .navigationTitle("Reading")
        }
    }

    private var paragraph: some View {
        Text(verbatim: text)
            .frame(maxWidth: .infinity, alignment: .leading)
    }
}
```

Apply the guide to the content whose edges should align, not indiscriminately to a full-bleed visual layer or to every ancestor and descendant. Read the guide near its consumer when numeric insets are required: Apple's contract makes the value position-dependent. Do not cache a root measurement as a universal margin for nested views.

## Keep a carousel's viewport wide and inset its content

Applying the guide outside a scroll view narrows the view itself. To keep the viewport edge-to-edge and inset only its resting content, read the guide at that scroll view and pass its `EdgeInsets` to the older `contentMargins(_:_:for:)` API with `.scrollContent`. Omitted edges read as zero. Scroll content margins add to the safe-area inset; do not add the same safe-area values again. `.scrollContent` does not change the scroll-indicator insets. [GeometryProxy reader](https://developer.apple.com/documentation/swiftui/geometryproxy/contentmargins(for:edges:)), [Placement and safe-area composition](https://developer.apple.com/documentation/swiftui/view/contentmargins(_:_:for:))

This is a complete iOS 27.1 view definition; select it behind `#available(iOS 27.1, *)` in an older-target app and retain that app's existing carousel there. Titles are unique identifiers for this small example, not a recommendation to identify mutable production records by display text.

```swift
import SwiftUI

@available(iOS 27.1, *)
struct MarginAlignedCarousel: View {
    private let titles = ["Lily", "Milo", "Leo", "Luna", "Cleo"]

    var body: some View {
        GeometryReader { proxy in
            ScrollView(.horizontal) {
                HStack(spacing: 12) {
                    ForEach(titles, id: \.self) { title in
                        Text(verbatim: title)
                            .frame(width: 140, height: 100)
                            .background(.orange.opacity(0.2), in: .rect(cornerRadius: 12))
                    }
                }
            }
            .contentMargins(
                .horizontal,
                proxy.contentMargins(for: .container, edges: .horizontal),
                for: .scrollContent
            )
        }
        .frame(height: 100)
    }
}
```

The reader is justified here because the consumer needs numeric insets, not because the layout needs to measure and reapply its safe area. Preserve the two directional values independently when adapting this pattern for custom drawing or a different scrolling container. Multiple scroll-margin modifiers for the same placement override higher ancestors; inspect existing modifiers before adding another one. [Override behavior](https://developer.apple.com/documentation/swiftui/view/contentmargins(_:_:for:))

## UIKit: constrain to the guide, not copied SwiftUI numbers

UIKit's margin guide and SwiftUI's container guide serve a similar alignment purpose, but their numeric values are not interchangeable. A view controller's root `layoutMargins` reflects system minimum margins and safe-area insets. Prefer `directionalLayoutMargins` when setting UIKit margins on iOS 11+, and use the guide's leading/trailing anchors for constraints. [UIKit margins](https://developer.apple.com/documentation/uikit/uiview/layoutmargins)

This is a complete UIKit view-controller definition, not a bridge required for the SwiftUI examples. It leaves safe-area adjustment and system minimum margins enabled.

```swift
import UIKit

final class MarginAlignedViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        let label = UILabel()
        label.text = "Aligned with the container margins."
        label.numberOfLines = 0
        label.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(label)

        let margins = view.layoutMarginsGuide
        NSLayoutConstraint.activate([
            label.leadingAnchor.constraint(equalTo: margins.leadingAnchor),
            label.trailingAnchor.constraint(equalTo: margins.trailingAnchor),
            label.topAnchor.constraint(equalTo: margins.topAnchor)
        ])
    }
}
```

Keep these separate UIKit controls explicit:

- `insetsLayoutMarginsFromSafeArea` defaults to `true`. Setting it to `false` permits margins outside the safe area; it is not a fix for unexpected extra spacing. [Safe-area adjustment](https://developer.apple.com/documentation/uikit/uiview/insetslayoutmarginsfromsafearea)
- `viewRespectsSystemMinimumLayoutMargins` defaults to `true`. Setting a small `directionalLayoutMargins` value on the root view may still be constrained by the system minimum. Disable the minimum only for an intentional custom design, not just to match one screenshot. [Root-view minimum](https://developer.apple.com/documentation/uikit/uiviewcontroller/viewrespectssystemminimumlayoutmargins)
- `preservesSuperviewLayoutMargins` defaults to `false`. A nested view has its own margins; opt in when it should also respect its superview's margins. This preserves their boundary, not an unconditional copy of their numeric insets. [Superview margins](https://developer.apple.com/documentation/uikit/uiview/preservessuperviewlayoutmargins)

## Check the layout, not a device-specific number

Inspect navigation-title alignment, both ends of horizontal scrolling, rotation, a right-to-left language, and nested padding. On Duo include outer/inner displays and the side next to a vertical bar. Keep the viewport, safe area, and content margins distinguishable when diagnosing an offset. Exercise larger text sizes separately; fixed-height sample cards are not a production Dynamic Type strategy.

The source article's exact point measurements and nested-reader values are simulator observations under its reported iOS 27.2 conditions (Duo: iOS 27.1), not constants or universal API guarantees. A zero margin must not be interpreted as a missing safe area, and an older-system padding fallback must not be described as identical alignment.

The API decisions above use the linked Apple contracts; numeric comparisons from the articles are not adopted as layout rules.

## Verification and remaining boundaries

The three code blocks were extracted verbatim and passed `swiftc -typecheck -swift-version 6 -strict-concurrency=complete` with Xcode 27.1 (27A9275), Apple Swift 6.4, the iPhoneOS 27.1 SDK, and an arm64 iOS 16.0 deployment target. New declarations remain availability-gated; the UIKit example does not require a SwiftUI bridge in production.

A temporary native host then displayed the paragraph, initial carousel layout, and UIKit label on the iOS 27.1 iPhone Duo simulator's outer display in portrait. Its linked simulator SDK was verified as 27.1. Screenshots showed aligned readable content and the system's side status region. A separate geometry probe in that setup read leading/trailing margins of 20/0 at its root, 12/0 after 8 points of horizontal padding, and 0/0 after applying the guide; these are observations of those positions, not layout constants.

Carousel scrolling to the final item, rotation, inner-display and partially folded layouts, right-to-left layout, accessibility settings, physical devices, other Apple platforms, and execution of the older-system branch were not verified. The checks above do not establish those behaviors.

## Sources

- [How to use layoutMarginsGuide in SwiftUI](https://sarunw.com/posts/swiftui-content-margin-guide/)
- [How to use layoutMarginsGuide in UIKit](https://sarunw.com/posts/uikit-layoutmarginsguide/)
