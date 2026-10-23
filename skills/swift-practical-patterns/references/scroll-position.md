# Scroll anchors and position readback

Use this reference when a `ScrollView` needs different initial and small-content alignment, or when a position binding is mistaken for a live offset or an end-of-content test. For card dimensions and margins, see [container-geometry.md](container-geometry.md) and [layout-margins.md](layout-margins.md).

Official documentation and SDK declarations checked 2026-10-23. `defaultScrollAnchor(_:)` requires iOS 17 / macOS 14. Its role-specific overload, `ScrollPosition`, and `onScrollGeometryChange` require iOS 18 / macOS 15 with an SDK that declares them. For iOS 17 / macOS 14, an optional ID binding with `scrollPosition(id:)` remains available; [ScrollViewReader](https://developer.apple.com/documentation/swiftui/scrollviewreader) supports iOS 14 / macOS 11 for older targets or proxy-based control. This reference covers `ScrollView`.

## Separate default placement from a scroll command

`defaultScrollAnchor(_:)` influences initial placement and size-change handling; it is not only an appearance setting. The user can scroll away, and the system may consult it when content size changes. On iOS 18 / macOS 15, use distinct roles when those policies differ: `.initialOffset` for first placement, `.sizeChanges` for content/container resizing, and `.alignment` for content smaller than the viewport. [Default anchor](https://developer.apple.com/documentation/swiftui/view/defaultscrollanchor(_:)), [role-specific anchor](https://developer.apple.com/documentation/swiftui/view/defaultscrollanchor(_:for:))

To start long content at the bottom but center short content, apply `.defaultScrollAnchor(.bottom)` followed by `.defaultScrollAnchor(.center, for: .alignment)`. The second modifier changes only small-content alignment. To change only initial placement instead, use `.defaultScrollAnchor(.bottom, for: .initialOffset)`. Neither configuration defines an application's policy for following incoming messages after the user scrolls away. Issue an explicit position change for a user-requested jump; do not repeatedly reset the binding in `onAppear` merely to set the default placement.

## Track identity with the matching type and anchor

Initialize `ScrollPosition(idType:)` with the actual target ID type and mark the repeating layout with `scrollTargetLayout()`. SwiftUI can use the identities supplied by `ForEach`; an additional `.id(...)` on every row is not inherently required. Check the target layout and ID type before adding identity modifiers. [Position binding and target discovery](https://developer.apple.com/documentation/swiftui/view/scrollposition(_:anchor:)), [scrollTo(id:anchor:)](https://developer.apple.com/documentation/swiftui/scrollposition/scrollto(id:anchor:))

The binding's `anchor` influences both programmatic alignment and which target's ID is reported while scrolling. `.bottom` prefers the bottommost target, rather than the default topmost target. `viewID(type:)` is optional and refers to a target; it is not a measurement proving that the entire last row is visible or that the content edge is reached. If that distinction matters, derive the required condition from scroll geometry. [Identity selection](https://developer.apple.com/documentation/swiftui/scrollposition), [typed ID readback](https://developer.apple.com/documentation/swiftui/scrollposition/viewid(type:))

Unwrap a missing destination before calling `scrollTo(id:)`. An `Optional<ID>`, including `nil`, is itself a valid generic identity argument; it does not mean “skip if absent.” Issue commands with existing concrete IDs and use that type for `idType` and typed readback. Optional-to-concrete readback can succeed, so readback alone does not validate that a target exists in the view hierarchy.

The following complete model and view definitions provide an explicit jump and display the selected target ID. The caller supplies uniquely identified lines and a finite viewport. This is not an automatic-follow chat implementation or an exact “at bottom” indicator.

```swift
import SwiftUI

struct TranscriptLine: Identifiable, Sendable {
    let id: Int
    let text: String
}

@available(iOS 18.0, macOS 15.0, *)
struct TranscriptView: View {
    let lines: [TranscriptLine]
    @State private var position = ScrollPosition(idType: TranscriptLine.ID.self)

    var body: some View {
        VStack {
            ScrollView {
                LazyVStack(alignment: .leading) {
                    ForEach(lines) { line in
                        Text(verbatim: line.text)
                            .frame(maxWidth: .infinity, alignment: .leading)
                    }
                }
                .scrollTargetLayout()
            }
            .defaultScrollAnchor(.bottom)
            .defaultScrollAnchor(.center, for: .alignment)
            .scrollPosition($position, anchor: .bottom)

            Text("Selected target: \(position.viewID(type: Int.self).map { String($0) } ?? "none")")

            Button("Jump to latest") {
                guard let id = lines.last?.id else { return }
                withAnimation {
                    position.scrollTo(id: id, anchor: .bottom)
                }
            }
            .disabled(lines.isEmpty)
        }
    }
}
```

## Read geometry for a live offset

`edge` and `point` describe a configured position rather than continuously measured gesture offsets. The `x` and `y` readback properties require iOS 26 / macOS 26 and have the same configured-position role. User positioning clears the edge/point configuration; `isPositionedByUser` describes that state, not whether a gesture is currently active. Use `onScrollGeometryChange` for live geometry and return only the `Equatable` value the feature needs, such as a threshold Boolean. Attach it directly to the intended scroll view: if its subtree contains multiple scroll views, only the first supplies callbacks and SwiftUI logs a runtime issue. [User-positioned state](https://developer.apple.com/documentation/swiftui/scrollposition/ispositionedbyuser), [x availability](https://developer.apple.com/documentation/swiftui/scrollposition/x), [y availability](https://developer.apple.com/documentation/swiftui/scrollposition/y), [geometry callbacks](https://developer.apple.com/documentation/swiftui/view/onscrollgeometrychange(for:of:action:))

Choose the position representation by what must stay meaningful as content changes. SwiftUI attempts to preserve a view identity through reordering and resizing. A point does not track the same logical item, is not kept at an exact offset through content-size changes, and does not update itself to report the new offset. An edge expresses edge positioning. These are documented policies, not a guarantee of exact offsets for every lazy layout. [Position stability](https://developer.apple.com/documentation/swiftui/scrollposition)

## Verification and remaining boundaries

The exact code block passed `swiftc -typecheck` with Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), the iPhoneOS 27.1 SDK targeting arm64 iOS 18.0, and the macOS 27.0 SDK targeting arm64 macOS 15.0. Checks used complete strict concurrency and warnings as errors: Swift 6 with both `nonisolated` and `MainActor` default isolation, plus Swift 5 mode with `nonisolated` default isolation.

A separate macOS executable passed 11 value-level checks in each language mode for typed ID readback, edge/point commands, and explicit `isPositionedByUser` mutation. A non-nil `Optional<Int>` command read back as `Int`; a nil optional produced a stored optional-nil identity. This checks value semantics without a hosted scroll view. Unguarded `x`/`y` reads were rejected at the macOS 15 deployment target; official documentation and the SDK declare their OS 26 requirement.

No ScrollView was hosted. Gesture-driven readback, ID discovery without extra `.id`, short-content alignment, resizing, data arrival/reordering, tall last rows, older-system execution, accessibility, and performance remain untested. The example and documented policies are not a verified automatic-follow implementation.

## Sources

- [Modern SwiftUI APIs for programmatic scrolling](https://nilcoalescing.com/blog/ModernSwiftUIAPIsForProgrammaticScrolling/)
