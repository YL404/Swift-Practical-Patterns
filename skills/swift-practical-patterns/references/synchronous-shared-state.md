# Synchronous shared state with Mutex and Atomic

Use this for a shared cache that must expose synchronous operations, or an independent numeric metric updated by concurrent callers. These examples require a Swift 6 toolchain and `Synchronization`: iOS 18+, macOS 15+, Mac Catalyst 18+, tvOS 18+, watchOS 11+, or visionOS 2+. Check the deployment target before choosing them; importing the module does not back-deploy its types. An actor remains a good choice when callers can await its operations.

## Protect the complete cache operation

[`Mutex`](https://developer.apple.com/documentation/synchronization/mutex) couples state with a non-recursive lock. Contending callers block their threads. Keep the stored mutex private and use `let`; the state inside it remains mutable. A final class whose only storage is a mutex can declare checked `Sendable` conformance, because the mutex is Sendable even when its protected state is not ([SE-0433](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0433-mutex.md)).

Complete type declaration; the cache stores already-computed integers and does not load resources:

```swift
import Synchronization

final class IntegerCache: Sendable {
    private let entries = Mutex<[String: Int]>([:])

    func value(for key: String) -> Int? {
        entries.withLock { $0[key] }
    }

    func insertIfAbsent(_ value: Int, for key: String) -> Bool {
        entries.withLock { entries in
            guard entries[key] == nil else { return false }
            entries[key] = value
            return true
        }
    }
}
```

The check and insertion share one critical section. Separate locked getters and setters would leave a gap in which another caller could insert. This protects insertion, not a download or computation performed before the call; deduplicating in-flight asynchronous work needs a separate design.

Keep critical sections short and synchronous. [`withLock`](https://developer.apple.com/documentation/synchronization/mutex/withlock(_:)) accepts a synchronous, throwing closure and releases the lock on exit, including a throw. It does not roll back changes made before that throw. Do not `await` inside it or block waiting for a task to finish. Never call another method that reacquires the same mutex from inside its closure: depending on the platform, recursion may trap or deadlock. Avoid arbitrary callbacks under the lock because they can reenter the owner. Mutex provides no fairness guarantee ([SE-0433](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0433-mutex.md)).

Return Sendable snapshots or results rather than an alias to mutable protected state. For non-Sendable state, preserve the `sending` boundary on initialization and access; do not bypass its diagnostics with unsafe annotations. A mutex does not protect accesses to an object through aliases outside that boundary.

## Count independent events with an atomic update

An atomic load followed by a store is still two operations; concurrent increments can overwrite each other even with sequentially consistent ordering. Use a read-modify-write operation such as `wrappingAdd`. Its overflow policy is intentional: the following unsigned diagnostic count wraps modulo 2⁶⁴. It is unsuitable for balances, capacity enforcement, or totals that must never wrap.

Complete type declaration, independent of the cache above:

```swift
import Synchronization

final class EventCounter: Sendable {
    private let count = Atomic<UInt64>(0)

    func record() {
        _ = count.wrappingAdd(1, ordering: .relaxed)
    }

    var snapshot: UInt64 {
        count.load(ordering: .relaxed)
    }
}
```

[`.relaxed`](https://developer.apple.com/documentation/synchronization/atomicupdateordering/relaxed) preserves the atomicity of that operation but does not order accesses to other variables. Use this example only when the count itself is the information being shared. Seeing a new count does not establish that a separate payload is ready; do not use it as a publication flag. Prefer a mutex or actor for related state unless the application has a separately justified synchronization protocol.

Keep `Atomic` storage in a `let` binding and supply literal orderings at call sites. [SE-0410](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0410-atomics.md) specifies its storage restrictions, compile-time ordering arguments, and read-modify-write operations. Separate atomics do not automatically preserve an invariant spanning multiple fields.

## Evidence and limits

Official sources checked 2026-10-22. The article's benchmarks do not establish a general speed ranking: its driver awaits an async operation even for synchronous primitives, and does not report enough build/hardware conditions for a matched performance conclusion. Measure the actual workload before replacing an actor for speed.

Validated on macOS 27.0.1 arm64, Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), complete strict concurrency, explicit default `nonisolated` isolation, and no additional feature flags. Both declarations compiled and ran with a macOS 15 deployment target in Swift 6 and Swift 5 language modes on that compiler. Eight child tasks each attempted 10,000 inserts of the same key and recorded 10,000 events: exactly one insertion succeeded and the final count was 80,000. Separate checks confirmed lock release without rollback on a throw and unsigned wrapping overflow; Swift 6 rejected `await` inside `withLock` and `var` atomic storage.

Both declarations also passed Swift 6 typechecking for an iOS 18 target with the iPhoneOS 27.1 SDK.

These are local correctness checks, not a performance comparison or an exhaustive interleaving proof. Older compilers/runtimes, other platforms, real cache workloads, and cross-variable publication were not executed; recursive locking was not deliberately triggered.

## Sources

- [The Synchronization Framework in Swift 6](https://blog.jacobstechtavern.com/p/the-synchronisation-framework) (2024-12-30)
