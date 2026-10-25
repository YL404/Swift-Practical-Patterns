# Immutable weak references and explicit Sendable suppression

Use this when a fixed weak delegate prevents checked `Sendable` conformance, or a library intentionally suppresses inferred sendability. These declarations express different contracts; neither makes mutable shared state safe by itself.

## Keep a fixed delegate weak

[SE-0481](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0481-weak-let.md), implemented in Swift 6.3, permits optional `weak let` storage. Code cannot replace the reference after initialization, but reads become `nil` when the target deallocates. The target must still meet the holder's sendability requirements.

Complete definitions:

```swift
protocol SelectionDelegate: AnyObject, Sendable {
    func didSelect()
}

final class SelectionSource: Sendable {
    weak let delegate: (any SelectionDelegate)?

    init(delegate: (any SelectionDelegate)?) {
        self.delegate = delegate
    }
}
```

If replacing the delegate is a real requirement, retain mutable storage and protect its accesses using appropriate isolation or synchronization. Weak references do not retain their target, so another owner must keep it alive. The immutable weak-capture change is distinct: SE-0481 lists `ImmutableWeakCaptures` for migration. Check language mode and feature settings before rewriting code that assigns to a `[weak x]` capture.

## State an intentional lack of conformance

[SE-0518](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0518-tilde-sendable.md), implemented in Swift 6.4, allows `~Sendable` on a struct, enum, or class declaration. It suppresses inference and documents an audited choice:

```swift
struct LocalToken: ~Sendable {
    let value: Int
}
```

It is not a generic negative constraint, an actor annotation, or a ban on every isolation transfer. A `T: Sendable` requirement rejects this type; region-based transfer is a separate question. Do not add suppression in an extension or combine it with unconditional `Sendable` conformance. Conditional conformances remain possible. Unlike an unavailable `Sendable` conformance on a base class, suppression does not prohibit a properly audited subclass from conforming. Removing an existing public `Sendable` conformance is a source-breaking API change.

## Evidence and limits

Proposals checked 2026-10-25. Definitions typechecked with warnings as errors in Swift 5/6 modes, complete strict concurrency, nonisolated default isolation, and no extra feature flags using Xcode 27.1 (27A9275), Apple Swift 6.4 on macOS 27.0.1 arm64. Local execution checked weak zeroing after target release. Negative Swift 6 checks rejected `weak var` in a checked Sendable holder, reassignment of `weak let`, a non-Sendable delegate in that holder, suppression in an extension, and using `LocalToken` for `T: Sendable`. Older compilers and weak-capture migration behavior were not tested; stored `weak let` acceptance here does not establish capture behavior in every language mode.

## Sources

- [Swift 6.4: What's New in Concurrency](https://www.avanderlee.com/concurrency/swift-6-4-whats-new-in-concurrency/) (2026-06-21)
