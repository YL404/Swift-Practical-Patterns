# Observation: equal assignments and in-place mutation

Use this when an `@Observable` value property receives duplicate snapshots, or an apparently unchanged collection edit still notifies observers. This guidance concerns concrete value properties transformed by the standard macro, not manual `Observable` implementations or all reference/generic property types.

Verified on 2026-10-23 with Apple Swift 6.4 / Xcode 27.1. SwiftUI supports Observation from iOS 17 / macOS 14 and aligned releases. Accessor generation depends on the build toolchain; an OS availability check does not select a different macro expansion. Recheck the expansion when changing compiler versions. [Apple migration guide](https://developer.apple.com/documentation/swiftui/migrating-from-the-observable-object-protocol-to-the-observable-macro)

## Identify the write path

- **Whole-property assignment:** with a concrete `Equatable` value type, the verified generated setter compares old and new values and skips Observation notification when equal. Equality must describe meaningful state; do not exclude fields merely to suppress updates.
- **In-place access through `_modify`:** the verified expansion registers mutation around the yielded storage without comparing before and after. An equal element write or an `inout` helper that changes nothing can still notify. Adding `Equatable` to array elements does not filter this path.
- **Property observers:** skipping notification does not skip the assignment. In the verified expansion, equal assignments still write backing storage, preserving `didSet`. Observation notification is not a deduplicated business-event channel.

These are implementation details confirmed against the [Swift macro source](https://github.com/swiftlang/swift/blob/7d24f7c0082f614351073e8c359bcb4012083947/lib/Macros/Sources/ObservationMacros/ObservableMacro.swift#L449) and the installed compiler's expansion, not a guarantee that every future macro or mutating expression behaves identically. The source deliberately avoids equality checking in `_modify` to avoid extra copy-on-write work; do not rewrite every mutation as copy-edit-reassign just to reach the setter.

## Guard known duplicate edits before mutation

The following complete model declarations accept whole snapshots and avoid entering the collection's modifying accessor for a duplicate field edit. IDs must be unique within a snapshot; a missing ID is deliberately ignored. Methods are synchronous on `MainActor`, with no suspension between the check and write.

```swift
import Observation

struct DeliveryStop: Equatable {
    let id: Int
    var isComplete: Bool
}

@MainActor
@Observable
final class DeliveryModel {
    private(set) var stops: [DeliveryStop] = []

    func applySnapshot(_ incoming: [DeliveryStop]) {
        stops = incoming
    }

    func setCompleted(_ completed: Bool, for id: Int) {
        guard let index = stops.firstIndex(where: { $0.id == id }),
              stops[index].isComplete != completed else { return }
        stops[index].isComplete = completed
    }
}
```

Use `applySnapshot` for incoming state snapshots; `[DeliveryStop]` has value equality including order. Use the guarded edit when repeated commands frequently request the existing value. The guard only removes this known redundant write: real edits still notify readers of the collection property, and it does not create per-element observation. Large-array equality and the ID lookup have costs; measure the actual workload before claiming a speedup.

## Validation scope

The model and a separate Observation harness compiled and executed on macOS 27.0.1 arm64, Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), in Swift 6 and Swift 5 language modes, complete strict concurrency, default isolation explicitly nonisolated, with no extra feature flags. The harness targeted macOS 15 because its thread-safe callback counter used `Mutex`; the model itself typechecked for macOS 14 and iOS 17 with the macOS 27.0 and iPhoneOS 27.1 SDKs respectively.

Each of 15 cases installed fresh [`withObservationTracking`](https://developer.apple.com/documentation/observation/withobservationtracking(_:onchange:)) tracking before one operation. Equal non-Equatable value assignment notified; equal Equatable value/array replacement did not. Changed replacements, equal array-element writes, unchanged `inout`, and append notified; an unrelated property did not. Equal assignment still ran `didSet`. The model notified for initial snapshots and changed edits, but not duplicate snapshots, guarded duplicate edits, or missing IDs.

These checks establish local model notifications, not SwiftUI body counts, rendered output, event ordering across asynchronous updates, or performance. No older compiler/runtime, iOS execution, simulator, or device behavior was tested.

## Sources

- https://nilcoalescing.com/blog/EquatablePropertiesInObservableClasses/
