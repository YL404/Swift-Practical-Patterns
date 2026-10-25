# Navigation-bar minimization and content stability

Use this reference when a scrolling screen needs a minimizing navigation bar, content shifts as that bar changes size, or the bar should return only at the scroll edge. Item visibility priorities and overflow placement are separate decisions.

Apple documentation and SDK declarations checked 2026-10-25. The example requires an SDK declaring the iOS 27 APIs and iOS/iPadOS 27 at runtime. Although the modifiers are declared on other platforms, the explicit `.onScrollDown`, `.disabled`, and `.atScrollEdge` values used here are iOS-only (including iPadOS and Mac Catalyst), not macOS/visionOS equivalents. Keep an ordinary navigation bar on earlier systems; a runtime check does not supply missing SDK declarations.

## Choose the scroll, safe-area, and restoration policies separately

- Apply `toolbarMinimizationBehavior(.onScrollDown, for: .navigationBar)` within a system navigation container. The documented supported placement is `.navigationBar`; an integrated top tab bar minimizes with it. [Minimization contract](https://developer.apple.com/documentation/swiftui/view/toolbarminimizationbehavior(_:for:))
- Keep automatic safe-area adjustment when content should reflow into the space vacated by the bar. Choose `toolbarMinimizationSafeAreaAdjustment(.disabled, for: .navigationBar)` when content should keep its safe area, such as media beneath the bar. This controls the adjustment caused by minimization; it does not disable all safe areas or fix every source of layout movement. Only `.navigationBar` currently supports this policy. [Safe-area contract](https://developer.apple.com/documentation/swiftui/view/toolbarminimizationsafeareaadjustment(_:for:))
- The default restoration responds to reversing scroll direction. Choose `toolbarMinimizationRestoration(.atScrollEdge, for: .navigationBar)` to wait for the content to reach the scroll edge. **This customization is currently honored only for `.navigationBar` together with `.onScrollDown`.** Do not promise the same result with `.onScrollUp` or another bar. [Restoration contract](https://developer.apple.com/documentation/swiftui/view/toolbarminimizationrestoration(_:for:)), [atScrollEdge restrictions](https://developer.apple.com/documentation/swiftui/toolbarminimizationrestoration/atscrolledge)

This complete view definition demonstrates a stable safe area and edge-only restoration. Those are explicit design choices, not defaults for every reading screen. The constant range is fixed demonstration content, not indices into mutable data. Callers supporting earlier systems must select this type with `#available(iOS 27, *)` and retain their ordinary navigation/scroll content in the fallback.

```swift
import SwiftUI

@available(iOS 27.0, *)
struct MinimizingReader: View {
    var body: some View {
        NavigationStack {
            ScrollView {
                LazyVStack(alignment: .leading, spacing: 24) {
                    ForEach(1...50, id: \.self) { number in
                        Text("Section \(number)")
                            .frame(maxWidth: .infinity, alignment: .leading)
                    }
                }
                .padding()
            }
            .navigationTitle("Reading")
            .toolbarMinimizationBehavior(.onScrollDown, for: .navigationBar)
            .toolbarMinimizationSafeAreaAdjustment(.disabled, for: .navigationBar)
            .toolbarMinimizationRestoration(.atScrollEdge, for: .navigationBar)
        }
    }
}
```

## Verification and remaining boundaries

The exact code block passed `swiftc -typecheck` with Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), and the iPhoneOS 27.1 SDK targeting arm64 iOS 26, retaining the type's iOS 27 availability annotation. Checks used Swift 6, complete strict concurrency, warnings as errors, and both `nonisolated` and `MainActor` default isolation. Removing the annotation at iOS 26 was rejected; a separate macOS 27 check rejected `.atScrollEdge` as unavailable.

No navigation container was launched. Minimization, safe-area movement, scroll-edge restoration, integrated tab bars, vertical-bar contexts, resize, search, accessibility, and earlier-system fallback execution were not tested. The behavior guidance follows the linked Apple contracts, not a reproduced scrolling session.

## Sources

- [Adaptive SwiftUI toolbars in iOS 27](https://nilcoalescing.com/blog/AdaptiveSwiftUIToolbarsInIOS27/)
