# Struct memberwise initializers and private storage

Use this for ordinary struct initialization and Swift 6.4 migrations. Check the compiler version separately from the Swift language mode. This is a compiler feature with no new OS deployment requirement; `#available` cannot enable it on an older compiler. Property wrappers, macros, and `init` accessors need their own expansion and initialization analysis.

## Decide whether to write an initializer

- Keep synthesis when member-by-member construction expresses the intended contract. An initialized `var` can supply a default argument; an initialized `let` is omitted rather than made overridable. [SE-0242, implemented in Swift 5.1](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0242-default-values-memberwise.md)
- Write an initializer for validation, derived values, or deliberately chosen access. A public struct still needs an explicit `public init` for construction from another module. [Swift access control](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/#Default-Memberwise-Initializers-for-Structure-Types)
- An initializer in the struct body suppresses memberwise synthesis. A distinct initializer in an extension can preserve it; retaining synthesis is inappropriate if it bypasses required validation. [Swift initialization](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/#Initializer-Delegation-for-Value-Types)

## Apply the Swift 6.4 exclusion rule

[SE-0502](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0502-exclude-private-from-memberwise-init.md) starts with the most accessible memberwise-initializable property, capped at `internal`. Unannotated properties account for their enclosing type's access. It excludes a property only when it is less accessible than that maximum **and** has an initial value, including an implicitly nil-initialized optional `var`.

Consequently:

- A private property without an initial value remains a parameter and still restricts access.
- An all-private set of eligible properties is not blanket-excluded.
- `private(set)` restricts the setter; it does not make an otherwise internal property private for this rule.

Complete command-line example for Swift 6.4:

```swift
struct Reading {
    let title: String
    var readCount = 0
    private var cachedSummary: String?
    private var cacheHits = 0
}

let first = Reading(title: "Swift")
let second = Reading(title: "Swift", readCount: 3)
assert(first.readCount == 0 && second.readCount == 3)
```

Both private fields keep their initial values without reducing the new initializer's access. With a compiler predating this change, write the intended initializer explicitly instead of making implementation storage public. Do not add a meaningless default merely to expose construction.

## Preserve required calls during migration

SE-0502 retains a compatibility overload for old signatures within their permitted access scope. Do not base a new API on its indefinite availability.

Before replacing it, inspect all construction paths. Adding only an explicit **private** replacement in the struct body also suppresses the newly usable memberwise initializer. If callers need both signatures, provide both explicitly. Moving an initializer to an extension preserves synthesis, but a signature matching the new synthesized initializer causes a redeclaration error. An untyped `let make = Reading.init` can also select the new signature; check function references as well as direct calls. [SE-0502 compatibility details](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0502-exclude-private-from-memberwise-init.md#source-compatibility)

## Evidence and validation limits

Official sources checked 2026-10-22. The Swift book's general claim that any private stored property makes the initializer private has not incorporated SE-0502; use the versioned proposal for Swift 6.4's exclusion behavior, and the book for the unchanged public-API and custom-initializer rules.

Validated on macOS 27.0.1 arm64, Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), Swift 6 mode, complete strict concurrency, default isolation explicitly `nonisolated`, and no additional feature flags. The example compiled and ran in Swift 6 and Swift 5 language modes on that compiler. Additional local checks covered explicit/nil private defaults, required private fields, retained all-private parameters, a fileprivate type, `private(set)`, initialized `let`, compatibility calls, initializer function references, and distinct/colliding extension initializers. Adding only a private initializer removed the caller's construction path; explicitly supplying both restored it. Separate-module clients were rejected for a synthesized internal initializer and accepted after adding `public init`.

These are compiler and local execution checks. Older compilers, other platforms, property-wrapper/macro expansions, and future language modes were not tested. Do not use this example to infer SwiftUI state lifetime or macro-specific initializer availability.

## Sources

- [Memberwise Initializer in Swift explained with Code Examples](https://www.avanderlee.com/swift/memberwise-initializers/) (2026-06-29)
