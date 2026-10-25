# Test the out-of-process App Intents path

Use this when directly calling `perform()` misses registration, parameter resolution, or entity transport problems. This guidance covers Xcode 27+ and iOS/iPadOS or macOS 27+. Keep ordinary business-logic unit tests; add these tests for the integration path.

Apple's [App Intents Testing guide](https://developer.apple.com/documentation/appintentstesting/testing-your-app-intents-code) requires a **UI Testing target**, a live app process, and the same signing team as the app. Import `AppIntentsTesting` in that target without importing the app's internal types. [`IntentDefinitions`](https://developer.apple.com/documentation/appintentstesting/intentdefinitions) starts with the app bundle identifier and looks up registered definitions by string names.

Typecheckable XCTest scaffold. Bundle/type/property names and the seed record are application placeholders: it will not run successfully until a matching app registers those definitions and data. The example expects the start intent to return an entity with a String `title`:

```swift
import XCTest
import AppIntentsTesting

@available(iOS 27.0, macOS 27.0, *)
final class TimerIntentTests: XCTestCase {
    @MainActor
    func testStartTimer() async throws {
        let app = XCUIApplication()
        app.launch()
        let definitions = IntentDefinitions(
            bundleIdentifier: "com.example.Timer"
        )
        let timer = definitions.entities["TimerEntity"]
            .makeReference(identifier: "deep-work")
        let result = try await definitions.intents["StartTimerIntent"]
            .makeIntent(timer: timer)
            .run()
        let returned: AnyAppEntity = try result.value
        let title: String = try returned.title
        XCTAssertEqual(title, "Deep Work")
    }
}
```

Seed deterministic data before the assertion. If using setup-only intents, exclude them from shipping configurations and set `isDiscoverable = false`; hiding discoverability alone does not remove an intent from the binary. Keep setup actions narrow and app-specific.

Dynamic names and return-property access move mistakes to runtime. Use `.intents` (plural), verify names and result shape, and exercise the test after renaming types. A failed lookup does not by itself prove existing saved shortcuts broke. `run()` exercises the App Intents infrastructure; this test does not send a spoken utterance or prove Siri discoverability or natural-language resolution. Add entity-query tests when that resolution is part of the requirement.

## Evidence and limits

Official pages and SDK interfaces checked 2026-10-25. The scaffold passed Swift 5/6 typechecking with warnings as errors, complete strict concurrency, nonisolated default isolation, and no extra feature flags using Xcode 27.1 (27A9275), Apple Swift 6.4, macOS 27.0 SDK and its developer test frameworks, targeting macOS 27. No matching app, signed UI test bundle, entity query, Siri/Shortcuts invocation, simulator/device execution, or release-build exclusion was tested. Compilation cannot verify the dynamic names or app contract above.

## Sources

- [iOS Code Review, issue 92](https://ioscodereview.com/issues/issue-92-xcode-27-1-is-final-app-review-names-bait-and-switch-and-your-widgets-are-headed-to-homeview/) (2026-10-20)
