# Awaited cleanup with defer

Use this when resource cleanup or asynchronous test teardown must finish before its caller continues, including an error exit. `defer { await cleanup() }` requires a **Swift 6.4 compiler** and an explicitly or implicitly asynchronous enclosing context. A runtime availability check cannot enable this compiler feature. With an older compiler, explicitly await cleanup on both success and error paths; preserve the operation's error policy.

## Keep cleanup within the operation

[SE-0493](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md), implemented in Swift 6.4, permits ordinary `await` calls inside `defer`; no `defer async` spelling is needed. Scope exit implicitly waits for those calls. Moving cleanup into `Task { ... }` instead creates independent work, so returning from the enclosing function does not wait for cleanup.

Register the `defer` immediately after successful acquisition. If acquisition partially succeeds and then throws before reaching it, acquisition must handle its own partial cleanup. The [Swift language reference](https://github.com/swiftlang/swift-book/blob/main/TSPL.docc/ReferenceManual/Statements.md#defer-statement) describes cleanup at the end of the enclosing lexical scope, including error exits, and reverse declaration order within that scope. A nested block's cleanup therefore finishes before execution continues beyond that block. An unreached `defer` has no effect.

Complete command-line demonstration; `Session` is an in-memory stand-in, not a file or network implementation. Compile with `swiftc -parse-as-library`:

```swift
enum WorkError: Error { case failed }

actor Session {
    private(set) var closed = false

    func work(fail: Bool) throws {
        if fail { throw WorkError.failed }
    }

    func close() async {
        await Task.yield()
        closed = true
    }
}

func use(_ session: Session, fail: Bool) async throws {
    defer { await session.close() }
    try await session.work(fail: fail)
}

@main struct Demo {
    static func main() async throws {
        for fail in [false, true] {
            let session = Session()
            do {
                try await use(session, fail: fail)
            } catch WorkError.failed {
                // The caller receives the work error after close finishes.
            }
            let closed = await session.closed
            precondition(closed)
        }
    }
}
```

## Preserve cancellation and error policy

- Cancellation is cooperative: [Task.cancel()](https://developer.apple.com/documentation/swift/task/cancel()) flags cancellation; it does not force arbitrary code to exit. Cleanup starts when control actually leaves its scope.
- Cancellation remains visible inside deferred cleanup. For example, [Task.sleep(for:)](https://developer.apple.com/documentation/swift/task/sleep(for:tolerance:clock:)) throws `CancellationError` if cancelled while waiting. A completed `defer` body does not prove a cancellation-sensitive resource API successfully released anything. Choose a cleanup API and failure policy that satisfy the resource's contract; shielding is an explicit, version-gated choice described below.
- A `defer` cannot transfer control out of its body. Catch cleanup errors inside it when they can be handled there. If cleanup failure must reach the caller, use explicit control flow and decide how to report both operation and cleanup failures; do not silently discard either with `try?`.
- The body inherits its enclosing isolation. Its awaited calls can still suspend; async cleanup does not make an actor operation atomic across suspension.

## Shield bounded cleanup when cancellation would prevent release

[SE-0504](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0504-task-cancellation-shields.md) adds `withTaskCancellationShield` in Swift 6.4. On Apple platforms the checked SDK requires OS 27+; unlike async `defer` syntax, this needs runtime support and cannot back-deploy. Use it only when a resource's cleanup contract needs a short, bounded operation to stop observing task cancellation. It neither makes cleanup infallible nor supplies a timeout.

This helper targets iOS/iPadOS and macOS. Call it only after successful acquisition. It preserves the caller's actor isolation. Complete helper; `release` represents the app's bounded, nonthrowing cleanup implementation, not a real resource API:

```swift
@available(iOS 27.0, macOS 27.0, *)
nonisolated(nonsending) func withReleasedResource(
    operation: () async throws -> Void,
    release: () async -> Void
) async throws {
    defer {
        await withTaskCancellationShield {
            await release()
        }
    }
    try await operation()
}
```

Inside the shield, static `Task.isCancelled` is false and `Task.checkCancellation()` does not throw; the task handle still exposes the real cancelled state. Leaving restores cancellation visibility. Structured children created within the shield do not inherit incoming cancellation automatically, but explicit child/group cancellation still works. Cancellation handlers installed inside the shield are suppressed; handlers installed outside are not covered. Shielding only `group.addTask` does not enclose a child's execution. Keep shields around the required cleanup rather than retries, main work, or waits whose completion depends on cancellation.

## Evidence and limits

Official sources checked 2026-10-22.

Validated on macOS 27.0.1 arm64, Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), complete strict concurrency, explicit default `nonisolated` isolation, and no extra feature flags. The example compiled and ran in Swift 6 and Swift 5 language modes on that compiler. Separate local checks used continuation gates to hold cleanup pending: neither successful return nor error delivery proceeded until cleanup was released. Nested scope/order checks passed; cancelled tasks still entered cleanup with cancellation visible, and cancellation-sensitive sleep threw as expected. A `Task` launched inside `defer` allowed the enclosing operation to return while cleanup remained pending. Synchronous enclosing functions and unhandled cleanup throws were rejected.

These checks establish local control flow, not production resource release. Older compilers, other platforms, and real file/network services were not tested. APIs used by actual cleanup retain their own OS requirements.

Cancellation-shield sources and SDK availability rechecked 2026-10-25. The added helper passed Swift 5/6 typechecking with the configuration above on macOS 27 and iOS 27 targets; unguarded use on macOS 26 was rejected. Local macOS 27.0.1 checks covered cancellation before and during a shield, static versus handle status, inherited child cancellation, error exit restoring visibility, and awaited deferred release. Cancellation-handler and explicit group-cancellation behavior above follows SE-0504 and was not exercised locally. Resource release, latency bounds, older runtimes, and iOS execution remain unverified.

## Sources

- [Defer in Swift explained with Code Examples](https://www.avanderlee.com/swift/defer-usage-swift/) (updated 2026-07-06)
- [Swift 6.4: What's New in Concurrency](https://www.avanderlee.com/concurrency/swift-6-4-whats-new-in-concurrency/) (2026-06-21)
