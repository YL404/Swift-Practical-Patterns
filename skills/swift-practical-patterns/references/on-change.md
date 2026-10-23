# onChange migration: captures and initial delivery

Use this when migrating `onChange(of:perform:)`, a capture list unexpectedly reports the new value, or initialization must run alongside change handling. This supplements basic overload selection; it does not define an event log for every assignment.

Verified on 2026-10-23 with Xcode 27.1 / Apple Swift 6.4. The zero- and two-parameter overloads require iOS 17 / macOS 14 and aligned releases. The single-parameter overload supports iOS 14 / macOS 11 and is deprecated from iOS 17 / macOS 14; retain it when an older deployment target requires it. [Modern View API](https://developer.apple.com/documentation/swiftui/view/onchange(of:initial:_:)), [legacy API](https://developer.apple.com/documentation/swiftui/view/onchange(of:perform:))

## Preserve the old value explicitly

The legacy action receives the new value; a capture list can supply the old value. The modern action runs the updated closure, so its captured values reflect the state after the observed change. Simply changing a one-parameter closure to a zero-parameter closure while retaining `[count]` changes its meaning. Use the two-parameter overload whenever the previous observed value is needed. The same updated-closure rule applies to `Scene.onChange`. [Two-parameter contract](https://developer.apple.com/documentation/swiftui/view/onchange(of:initial:_:)-4psgg), [Scene contract](https://developer.apple.com/documentation/swiftui/scene/onchange(of:initial:_:))

This complete view declaration illustrates the modern behavior. Supply `count` from a parent; the explicit capture is diagnostic, not needed in normal change handlers.

```swift
import SwiftUI

struct CountChangeProbe: View {
    let count: Int

    var body: some View {
        Text(count, format: .number)
            .onChange(of: count, initial: true) { [count] oldValue, newValue in
                print("previous:", oldValue, "current:", newValue, "captured:", count)
            }
    }
}
```

For a delivered change from 0 to 1, `oldValue` is 0 and `newValue` and the explicit capture are 1. With the legacy overload, `{ [count] newValue in ... }` uses the previous capture instead. Do not use a mutable reference capture as an immutable snapshot of that object's fields.

## Treat initial delivery as appearance work

`initial` defaults to `false`. With `true`, the action also runs when the view initially appears; in the two-parameter overload, old and new values are equal on that call. Keep the work safe to repeat if the view is removed and created again. This is not a persistent once-per-app initialization mechanism. [Initial parameter and old-value contract](https://developer.apple.com/documentation/swiftui/view/onchange(of:initial:_:)-4psgg)

The observed value must be `Equatable`; an equal assignment is not a new change event. A binding setter and an `onChange` handler have different responsibilities: a setter participates in writes through that binding, while `onChange` responds to observed value changes, including programmatic ones. Preserve validation or normalization at the write boundary when migrating side effects.

Keep the synchronous action short. Creating a `Task` does not by itself guarantee expensive work leaves the main actor or give it view-owned cancellation; choose execution isolation and task ownership for the actual asynchronous operation. [Task ownership and cancellation](https://developer.apple.com/documentation/swift/task)

## Validation scope

The complete declaration above typechecked in Swift 5 and Swift 6 language modes, complete strict concurrency, default isolation explicitly nonisolated, with no additional feature flags: macOS 14 target against the macOS 27.0 SDK, and iOS 17 target against the iPhoneOS 27.1 SDK. Unguarded use was rejected at macOS 13 / iOS 16 targets as expected.

A separate `NSHostingView` harness executed on macOS 27.0.1 arm64, Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), Swift 6 mode with the same concurrency configuration. It checked initial delivery, changes from 0 to 1 and 1 to 2, an equal assignment, and creation of a new hosting view. The legacy capture reported the old value; zero- and two-parameter captures reported the new value. Initial delivery used equal old/new values, equal assignment produced no action, and a new view identity received initial delivery again.

These checks cover local View callbacks, not rendered appearance, Scene callbacks, rapid/coalesced writes, asynchronous work, background execution, iOS runtime behavior, or older systems/toolchains. Treat the callback as a reaction to observed state, not a guaranteed record of each intermediate write.

## Sources

- https://nilcoalescing.com/blog/OverviewOfonChangeInSwiftUI/
