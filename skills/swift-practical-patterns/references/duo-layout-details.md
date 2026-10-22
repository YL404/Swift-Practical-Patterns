# Duo sizing and presentation details

Use this guidance to tune an existing split arrangement's sizes, choose a sheet placement with the intended bar direction, or review an existing custom vertical toolbar item. For system container margins and carousel insets, read [layout-margins.md](layout-margins.md).

Official documentation verified 2026-10-22. This reference and its examples are iOS-focused. Split sizing and vertical-bar APIs require the iOS 27.1 SDK and runtime; sheet placement itself requires iOS 27.0. Rebuild against the 27.1 SDK for Duo's vertical-bar adaptation; a runtime availability branch alone does not opt an older-SDK binary into that behavior. [Duo preparation](https://developer.apple.com/documentation/technologyoverviews/preparing-your-app-for-iphone-duo)

Read only the applicable section:

- [Split sizing](#split-sizing): ratios, point ranges, ideal size, and priority.
- [Sheet placement and bar direction](#sheet-placement-and-bar-direction): presentation intent and stable bar policy.
- [Custom vertical toolbar legibility](#custom-vertical-toolbar-legibility): fixed width, backgrounds, spacing, and preferred-edge semantics.

## Split sizing

Use these controls for an existing two-region **split** arrangement, with the iOS/iPadOS 27.1 SDK or later and an iOS/iPadOS 27.1+ runtime. Size against the arrangement's available container area, not a percentage of the physical screen. The containing view's aspect ratio and reserved regions affect the arrangement; the same scene can resize in multitasking. The default equal division is documented for a display with one region, not as a promise for every folded configuration. Sources: [split arrangement sizing](https://developer.apple.com/documentation/swiftui/splitarrangementviewstyle), [container-relative layout](https://developer.apple.com/documentation/technologyoverviews/preparing-your-app-for-iphone-duo).

### Choose the sizing input

| Requirement | Modifier and interpretation |
|---|---|
| A region should receive a share of the available space | `splitArrangementLayoutRatio(_:)` takes a preferred ratio from zero to one. Use the axis-specific overload when a side-by-side region needs a different share from a stacked region. Its `minHorizontal`, `idealHorizontal`, and `maxHorizontal` apply to horizontal splits; the corresponding `Vertical` values apply to vertical splits. Sources: [single ratio](https://developer.apple.com/documentation/swiftui/view/splitarrangementlayoutratio(_:)), [axis-specific ratio bounds](https://developer.apple.com/documentation/swiftui/view/splitarrangementlayoutratio(minhorizontal:idealhorizontal:maxhorizontal:minvertical:idealvertical:maxvertical:)). |
| A region has a useful size range in points | `splitArrangementLayoutSize(minWidth:idealWidth:maxWidth:minHeight:idealHeight:maxHeight:)` expresses point-based minimum, ideal, and maximum values. Width constraints apply only to horizontal splits; height constraints apply only to vertical splits. For example, `minWidth: 200, idealWidth: 320, maxWidth: 400` controls the horizontal case, not the height of a stacked region. Source: [point-based size constraints](https://developer.apple.com/documentation/swiftui/view/splitarrangementlayoutsize(minwidth:idealwidth:maxwidth:minheight:idealheight:maxheight:)). |
| A compact region should prefer its content's ideal size | `splitArrangementFixedLayoutSize(horizontal:vertical:)` asks the arrangement to limit the view to its ideal size on the selected axes; both arguments default to `true`. For example, `horizontal: true, vertical: false` prefers the content's ideal width in a horizontal split without requesting its ideal height in a vertical split. This is a preference, not an unshrinkable frame. Source: [fixed ideal size](https://developer.apple.com/documentation/swiftui/view/splitarrangementfixedlayoutsize(horizontal:vertical:)). |

Choose one sizing strategy for the region being tuned rather than layering these modifiers and relying on undocumented precedence. Set only the axis values that express the content's needs. A width constraint does not choose a horizontal layout: `.split.axes(_:)` controls which split directions are permitted. For an excluded axis, the arrangement displays only the primary view. Source: [allowed split axes](https://developer.apple.com/documentation/swiftui/splitarrangementviewstyle/axes(_:)).

### Priority and remaining space

Treat `layoutPriority(_:)` as part of the sizing decision when both regions express competing preferences:

- **Ratio:** the arrangement sizes the view with the highest layout priority using its ratio. If a ratio would exceed the container's remaining size, it uses the remainder instead. If the ratios prevent the two views from filling the container, it increases the highest-priority view's size. A requested fraction therefore is not an exact final-size guarantee. Source: [ratio allocation rules](https://developer.apple.com/documentation/swiftui/view/splitarrangementlayoutratio(_:)).
- **Points:** the arrangement sizes the higher-priority view using its ideal size, clamped to its minimum and maximum values. These constraints follow the active split axis. Source: [point allocation rules](https://developer.apple.com/documentation/swiftui/view/splitarrangementlayoutsize(minwidth:idealwidth:maxwidth:minheight:idealheight:maxheight:)).
- **Fixed ideal size:** the arrangement can still make the view smaller depending on its priority. Do not use “fixed” to imply that compression is impossible. Source: [fixed-size priority caveat](https://developer.apple.com/documentation/swiftui/view/splitarrangementfixedlayoutsize(horizontal:vertical:)).

Give the region whose preference should lead an explicit higher priority, rather than assuming which equal-priority child wins. The general priority contract encourages higher-priority views to shrink later and stretch sooner; it is not a substitute for content that can adapt or scroll when space is scarce. Source: [layout priority](https://developer.apple.com/documentation/swiftui/view/layoutpriority(_:)).

### Complete example: editable text with reachable details

This self-contained iOS view definition uses different preferred shares for horizontal and vertical splits. `NavigationStack` stays outside the arrangement, while each region owns its scrolling. The details link remains in the primary region because even an unrestricted split can show only the primary on a closed iPhone Duo in landscape. The iOS 16–27.0 fallback retains the same editor and details destination rather than dropping functionality. Sources: [arrangement behavior and nesting](https://developer.apple.com/documentation/swiftui/arrangementview), [same functionality across poses and navigation outside](https://developer.apple.com/design/human-interface-guidelines/designing-for-iphone-duo).

```swift
import SwiftUI

@available(iOS 16.0, *)
struct SplitSizingEditor: View {
    @State private var text = ""

    var body: some View {
        NavigationStack {
            Group {
                if #available(iOS 27.1, *) {
                    ArrangementView {
                        editor
                    } secondary: {
                        details
                            .splitArrangementLayoutRatio(
                                idealHorizontal: 0.3,
                                idealVertical: 0.25
                            )
                            .layoutPriority(1)
                    }
                    .arrangementViewStyle(.split)
                } else {
                    editor
                }
            }
            .navigationTitle("Draft")
        }
    }

    private var editor: some View {
        VStack(alignment: .leading) {
            TextEditor(text: $text)
                .accessibilityLabel("Draft text")

            NavigationLink("Document details") {
                details
                    .navigationTitle("Document details")
            }
        }
        .padding()
    }

    private var details: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 12) {
                Text("Document details")
                    .font(.headline)
                Text("Characters: \(text.count)")
            }
            .frame(maxWidth: .infinity, alignment: .leading)
            .padding()
        }
    }
}
```

The secondary region's priority of `1` makes its preferred ratio lead the allocation; the primary retains the default priority. The ratio is only a sizing preference, not a visibility policy. If you later restrict the split to `.horizontal`, keep the primary's details link: excluding the vertical axis can suppress the secondary region. Do not remove an essential feature just because its companion region is not on screen. Sources: [ratio allocation](https://developer.apple.com/documentation/swiftui/view/splitarrangementlayoutratio(_:)), [axis exclusion](https://developer.apple.com/documentation/swiftui/splitarrangementviewstyle/axes(_:)), [functionality continuity](https://developer.apple.com/design/human-interface-guidelines/designing-for-iphone-duo).

If a region needs different *internal* composition for a stacked split, read `@Environment(\.splitArrangementAxis)` inside that region. The documented value is the current split axis, or `nil` outside a split arrangement; it does not replace the axis-specific sizing inputs above. Source: [split-axis environment](https://developer.apple.com/documentation/swiftui/environmentvalues/splitarrangementaxis).

## Sheet placement and bar direction

Use the system sheet first; choose an explicit side only when that placement has a purpose, such as keeping relevant presenting content visible. [`presentationPlacement(_:)`](https://developer.apple.com/documentation/swiftui/view/presentationplacement(_:)) is available on **iOS 27.0+**. Apply it **inside the sheet's content closure**, to the presented content—not to the presenting button or screen. Only sheet presentations respect it; it does not position a popover, alert, or a custom overlay.

[`PresentationPlacement`](https://developer.apple.com/documentation/swiftui/presentationplacement) offers:

| Value | Requested placement |
| --- | --- |
| `.automatic` | Let the system choose; this is the default. |
| `.center` | Center within the presenting view. |
| `.leading` | Use the presenting view's leading edge. |
| `.trailing` | Use the presenting view's trailing edge. |

Placement and toolbar direction are related, but are not the same setting. Apple's [Preparing your app for iPhone Duo](https://developer.apple.com/documentation/technologyoverviews/preparing-your-app-for-iphone-duo) specifies these **default** sheet bar arrangements:

| Context | Default bar arrangement |
| --- | --- |
| Outer display | Vertical, when the sheet has toolbar content. |
| Inner display, centered or leading placement | Horizontal. |
| Inner display, trailing placement | Vertical. |

Keep semantic toolbar actions rather than manually positioning a close control beside the sheet. The system sheet also adapts around the fold: Apple demonstrates sheets moving aside when necessary in [Design for iPhone Duo, Sheet behavior and Fold avoidance](https://developer.apple.com/videos/play/tech-talks/111466/?time=516). Do not promise a fixed half of the display for `.automatic` or `.center` in every partially folded pose, or calculate a sheet offset from the hinge angle. Placement expresses intent within the system presentation, not a fixed frame.

The following are complete view definitions for an iOS app with an **iOS 16+ deployment target and an iOS 27+ SDK**. The example intentionally requests trailing placement; omit the modifier for ordinary automatic placement. On older systems the same sheet and its working close action remain available. No vertical-bar opt-out or detent workaround is part of this default example.

```swift
import SwiftUI

@available(iOS 16.0, *)
struct ReadingGuidePresenter: View {
    @State private var isGuidePresented = false

    var body: some View {
        Button("Show reading guide") {
            isGuidePresented = true
        }
        .sheet(isPresented: $isGuidePresented) {
            if #available(iOS 27.0, *) {
                ReadingGuideSheet()
                    .presentationPlacement(.trailing)
            } else {
                ReadingGuideSheet()
            }
        }
    }
}

@available(iOS 16.0, *)
private struct ReadingGuideSheet: View {
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            List {
                Text("Keep text at a comfortable reading size.")
                Text("Use the system sheet controls to close this guide.")
            }
            .navigationTitle("Reading guide")
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button {
                        dismiss()
                    } label: {
                        Label("Close", systemImage: "xmark")
                    }
                }
            }
        }
    }
}
```

The sheet reads [`dismiss`](https://developer.apple.com/documentation/swiftui/environmentvalues/dismiss) from its own environment. An ordinary button with a `Label` avoids requiring a newer semantic button-role initializer merely to dismiss the presentation.

**Preserve automatic vertical behavior unless a specific interface benefits from horizontal bars.** [`toolbarVerticalBehavior(_:)`](https://developer.apple.com/documentation/swiftui/view/toolbarverticalbehavior(_:)) requires **iOS 27.1+**, separately from the iOS 27.0 placement API. A sheet with only a close action is a possible exception if dedicating a vertical strip to that action reduces useful content width; Apple's [Raise the bar with iPhone Duo, When to opt out](https://developer.apple.com/videos/play/tech-talks/111462/?time=861) demonstrates this case. If that tradeoff is justified, apply `.toolbarVerticalBehavior(.disabled)` to the content inside the sheet's `NavigationStack`, behind `if #available(iOS 27.1, *)` when needed, retaining the unchanged content on older systems. This returns actions to standard horizontal top/bottom bars. Make the choice stable for that interface; do not toggle it with temporary selection, loading, or editing state, and do not use it as a way to hide bars. A sheet resolves its presentation preference independently of its presenter.

**Checks to perform:** compare automatic and deliberately selected placement on both displays, fully open and partially folded, in both orientations and a right-to-left language. Verify that Close remains reachable, toolbar content clears system status/camera regions, and useful content is not clipped. These are checks to perform, not a record of tested behavior. An article's iOS 27.1 beta observation that `.presentationDetents([.large])` avoided a trailing-sheet toolbar overlap is not a documented placement requirement; do not add it to a default template. A reproduced issue on a particular runtime needs its own diagnosis and narrowly justified workaround.

## Custom vertical toolbar legibility

This is a review pass for an **existing custom item in a system toolbar**, not a reason to introduce custom chrome or replace standard buttons. The iOS 27.1+ axis/environment APIs and basic toolbar setup are separate from the content checks below. Apple's [Raise the bar with iPhone Duo, Adapt custom views](https://developer.apple.com/videos/play/tech-talks/111462/?time=607) adds three concrete constraints:

1. **Fit a fixed width.** A vertical bar has fixed width and flexible item height, unlike a horizontal bar's fixed height and flexible width. Audit custom content for horizontal assumptions, clipped labels, and unnecessarily tall representations. A genuinely adaptable custom item may opt into `.axisBehavior(.verticalPreferred)` on iOS 27.1+, but that preference does not make an oversized view fit. If text carries essential information rather than merely repeating the icon, keep that information available instead of hiding it to force the item vertical. Do not opt a view into a vertical bar until its compact representation is usable.
2. **Check both backgrounds.** The vertical bar has no scroll-edge effect by default, but receives a background when **Reduce Transparency** is enabled. Inspect the custom item's symbols, text, selection indication, and any custom surface over both the underlying content and that accessibility background. Check light/dark appearance and varied underlying imagery; avoid a design whose meaning or contrast depends on the transparent case alone. Prefer system controls and semantic styles where they already express the action.
3. **Remove spacing that imitates bar layout.** Flexible toolbar spacers have zero size on the vertical axis; fixed spacers retain their minimum size. Do not add padding, empty items, or fixed spacers to recreate a presumed horizontal gap after migration. Let the system place and group toolbar items. Review existing fixed gaps deliberately, because they still consume scarce vertical room; do not assume all spacers collapse vertically.

If adapting a custom representation, keep real `Button` actions and their meaningful labels. Hiding visible titles must not erase the accessible action name or essential status. Test the actual action as well as its appearance; a compact icon that no longer exposes the same operation is not an adaptation.

**Do not mistake preferred edge for visibility.** On iOS 27.1+, [`toolbarVerticalEdge`](https://developer.apple.com/documentation/swiftui/environmentvalues/toolbarverticaledge) reflects the system's **preferred edge in the current context, regardless of whether a vertical bar is currently visible**. `nil` means the system never places a vertical bar in that device/context; non-`nil` is not proof that a bar is visible or that this particular item is currently vertical. Read it near the custom UI as an edge preference, not as an `isVerticalBarVisible` flag. In particular, do not reserve/reclaim content space, hide an action, or assume a custom item's actual axis solely from a non-`nil` value.

**Checks to perform:** inspect the custom item with Reduce Transparency off/on, light/dark appearance, larger text, and VoiceOver; confirm the label and action survive any symbol-only representation. Include the short outer-display landscape case and transitions between horizontal and vertical contexts to expose clipping and fixed gaps. These checks supplement the usual toolbar review; they do not require introducing a custom item, manual spacing, or a separate bar architecture.

## Verification and remaining boundaries

The two code blocks were extracted verbatim and passed `swiftc -typecheck -swift-version 6 -strict-concurrency=complete` with Xcode 27.1 (27A9275), Apple Swift 6.4, the iPhoneOS 27.1 SDK, and an arm64 iOS 16.0 deployment target. The examples retain older-system functionality without raising the deployment target.

A temporary native host with linked simulator SDK 27.1 displayed `SplitSizingEditor` on the iOS 27.1 iPhone Duo simulator's outer portrait display, including its details link and character count. The same host presented the example's `ReadingGuideSheet` with trailing placement; the screenshot showed readable sheet content and Close in the side bar. The sheet was opened by the host, not by interacting with `ReadingGuidePresenter`'s button.

Typing, details-link navigation, dismissal, exact size ratios or priority competition, the older-system fallback at runtime, rotation, inner-display/partially folded layouts, right-to-left layout, Reduce Transparency, VoiceOver, larger text, and physical-device behavior were not exercised. The custom-toolbar section is supported by Apple's contracts and talk, not a local accessibility or legibility experiment. Do not infer that the full pose or accessibility matrix passed from these initial-display smoke checks.

Sizing preferences and bar policies follow the Apple sources linked beside each conclusion. Article-specific content compression, environmental propagation, fixed-half placement, and beta workarounds are not promoted to universal requirements.

## Sources

- [Set the split size of ArrangementView with splitArrangementLayoutRatio](https://sarunw.com/posts/splitarrangementlayoutratio/)
- [ArrangementView in SwiftUI: split and overlay layouts for iPhone Duo](https://nilcoalescing.com/blog/ArrangementViewInSwiftUISplitAndOverlayLayoutsForIPhoneDuo)
- [Presentation placement on iPhone Duo](https://sarunw.com/posts/presentation-placement-on-iphone-duo/)
- [Presenting SwiftUI sheets on iPhone Duo](https://nilcoalescing.com/blog/PresentingSwiftUISheetsOnIPhoneDuo/)
- [Sheets and fold avoidance on iPhone Duo](https://sarunw.com/posts/sheets-and-fold-avoidance-on-iphone-duo)
- [Adapt custom toolbar views for iPhone Duo](https://sarunw.com/posts/adapt-custom-toolbar-views-for-iphone-duo/)
