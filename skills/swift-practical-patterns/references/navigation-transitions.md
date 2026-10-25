# Choosing a navigation transition at runtime

Use this reference when the same sheet needs different presentation transitions, or when a conditional transition fails because its branches have different concrete types. It covers iOS/iPadOS sheets; it does not prescribe a transition for every navigation path.

Verified against Apple documentation on 2026-10-25. `navigationTransition` requires iOS 18; `AnyNavigationTransition` and `crossFade` require iOS 27 and an SDK that declares them. Earlier systems can keep the default sheet presentation.

## Configure the presented root

Apply [navigationTransition(_:)](https://developer.apple.com/documentation/swiftui/view/navigationtransition(_:)) to the root content passed to the sheet, outside its internal layout containers. Use [.automatic](https://developer.apple.com/documentation/swiftui/navigationtransition/automatic) for the system's context-dependent choice. Use [.crossFade](https://developer.apple.com/documentation/swiftui/navigationtransition/crossfade) when the intended sheet appearance is a fade over the presenting content. A cross-fade does not require a matched source view.

The modifier takes one concrete `NavigationTransition` type. When runtime choices have different types, wrap each in [AnyNavigationTransition](https://developer.apple.com/documentation/swiftui/anynavigationtransition), then pass the resulting value to the same modifier. Erase the transition value rather than wrapping the presented view in `AnyView` or forcing a new `.id`.

This is a complete iOS-only view definition, with an unmodified sheet on iOS 14–26. The Boolean represents an app-selected presentation policy; determine it for the presentation rather than assuming that changing it while a sheet is open restarts the transition.

```swift
import SwiftUI

@available(iOS 14.0, *)
struct AdaptiveSheetTransition: View {
    let useCrossFade: Bool
    @State private var showDetails = false

    var body: some View {
        Button("Show details") {
            showDetails = true
        }
        .sheet(isPresented: $showDetails) {
            if #available(iOS 27.0, *) {
                let transition = useCrossFade
                    ? AnyNavigationTransition(.crossFade)
                    : AnyNavigationTransition(.automatic)
                Text("Details")
                    .navigationTransition(transition)
            } else {
                Text("Details")
            }
        }
    }
}
```

Keep the existing deployment target. A runtime availability check cannot make an older SDK recognize a new type. Also check each wrapped transition separately: `AnyNavigationTransition` exists on macOS 27, but [`CrossFadeNavigationTransition`](https://developer.apple.com/documentation/swiftui/crossfadenavigationtransition) is unavailable on macOS. Type erasure does not add platform support to the value it wraps.

This API controls navigation/presentation. Ordinary conditional content insertion uses `transition(_:)`; it is a different operation. Retain normal dismissal controls and test the selected presentation with Reduce Motion and the actual sheet configuration.

## Validation scope

The exact view definition passed typechecking with Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), and the iPhoneOS 27.1 SDK targeting arm64 iOS 14. Checks used Swift 6, complete strict concurrency, warnings as errors, and both `nonisolated` and `MainActor` default isolation. Separate availability checks accepted the new types on iOS 27, rejected unguarded use on iOS 26, and confirmed that `.crossFade` is unavailable on macOS 27 even though `AnyNavigationTransition(.automatic)` is available there.

No sheet was launched. Appearance, dismissal gestures, live policy changes, older-runtime fallback, Reduce Motion, and full-screen-cover or stack transitions were not exercised. Compilation does not establish their visual behavior.

## Sources

- https://nilcoalescing.com/blog/SwiftUINavigationTransitionUpdatesIniOS27/
