# Inspector presentation, sizing, and sheet boundaries

Use this reference when choosing `inspector(isPresented:content:)` for an adaptive trailing panel / compact sheet, deciding where to attach it, or diagnosing width, toolbar, and sheet-detent behavior on iOS 26–27.

Apple documentation, release notes, and SDK interfaces verified 2026-11-24. [`inspector(isPresented:content:)`](https://developer.apple.com/documentation/swiftui/view/inspector(ispresented:content:)), [`inspectorColumnWidth(_:)`](https://developer.apple.com/documentation/swiftui/view/inspectorcolumnwidth(_:)), and [`inspectorColumnWidth(min:ideal:max:)`](https://developer.apple.com/documentation/swiftui/view/inspectorcolumnwidth(min:ideal:max:)) require iOS 17 / macOS 14 and are unavailable on visionOS.

## Attach `.inspector` outside navigation containers and support explicit dismissal

Place `.inspector(isPresented:content:)` outside `NavigationStack` or `NavigationSplitView` when the panel should span the full column height alongside navigation chrome rather than sitting under a pushed view's navigation bar. In local iPad Pro 13-inch (iOS 27.2) simulator measurements, attaching `.inspector` outside `NavigationStack` or `NavigationSplitView` produced a full-height trailing column (`1,319` pt tall), whereas attaching it inside `NavigationStack` constrained it below the navigation bar (`1,213` pt tall).

- **Provide an explicit close action:** set the presentation binding to `false` (or call `dismiss()` inside the inspector content) so users have a reliable dismissal control in both trailing-column and compact-sheet presentations.
- **Account for iOS 26+ inspector toolbars and column requirements:** Apple's [iOS & iPadOS 26 Release Notes](https://developer.apple.com/documentation/ios-ipados-release-notes/ios-ipados-26-release-notes) document (145686228) that apps compiled against iOS/iPadOS 26 may require a `NavigationStack` inside the inspector content view to display toolbar items, and that on iOS/iPadOS 26 inspector columns did not present without a `NavigationSplitView` in the hierarchy. In local iPad Pro 13-inch iOS 27.2 simulator checks, `NavigationStack` without `NavigationSplitView` did present a trailing column, so treat the `NavigationSplitView` requirement as an iOS 26 compatibility check when targeting iOS 26 runtimes.
- **State restoration and macOS collapse:** Apple documents that trailing-column inspectors have their presentation state restored by the framework, and that a fixed `inspectorColumnWidth(_:)` does not prevent users from collapsing the inspector on macOS—add [`interactiveDismissDisabled(_:)`](https://developer.apple.com/documentation/swiftui/view/interactivedismissdisabled(_:)) on the inspector content when collapse should be disallowed. [inspector](https://developer.apple.com/documentation/swiftui/view/inspector(ispresented:content:)), [inspectorColumnWidth(_:)](https://developer.apple.com/documentation/swiftui/view/inspectorcolumnwidth(_:))

Complete view definition for iOS 17+ / macOS 14+. `ContentPlaceholder` and the `Form` text stand in for real editor and inspector controls. Inspector content is wrapped in its own `NavigationStack` so toolbar actions remain supported on iOS 26+, and offers an explicit close button in both column and compact-sheet presentations.

```swift
import SwiftUI

@available(iOS 17.0, macOS 14.0, *)
struct DocumentEditorView: View {
    @State private var isInspectorPresented = false

    var body: some View {
        NavigationStack {
            ContentPlaceholder()
                .navigationTitle("Document")
                .toolbar {
                    Button {
                        isInspectorPresented.toggle()
                    } label: {
                        Label("Inspector", systemImage: "sidebar.trailing")
                    }
                }
        }
        .inspector(isPresented: $isInspectorPresented) {
            NavigationStack {
                Form {
                    Text("Formatting options")
                }
                .navigationTitle("Inspector")
                .toolbar {
                    ToolbarItem(placement: .cancellationAction) {
                        Button("Close") {
                            isInspectorPresented = false
                        }
                    }
                }
            }
            .inspectorColumnWidth(min: 220, ideal: 280, max: 400)
            .presentationDetents([.medium, .large])
        }
    }
}

@available(iOS 17.0, macOS 14.0, *)
private struct ContentPlaceholder: View {
    var body: some View {
        Text("Editor canvas")
    }
}
```

## Treat `inspectorColumnWidth` as a preference, not a cross-version guarantee

Apple documents both `inspectorColumnWidth` overloads as *preferred* widths and notes that SwiftUI may use a different width if the presentation environment does not support the requested constraint. In addition, Apple's [iOS & iPadOS 26 Release Notes](https://developer.apple.com/documentation/ios-ipados-release-notes/ios-ipados-26-release-notes) list Known Issue `145162377`: on iOS and iPadOS 26, `inspector` does not respect the width applied with `inspectorColumnWidth`. [inspectorColumnWidth(min:ideal:max:)](https://developer.apple.com/documentation/swiftui/view/inspectorcolumnwidth(min:ideal:max:))

In local iOS 27.2 (Xcode 27.2, 27B52) simulator measurements on iPad Pro 13-inch, both `.inspectorColumnWidth(450)` and `.inspectorColumnWidth(min: 380, ideal: 450, max: 520)` widened the trailing column from the default `280` pt to `450` pt across `NavigationStack` and `NavigationSplitView` placements. Keep inspector layouts flexible anyway: apps supporting iOS 26 or narrower regular contexts (such as an unfolded iPhone Duo inner display, where width behavior was not verified locally) cannot rely on an exact column width.

## Do not rely on programmatic `presentationDetents(_:selection:)` inside compact `.inspector`

When an inspector adapts to a sheet in a compact horizontal size class, [`presentationDetents(_:)`](https://developer.apple.com/documentation/swiftui/view/presentationdetents(_:)) lets the user drag between detents (for example `[.medium, .large]`) without affecting the trailing-column presentation.

However, in local iOS 27.2 (Xcode 27.2, 27B52) simulator tests on iPhone 18 Pro, **programmatically updating the selection binding of [`presentationDetents(_:selection:)`](https://developer.apple.com/documentation/swiftui/view/presentationdetents(_:selection:)) did not resize a compact `.inspector` sheet**: updating `selectedDetent` from `.medium` to `.large` updated the SwiftUI state, yet the inspector sheet remained `402 × 436` pt (both inside and outside `NavigationStack`), whereas an identical `.sheet(isPresented:)` resized immediately from `402 × 436` pt to `402 × 778` pt. If a compact panel requires code-driven detent transitions rather than user dragging alone, verify on the target runtime or present a `.sheet` on compact width instead of assuming `.inspector` matches `.sheet` parity.

Regarding interactive swipe-to-dismiss on compact width: the source article observed on iOS 27.2 that an inspector dismissed by swiping down failed to reopen until closed via its button. That failure did **not** reproduce in local Xcode 27.2 (27B52) / iOS 27.2 simulator tests across iPhone 18 Pro and iPhone Duo's outer compact display (with and without `presentationDetents`, inside and outside `NavigationStack`): swiping down reset `isPresented` to `false` and tapping the show button reopened the inspector in all tested configurations. Still verify swipe dismissal and reopen on target devices—especially across live compact/regular transitions such as folding and unfolding—rather than assuming every build behaves identically.

## Verification and remaining boundaries

The example above was extracted verbatim and passed `swiftc -typecheck` with Xcode 27.2 (27B52), Apple Swift 6.4.2 (swiftlang-6.4.2.1.9), the iPhoneOS 27.2 SDK targeting arm64 iOS 17.0, and the macOS 27.2 SDK targeting arm64 macOS 14.0, using Swift 6 with both `nonisolated` and `MainActor` default isolation plus Swift 5 with `nonisolated` default isolation, complete strict concurrency, and warnings as errors.

A separate temporary XCUITest probe (built with Xcode 27.2 / 27B52, iOS 26.0 deployment target) ran on iOS 27.2 simulators:
- **iPhone 18 Pro (compact, iOS 27.2):** `.inspector` (inside and outside `NavigationStack`) with `.presentationDetents([.medium, .large], selection:)` opened at `402 × 436` pt, failed to resize (`402 × 436` pt) when `selectedDetent` was set to `.large` in code, and reopened after swipe-down dismissal (`isPresented` returned to `false`). The identical `.sheet` control resized from `402 × 436` pt to `402 × 778` pt on the same state write.
- **iPhone Duo outer display (compact, iOS 27.2):** `.inspector` across four configurations (inside/outside `NavigationStack`, no detents / set-only detents / selection detents) opened (`374 × 636` or `374 × 356` pt), reset `isPresented` to `false` on swipe-down dismissal, and reopened cleanly.
- **iPad Pro 13-inch (M5) (regular, iOS 27.2):** default trailing columns measured `280` pt wide (`1,213` pt tall inside `NavigationStack`, `1,319` pt tall outside `NavigationStack` or `NavigationSplitView`); `.inspectorColumnWidth(450)` and `.inspectorColumnWidth(min: 380, ideal: 450, max: 520)` all measured `450` pt wide.

iOS 26 runtimes, macOS runtime execution and divider-collapse behavior, iPhone Duo unfolded inner-display and fold/unfold transitions, manual user detent dragging, inspector toolbar rendering, state restoration across cold restarts, VoiceOver, and physical devices were not exercised locally.

## Sources

- https://tanaschita.com/swiftui-inspector
