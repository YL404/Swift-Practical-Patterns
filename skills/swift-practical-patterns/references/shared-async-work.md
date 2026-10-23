# Share an in-flight asynchronous load

Use this when concurrent callers need the same immutable snapshot and should share one load, then reuse its successful result. This is an app-local, single-value cache, not a complete authentication or refresh service. Scope: Swift concurrency with a `Sendable` result; the exact implementation below was checked with Swift 6.4 in Swift 5 and Swift 6 language modes.

## Publish the task before suspending

An actor serializes access to its state, but another call can enter while an earlier call awaits I/O. Checking a cache and then awaiting a load can therefore start duplicate work. Store an in-flight task before the first suspension and let later callers await that same handle. Apple's [actor reentrancy example](https://developer.apple.com/videos/play/wwdc2021/10133/?time=719) uses this distinction between in-progress and ready entries.

Complete actor definition; the injected operation supplies the application-specific load:

```swift
actor SharedSnapshot<Value: Sendable> {
    private var cached: Value?
    private var inFlight: Task<Value, Error>?
    private let load: @Sendable () async throws -> Value

    init(load: @escaping @Sendable () async throws -> Value) {
        self.load = load
    }

    func value() async throws -> Value {
        if let cached { return cached }
        if let inFlight { return try await inFlight.value }

        let task = Task { [load] in try await load() }
        inFlight = task
        defer { inFlight = nil }

        let value = try await task.value
        cached = value
        return value
    }
}
```

Only the call that creates the task clears `inFlight` and publishes the successful result. A failed load is not cached; after that call unwinds, a subsequent call can retry. Joining callers only await the handle. This ownership avoids an old joiner's cleanup clearing a later retry. There is no suspension between successful publication and clearing the task.

## Choose who owns cancellation

The stored `Task` deliberately has a lifetime independent of any one waiter. Its handle supports result/error retrieval and explicit cancellation; unstructured does not mean uncontrollable. [SE-0304](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#unstructured-tasks)

This implementation does not reject already-cancelled callers or stop awaiting when a caller is cancelled. A cancelled caller may start or join a load and receive its result; cancellation is not forwarded to the shared task. Check cancellation in the caller before starting work or committing a result when appropriate. Do not forward each waiter's cancellation to `task.cancel()`: that would cancel work needed by other callers. Cancellation is cooperative, as described by [Task](https://developer.apple.com/documentation/swift/task).

Use this policy only when finishing the shared operation is acceptable. Prompt cancellation of individual waits, stopping work when the last waiter leaves, or owner shutdown requires a different lifetime design. A task group tied to one requesting caller also does not automatically express shared ownership by independent callers.

## Keep execution and freshness separate

Creating this task is not an offloading guarantee. The injected operation's isolation determines where its work executes; choose an explicit `@concurrent` async operation for expensive work that must leave actor isolation with a supporting Swift 6.2+ compiler. A plain nonisolated async operation's behavior depends on `NonisolatedNonsendingByDefault`, independently of the module's default actor isolation. [SE-0461](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md), [SE-0466](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)

The successful value is retained for this instance's lifetime. There is no expiry, invalidation, logout, key-based lookup, or capacity policy. If adding invalidation or replacement while a task is suspended, guard both publication and cleanup with an operation identity or generation; an old completion must not overwrite newer state. The article's token example does not implement token expiry or logout safety.

## Evidence and limits

Sources checked 2026-10-23. Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), macOS 27.0.1 arm64. The exact actor definition passed typechecking for arm64 iOS 15 with the iPhoneOS 27.1 SDK and arm64 macOS 12 with the macOS 27.0 SDK. Six configurations used complete strict concurrency and warnings as errors: Swift 6 with nonisolated/MainActor defaults, and Swift 5 with a nonisolated default, each with NonisolatedNonsendingByDefault disabled/enabled.

A local macOS harness using that definition passed in all six configurations: 64 callers with a gated producer used one load; successful values, including an optional nil, were cached; failure allowed a later retry; cancelling the initiating task while its producer was suspended did not cancel the producer; that caller and a second cancelled caller both received its value. Separate isolation probes confirmed the ordinary nonisolated async call changed with the feature flag, while explicit nonsending/concurrent calls did not. These are local control-flow observations, not proof of behavior on older compilers or runtimes. No production network, authentication, UI, device, invalidation implementation, or performance test was run.

## Sources

- [How I Architect SwiftUI Apps for Swift Concurrency – So the Compiler Stays Quiet](https://www.swiftyplace.com/blog/swiftui-architecture-swift-concurrency)
