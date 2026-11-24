# Fixing view size with `fixedSize()`

Use this reference when a view should adopt its ideal size on one or both axes, such as letting wrapped text grow vertically without truncating or letting a flexible decoration match sibling text height. Do not use unconditional `fixedSize()` as a generic anti-truncation switch inside fixed bounds.

Apple documentation and SDK interfaces verified 2026-11-24. [`fixedSize()`](https://developer.apple.com/documentation/swiftui/view/fixedsize()) and [`fixedSize(horizontal:vertical:)`](https://developer.apple.com/documentation/swiftui/view/fixedsize(horizontal:vertical:)) are available from iOS 13 / macOS 10.15 / tvOS 13 / watchOS 6 / visionOS 1.

## Replace only the axes that need an unspecified proposal

`fixedSize(horizontal:vertical:)` intercepts the parent's [`ProposedViewSize`](https://developer.apple.com/documentation/swiftui/proposedviewsize): along each `true` axis it sends an unspecified (`nil`) proposal to its child so the child reports its ideal size, while along each `false` axis it forwards the parent's proposal unchanged. `fixedSize()` sets both axes to `true`. [fixedSize(horizontal:vertical:)](https://developer.apple.com/documentation/swiftui/view/fixedsize(horizontal:vertical:))

- **Allow multiline wrapping without vertical truncation:** use `.fixedSize(horizontal: false, vertical: true)` on `Text` when the surrounding container can accommodate the full height (such as a scrollable list or dynamically sized row). Horizontal wrapping still respects the proposed width.
- **Avoid bare `.fixedSize()` on prose:** fixing both axes asks `Text` for its single-line ideal width (preserving explicit line breaks), which ignores narrow containers and overflows horizontally instead of wrapping.
- **Check enclosing height constraints:** vertical fixing still overflows a parent with an explicit fixed height rather than shrinking the text.

## Supply an ideal size before fixing flexible views

A view's response to an unspecified proposal depends on its type:

- `Text` measures its content on a single line unless width stays proposed (`horizontal: false`).
- Non-resizable `Image` already reports its natural size for any proposal, so `fixedSize()` has no layout effect; a resizable `Image` falls back to its natural size.
- Flexible views without intrinsic content, such as `Shape` and `Color`, fall back to a default 10×10-point ideal size when fixed.

When a flexible shape needs a specific fixed size, place [`frame(minWidth:idealWidth:maxWidth:minHeight:idealHeight:maxHeight:alignment:)`](https://developer.apple.com/documentation/swiftui/view/frame(minwidth:idealwidth:maxwidth:minheight:idealheight:maxheight:alignment:)) **inside** (before) `.fixedSize()`. Putting `.frame(idealWidth:idealHeight:)` outside `.fixedSize()` leaves the inner shape collapsed at 10×10 points because the inner modifier has already answered with the shape's default ideal size.

## Match flexible sibling height at the container level

Layout containers derive their ideal size from their children. Applying `.fixedSize(horizontal: false, vertical: true)` to an `HStack` lets the stack measure its children at the proposed width, adopt the tallest child's ideal height (for example, wrapped text), and propose that concrete height to every sibling. Flexible siblings such as a leading accent bar or `Divider` then match the wrapped text height instead of expanding to fill an enclosing container's full height, without measuring frames into state via `GeometryReader`.

Complete view definition for iOS 14+ / macOS 11+ (the minimum for `accessibilityHidden`; `fixedSize(horizontal:vertical:)` itself starts at iOS 13 / macOS 10.15). The caller constrains the card's width; the surrounding layout must allow the row's height to grow with Dynamic Type and longer copy.

```swift
import SwiftUI

@available(iOS 14.0, macOS 11.0, *)
struct CalloutRow: View {
    let message: String

    var body: some View {
        HStack(spacing: 12) {
            RoundedRectangle(cornerRadius: 3)
                .fill(Color.accentColor)
                .frame(width: 6)
                .accessibilityHidden(true)

            Text(verbatim: message)
                .frame(maxWidth: .infinity, alignment: .leading)
        }
        .fixedSize(horizontal: false, vertical: true)
    }
}
```

Do not apply `.fixedSize()` (both axes) to this `HStack`: that would stop the text from wrapping and widen the entire row.

## Account for unclipped overflow and sibling starvation

A fixed-size view can report dimensions larger than its parent proposed. SwiftUI places the view according to the parent's alignment and does not clip overflow automatically; content extending past parent bounds remains visible and can overlap adjacent views. With `.center` alignment it overflows both edges symmetrically; with `.leading` alignment it overflows only past the trailing edge.

Inside an `HStack` or `VStack`, fixing a child along the stack's primary axis makes that child inflexible, so the stack allocates its full ideal extent before dividing remaining space among flexible siblings. If the fixed child's ideal size exceeds the container's available space, flexible siblings (such as a `Shape` with zero minimum width) collapse to zero while the stack overflows its frame.

Prefer `ViewThatFits`, scrolling containers, or single-axis fixing when space is bounded and content length varies (including localized strings and accessibility Dynamic Type sizes). Add `.clipped()` only when visual truncation at container bounds is the explicit design intent, never as a substitute for an adaptive layout.

## Verification and remaining boundaries

The example above was extracted verbatim and passed `swiftc -typecheck` with Xcode 27.2 (27B52), Apple Swift 6.4.2 (swiftlang-6.4.2.1.9), the iPhoneOS 27.2 SDK targeting arm64 iOS 14.0 (needed for `accessibilityHidden`), and the macOS 27.2 SDK targeting arm64 macOS 11.0, using Swift 6 with both `nonisolated` and `MainActor` default isolation plus Swift 5 with `nonisolated` default isolation, complete strict concurrency, and warnings as errors.

A separate local AppKit/SwiftUI `NSHostingView` measurement probe ran on arm64 macOS 27.2 (Xcode 27.2):
- `Capsule().fixedSize()` and `Color.red.fixedSize()` resolved to `10.0 × 10.0`; `Capsule().frame(idealWidth: 100, idealHeight: 40).fixedSize()` resolved to `100.0 × 40.0`; reversing the order to `Capsule().fixedSize().frame(idealWidth: 100, idealHeight: 40)` stayed `10.0 × 10.0`.
- For a `48 × 24` bitmap `Image` inside a `100 × 60` frame: non-resizable `Image` measured `48.0 × 24.0` with or without `.fixedSize()`; `.resizable()` expanded to `100.0 × 60.0`, and `.resizable().fixedSize()` returned to `48.0 × 24.0`.
- For a 60-character `Text` inside a `120 × 30` frame: unmodified text measured `115.5 × 16.0`, `.fixedSize()` measured `365.0 × 16.0`, and `.fixedSize(horizontal: false, vertical: true)` measured `116.0 × 64.0`.
- For an `HStack` of `RoundedRectangle` + wrapped `Text` inside `180 × 160`: without `fixedSize`, the rectangle expanded to `86.0 × 160.0` beside `86.0 × 80.0` text; with `.fixedSize(horizontal: false, vertical: true)` on the `HStack`, both the rectangle and stack matched the text's `80.0` height (`86.0 × 80.0` and `180.0 × 80.0`).
- Inside a `120 × 30` frame at `x = 140.0`, `365.0`-wide fixed text started at `x = 17.5` with `.center` alignment (symmetric overflow) and `x = 140.0` with `.leading` alignment (trailing-only overflow).
- Inside an `HStack(spacing: 8)` constrained to `120 × 40`, pairing a `RoundedRectangle` with `365.0`-wide fixed text shrank the rectangle to width `0.0` and widened the stack to `373.0`.

iOS runtime layout, Dynamic Type scaling, right-to-left mirroring, VoiceOver, and visual clipping were not exercised locally.

## Sources

- https://nilcoalescing.com/blog/TheEffectOfFixedSizeOnViewSizingAndLayoutInSwiftUI
