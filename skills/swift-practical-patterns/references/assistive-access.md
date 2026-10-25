# Choose an Assistive Access integration

Use this for an iPhone or iPad experience intended for people with cognitive disabilities. An enlarged font alone does not establish a streamlined interaction flow. Choose the integration before adding configuration keys.

## Match the configuration to the experience

| Intent | Configuration and version |
|---|---|
| Provide a separate, focused interface with Assistive Access control styling | On iOS/iPadOS 26+, set the Boolean [`UISupportsAssistiveAccess`](https://developer.apple.com/documentation/bundleresources/information-property-list/uisupportsassistiveaccess) to `YES` and add an `AssistiveAccess` scene alongside the normal scene. |
| Keep an existing suitable interface and expand its available space | On iOS/iPadOS 17+, set the Boolean [`UISupportsFullScreenInAssistiveAccess`](https://developer.apple.com/documentation/bundleresources/information-property-list/uisupportsfullscreeninassistiveaccess) to `YES`. This uses the ordinary interface above the system Back button. |
| Make a small contextual adjustment | Read [`accessibilityAssistiveAccessEnabled`](https://developer.apple.com/documentation/swiftui/environmentvalues/accessibilityassistiveaccessenabled) inside a view. This SwiftUI environment value requires iOS/iPadOS 18+, independently of the feature's iOS 17 introduction. |

Apple's [integration guide](https://developer.apple.com/documentation/accessibility/optimizing-your-app-for-assistive-access) distinguishes the separate-scene and existing-interface paths. Without full-screen opt-in, a supporter can still enable an app, which normally runs in a reduced frame. Full screen still leaves the system Back button; preserve safe areas and adaptable dimensions. These keys declare an intended experience, rather than verifying its usability.

## Keep the normal scene and supply the focused scene

[`AssistiveAccess`](https://developer.apple.com/documentation/swiftui/assistiveaccess) is used on iOS/iPadOS; its declaration on other platforms does not give them this experience. Gate it by availability when preserving an earlier deployment target.

Complete demonstration for an iOS/iPadOS 26+ app; the button changes local text only. Set `UISupportsAssistiveAccess = YES` in the built app's Info.plist separately:

```swift
import SwiftUI

@main
struct AssistiveAccessDemo: App {
    var body: some Scene {
        WindowGroup {
            GreetingView(showsDetails: true)
        }
        AssistiveAccess {
            GreetingView(showsDetails: false)
        }
    }
}

struct GreetingView: View {
    let showsDetails: Bool
    @State private var greeting = "Ready"

    var body: some View {
        VStack {
            Text(greeting)
            Button("Say hello", systemImage: "hand.wave") {
                greeting = "Hello"
            }
            if showsDetails {
                Text("This demonstration changes text on this screen.")
            }
        }
        .padding()
    }
}
```

Provide the focused root with the models and environment dependencies it needs; do not assume modifiers attached only to the `WindowGroup` content also configure the other scene. The demonstration has no shared data or persistence.

For layout inspection in Xcode 26+, the following separate preview fragment uses the view above:

```swift
#Preview(traits: .assistiveAccess) {
    GreetingView(showsDetails: false)
}
```

The [preview trait](https://developer.apple.com/documentation/developertoolssupport/previewtrait/assistiveaccess) requires iOS/iPadOS 26+ for this appearance and is a no-op on other platforms. It previews styling, not installed app configuration, scene selection, or the complete system experience.

Apple's [WWDC25 session](https://developer.apple.com/videos/play/wwdc2025/238/) recommends a small set of core tasks, visible controls instead of hidden gestures, and interaction without time pressure. Check the actual navigation and necessary workflows with users; standard control styling alone cannot establish cognitive accessibility.

## Evidence and limits

Official documentation and SDK interfaces checked 2026-10-25. The exact app example, with and without the preview fragment, passed Swift 6 typechecking targeting iOS 26 with the iPhoneOS 27.1 SDK, complete strict concurrency, warnings as errors, and both nonisolated and MainActor default isolation. An isolated environment-reader probe passed targeting iOS 18 and failed at iOS 17; the app example failed at iOS 18 because its scene needs iOS 26. Checks used Xcode 27.1 (27A9275), Apple Swift 6.4, macOS 27.0.1 arm64, and no extra upcoming features.

No app bundle, preview, simulator, or device was run. Info.plist packaging, Settings listing, scene activation, row/grid configurations, the system Back button, environment sharing, and usability were not verified by compilation. UIKit scene bridging is outside this example.

## Sources

- [Fatbobman's Swift Weekly #158: Who Is the Real Threat to AI-Relay Apps?](https://fatbobman.com/en/weekly/issue-158/) (2026-10-19)
