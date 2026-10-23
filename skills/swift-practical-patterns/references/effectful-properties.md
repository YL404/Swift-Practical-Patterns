# Async and throwing computed properties

Use this when a value-like query can suspend or fail, or when a protocol getter must accommodate asynchronous implementations. Prefer a method when callers supply arguments or request an action. Document nonconstant computed-property complexity; property syntax does not imply cheap access. [Swift API design guidelines](https://www.swift.org/documentation/api-design-guidelines/#general-conventions)

## Declare effects on the getter

Swift 5.5 introduced `get async`, `get throws`, and `get async throws` for **read-only** computed properties and subscripts. Put effects after `get`; callers use `await`, `try`, or `try await` according to the declaration. Adding effects to an existing public getter changes its callers' source requirements. [SE-0310](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0310-effectful-readonly-properties.md)

The following is a complete model with an injected validation operation, not a network client or a filesystem-safety policy. The `throws(FilenameError)` spelling requires Swift 6.0 or later; on Swift 5.5–5.10, use plain `throws` and explicit `FilenameError` case names. Typed throws constrains errors escaping the getter; it does not convert other failures into that type. [SE-0413](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md)

```swift
enum FilenameError: Error {
    case missingExtension
    case rejected
}

struct Filename {
    let stem: String
    let pathExtension: String
    let validate: @Sendable (String) async -> Bool

    var checked: String {
        get async throws(FilenameError) {
            guard !pathExtension.isEmpty else {
                throw .missingExtension
            }
            let candidate = stem + "." + pathExtension
            guard await validate(candidate) else {
                throw .rejected
            }
            return candidate
        }
    }
}

func example() async throws -> String {
    let filename = Filename(stem: "report", pathExtension: "pdf") { name in
        name != "reserved.pdf"
    }
    return try await filename.checked
}
```

Every read with a nonempty extension invokes validation again; the getter does not cache or share a result. The injected validator is nonthrowing; a production operation that throws cancellation or transport errors needs an explicit error contract. If multiple callers should share one operation, explicitly design its ownership and invalidation; see [shared-async-work.md](shared-async-work.md). `async` permits suspension, not guaranteed background execution. Confirm isolation and project settings separately. [SE-0296](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md), [SE-0461](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)

## Preserve synchronous API boundaries

- An effectful getter cannot also declare `set`. Model writes as explicit methods when they suspend or fail.
- Key paths cannot refer to effectful properties. Pass an async/throwing closure to an API designed for those effects, or load a stored snapshot for a synchronous consumer.
- A synchronous UI getter or a synchronous `Binding` closure cannot await this property. Load it from an async lifecycle or event handler and expose separately owned state, including loading/error behavior. Do not conceal the operation in a fire-and-forget getter that returns before it completes.

The read-only and key-path restrictions are language constraints from [SE-0310](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0310-effectful-readonly-properties.md); the snapshot and explicit-write choices are implementation advice.

## Let protocol implementations have fewer effects

A protocol may permit both effects even when a concrete implementation needs neither. Requirements bound what a witness may do; they do not force every witness to suspend or throw. An async getter cannot satisfy a synchronous requirement. [SE-0310 protocol conformance](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0310-effectful-readonly-properties.md#protocol-conformance)

This complete, independent example distinguishes concrete access from access through a protocol requirement:

```swift
protocol NameSource {
    var name: String { get async throws }
}

struct FixedName: NameSource {
    let name: String
}

func readName<Source: NameSource>(_ source: Source) async throws -> String {
    try await source.name
}

func concreteName() -> String {
    FixedName(name: "Local").name
}
```

## Verification scope

Verified on 2026-10-23 against the linked Swift sources. With Xcode 27.1 / Apple Swift 6.4, the exact examples passed macOS typechecking in Swift 5 and 6 modes with nonisolated and MainActor defaults, a separate Swift 6 check with `NonisolatedNonsendingByDefault`, and iOS 27 typechecking using SDK 27.1. The untyped-throws variant passed Swift 5 mode on that same compiler; historical toolchains were not run. Local execution on arm64 macOS 27.0.1 checked successful/rejected/missing-extension reads, repeated validation, and all four combinations of protocol-witness effects. Five negative typechecks confirmed setter, async/throwing key-path, synchronous-requirement, and wrong-error-type diagnostics. No network, filesystem, UI lifecycle, device, or performance behavior was tested.

## Sources

- https://www.avanderlee.com/swift/computed-property/
