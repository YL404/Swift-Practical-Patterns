# Scroll-driven effects and visibility callbacks

Use this reference when choosing how to drive visual effects, visibility-triggered behavior, or external UI updates from a `ScrollView`. For default scroll anchors, `ScrollPosition`, and programmatic scrolling, see [scroll-position.md](scroll-position.md). For container-relative card sizing and non-scroll geometry breakpoints, see [container-geometry.md](container-geometry.md).

Apple documentation and Xcode 27.2 SDK interfaces verified 2026-11-24. `scrollTransition(_:axis:transition:)` and `visualEffect(_:)` require iOS 17 / macOS 14 / tvOS 17 / watchOS 10 / visionOS 1. `onScrollVisibilityChange(threshold:_:)`, `onScrollTargetVisibilityChange(idType:threshold:_:)`, and `onScrollGeometryChange(for:of:action:)` require iOS 18 / macOS 15 / tvOS 18 / watchOS 11 / visionOS 2.

## Choose the narrowest scroll API

Start with the most constrained API that can express the effect. Stateless visual modifiers avoid body re-evaluation during every scroll frame, while state-updating modifiers trade extra invalidation for the ability to coordinate other views or side effects.

| Requirement | Preferred API | Minimum iOS / macOS | Writes state / affects layout |
|---|---|---|---|
| Fade, scale, or rotate a row as it enters or leaves the visible region | `scrollTransition(_:axis:transition:)` | 17 / 14 | No / No |
| Continuous per-item effect tied to exact position in the scroll viewport (e.g., parallax) | `visualEffect(_:)` with `.scrollView` geometry | 17 / 14 | No / No |
| Start/stop animations, playback, or work when one view crosses a visibility fraction | `onScrollVisibilityChange(threshold:_:)` | 18 / 15 | Yes (on crossings) / Via state |
| Track which identified targets are visible across a scroll container | `onScrollTargetVisibilityChange(idType:threshold:_:)` + `scrollTargetLayout()` | 18 / 15 | Yes (on crossings) / Via state |
| Drive UI outside the scrolled content from offset, insets, or content/container sizes | `onScrollGeometryChange(for:of:action:)` | 18 / 15 | Yes (on transformed changes) / Via state |

Do not store raw `contentOffset` in `@State` with `onScrollGeometryChange` merely to drive per-item `.scaleEffect`, `.opacity`, or `.offset`, and do not manually compute visibility percentages from scroll geometry when `onScrollVisibilityChange` or `onScrollTargetVisibilityChange` expresses the condition directly.

## Keep per-item visual effects stateless and layout-free

[`scrollTransition(_:axis:transition:)`](https://developer.apple.com/documentation/swiftui/view/scrolltransition(_:axis:transition:)) interpolates [`VisualEffect`](https://developer.apple.com/documentation/swiftui/visualeffect) modifiers across [`ScrollTransitionPhase`](https://developer.apple.com/documentation/swiftui/scrolltransitionphase) (`.topLeading`, `.identity`, `.bottomTrailing`). Leave the `.identity` phase unmodified unless the effect is meant to persist while the view rests on screen; use `phase.value` (`-1`, `0`, `1`) when the leading and trailing edges need opposite directions (such as mirrored rotation or offset). The configuration applies symmetrically on both entry and exit: `.interactive` (default) tracks scroll progress continuously, while `.animated` animates between phases once the view crosses the configured threshold. Passing `axis: nil` uses the innermost scroll view's axis, defaulting to `.vertical` for bidirectional scroll views.

[`visualEffect(_:)`](https://developer.apple.com/documentation/swiftui/view/visualeffect(_:)) receives a `GeometryProxy` alongside the `EmptyVisualEffect` placeholder so an effect can vary continuously with [`frame(in: .scrollView)`](https://developer.apple.com/documentation/swiftui/geometryproxy/frame(in:)) and [`bounds(of: .scrollView)`](https://developer.apple.com/documentation/swiftui/geometryproxy/bounds(of:)). Neither `scrollTransition` nor `visualEffect` changes layout geometry; clip overflow explicitly (for example on a card frame) when offsetting an image for parallax.

Both closures are `@Sendable` in the SDK. Under Swift 6 strict concurrency, capture immutable values explicitly rather than reading mutable `@MainActor` properties inside the closure. Respect `@Environment(\.accessibilityReduceMotion)` by passing a captured flag and keeping motion minimal or zero when Reduce Motion is enabled.

Complete view definitions using caller-supplied identifiers and card height. Replace `RoundedRectangle` with your card media.

```swift
import SwiftUI

@available(iOS 17.0, macOS 14.0, *)
struct ParallaxCarouselCard: View {
    let title: String
    let parallaxFactor: CGFloat
    @Environment(\.accessibilityReduceMotion) private var reduceMotion

    var body: some View {
        let factor = reduceMotion ? 0 : parallaxFactor
        ZStack(alignment: .bottomLeading) {
            LinearGradient(
                colors: [.blue, .indigo],
                startPoint: .topLeading,
                endPoint: .bottomTrailing
            )
            .visualEffect { [factor] effect, proxy in
                let frame = proxy.frame(in: .scrollView)
                let viewportWidth = proxy.bounds(of: .scrollView)?.width ?? 0
                return effect.offset(x: (viewportWidth / 2 - frame.midX) * factor)
            }

            Text(verbatim: title)
                .font(.headline)
                .foregroundStyle(.white)
                .padding()
        }
        .frame(width: 260, height: 180)
        .clipShape(.rect(cornerRadius: 16))
        .scrollTransition { [reduceMotion] effect, phase in
            effect
                .scaleEffect(reduceMotion || phase.isIdentity ? 1 : 0.92)
                .opacity(phase.isIdentity ? 1 : 0.6)
        }
    }
}
```

## Trigger state from visibility thresholds

[`onScrollVisibilityChange(threshold:_:)`](https://developer.apple.com/documentation/swiftui/view/onscrollvisibilitychange(threshold:_:)) fires when the fraction of the view inside the scroll viewport crosses `threshold` (default `0.5`). Per Apple's contract, if the view first appears with the threshold already satisfied, the action is called immediately—allowing visibility-dependent state to initialize without a separate `onAppear` check.

When toggling a `.repeatForever()` animation from visibility, switch the `.animation(_:value:)` modifier to a non-repeating animation (such as `.default`) when the view leaves the threshold; keeping the repeating animation active can prevent the property from settling back to its resting value.

```swift
import SwiftUI

@available(iOS 18.0, macOS 15.0, *)
struct VisibilityPulseCard: View {
    let label: String
    @State private var isVisible = false
    @Environment(\.accessibilityReduceMotion) private var reduceMotion

    var body: some View {
        Text(verbatim: label)
            .frame(width: 240, height: 160)
            .background(.teal.opacity(0.2), in: .rect(cornerRadius: 16))
            .scaleEffect(isVisible && !reduceMotion ? 1.06 : 1)
            .animation(
                isVisible && !reduceMotion
                    ? .easeInOut(duration: 3).repeatForever(autoreverses: true)
                    : .default,
                value: isVisible
            )
            .onScrollVisibilityChange(threshold: 0.9) { visible in
                isVisible = visible
            }
    }
}
```

When the parent container needs a single summary of which items are visible (for example, to coordinate playback across a feed), attach [`onScrollTargetVisibilityChange(idType:threshold:_:)`](https://developer.apple.com/documentation/swiftui/view/onscrolltargetvisibilitychange(idtype:threshold:_:)) to the `ScrollView` and mark the repeating child layout with [`scrollTargetLayout(isEnabled:)`](https://developer.apple.com/documentation/swiftui/view/scrolltargetlayout(isenabled:)). Use the `idType:` parameter label declared by the SDK—Apple's prose discussion sample mistakenly writes `for:`, which fails to compile.

```swift
import SwiftUI

@available(iOS 18.0, macOS 15.0, *)
struct VisibleIDTracker: View {
    let itemIDs: [Int]
    @State private var visibleIDs: [Int] = []

    var body: some View {
        ScrollView {
            LazyVStack(spacing: 12) {
                ForEach(itemIDs, id: \.self) { id in
                    Text("Item \(id)")
                        .frame(maxWidth: .infinity, minHeight: 80)
                }
            }
            .scrollTargetLayout()
        }
        .onScrollTargetVisibilityChange(idType: Int.self, threshold: 0.5) { ids in
            visibleIDs = ids
        }
    }
}
```

## Clamp bounced offsets when driving external UI

Use [`onScrollGeometryChange(for:of:action:)`](https://developer.apple.com/documentation/swiftui/view/onscrollgeometrychange(for:of:action:)) when scrolling must update views outside the `ScrollView` (such as a backdrop or header) and reduce [`ScrollGeometry`](https://developer.apple.com/documentation/swiftui/scrollgeometry) to the smallest `Equatable` value the UI depends on. Attach it directly to the target `ScrollView` (see [scroll-position.md](scroll-position.md) for the multi-scroll-view rule).

Rubber-banding (overscroll bounce) pushes `contentOffset` below zero and beyond the trailing content bound. When projecting `contentOffset` onto an array index, guard against empty collections and non-positive strides (where `cardStride` is card width plus stack spacing), account for leading content insets if present, and clamp the result before indexing:

```swift
import SwiftUI

struct PageSlide: Identifiable, Sendable {
    let id: Int
    let title: String
}

@available(iOS 18.0, macOS 15.0, *)
struct ClampedPageIndexReader: View {
    let pages: [PageSlide]
    let cardWidth: CGFloat
    let spacing: CGFloat
    @State private var activeIndex = 0

    var body: some View {
        let cardStride = cardWidth + spacing
        ScrollView(.horizontal) {
            HStack(spacing: spacing) {
                ForEach(pages) { page in
                    Text(verbatim: page.title)
                        .frame(width: max(cardWidth, 1), height: 180)
                }
            }
            .padding(.horizontal)
        }
        .onScrollGeometryChange(for: Int.self) { geometry in
            guard !pages.isEmpty, cardStride > 0 else { return 0 }
            let adjustedOffset = geometry.contentOffset.x + geometry.contentInsets.leading
            let rawIndex = Int((adjustedOffset / cardStride).rounded())
            return min(max(rawIndex, 0), pages.count - 1)
        } action: { _, newIndex in
            activeIndex = newIndex
        }
        .overlay(alignment: .topLeading) {
            if pages.indices.contains(activeIndex) {
                Text(verbatim: pages[activeIndex].title)
                    .padding(8)
            }
        }
    }
}
```

## Verification and remaining boundaries

All four code blocks were extracted verbatim and passed `swiftc -typecheck` with Xcode 27.2 (27B52), Apple Swift 6.4.2 (swiftlang-6.4.2.1.9), the iPhoneOS 27.2 SDK targeting arm64 iOS 17.0 (first example) and iOS 18.0 (remaining examples), and the macOS 27.2 SDK targeting arm64 macOS 14.0 / 15.0, under Swift 6 with both `nonisolated` and `MainActor` default isolation plus Swift 5 with `nonisolated` default isolation, complete strict concurrency, and warnings as errors.

A separate compiler probe confirmed that `.onScrollTargetVisibilityChange(for: Int.self, threshold: 0.2)` fails to typecheck with `missing argument for parameter 'idType' in call`, while `idType:` compiles cleanly.

No scroll view was hosted or scrolled at runtime. Gesture-driven interpolation, bounce bounds, initial visibility firing, repeating-animation termination, parallax visual clipping, Reduce Motion rendering, older OS execution, and scroll performance remain unverified locally.

## Sources

- [Scroll-driven effects in SwiftUI: phases, geometry and scroll state](https://nilcoalescing.com/blog/ScrollDrivenEffectsInSwiftUI)
