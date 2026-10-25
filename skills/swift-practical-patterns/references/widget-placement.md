# Widget placement preferences are not privacy controls

Use this when a widget is unsuitable for a particular gallery or shared display. On iOS/iPadOS, `disfavoredLocations(_:for:)` requires iOS 17+; [`.carPlay`](https://developer.apple.com/documentation/widgetkit/widgetlocation/carplay) requires iOS 26+. Check each location's availability instead of assuming the modifier's introduction covers every value.

Apple describes [`disfavoredLocations(_:for:)`](https://developer.apple.com/documentation/swiftui/widgetconfiguration/disfavoredlocations(_:for:)) as placing a widget in the gallery's **Other** section for the specified locations and families. It is a recommendation to the gallery, not a documented prohibition on placement or a mechanism for removing sensitive data.

Complete helper definition for iOS/iPadOS 26+; pass the app's existing configuration and list every family the preference should cover:

```swift
import SwiftUI
import WidgetKit

@available(iOS 26.0, *)
@MainActor
func discourageCarPlay<C: WidgetConfiguration>(
    _ configuration: C,
    families: [WidgetFamily]
) -> some WidgetConfiguration {
    configuration.disfavoredLocations([.carPlay], for: families)
}
```

Do not report that this helper hides a banking or health widget. A product requiring confidentiality needs an independently verified content and authorization policy. The checked [`WidgetLocation`](https://developer.apple.com/documentation/widgetkit/widgetlocation) catalog has no HomeView-specific value; that absence does not establish a supported HomeView exclusion technique. Do not infer a guaranteed HomeView block from `.carPlay`.

## Evidence and limits

Apple documentation and iPhoneOS 27.1 SDK declarations checked 2026-10-25. The helper passed Swift 5/6 typechecking, warnings as errors, complete strict concurrency, nonisolated default isolation, and no extra feature flags using Xcode 27.1 (27A9275), Apple Swift 6.4, targeting arm64 iOS 26. An unguarded `.carPlay` helper targeting iOS 18 was rejected. No widget extension, gallery, CarPlay, HomeView, redaction, or privacy behavior was run or verified.

## Sources

- [iOS Code Review, issue 92](https://ioscodereview.com/issues/issue-92-xcode-27-1-is-final-app-review-names-bait-and-switch-and-your-widgets-are-headed-to-homeview/) (2026-10-20)
