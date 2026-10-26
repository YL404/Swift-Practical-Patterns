# Type-directed Swift macro expansions

Use this when a macro needs a property constraint diagnostic, a generated signature for an inferred field, or tests of compiler-enforced contracts. Prefer an explicit annotation when it keeps a small macro or a public API simpler. These techniques let the **compiler** select or infer types; the macro implementation still receives syntax, not a general type-checker query API. [SE-0382](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md#type-checked-macro-arguments-and-results)

Attached macros require a Swift 5.9-capable toolchain. The nested-protocol form below additionally requires Swift 5.10 and a non-generic context; an OS availability check cannot add a compiler feature. Match SwiftSyntax to the macro target's toolchain. [SE-0389](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0389-attached-macros.md), [SE-0404](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0404-nested-protocols.md)

## Select a diagnostic with macro overloads

Give a helper macro a constrained overload and an unconstrained fallback. Macro arguments are checked before expansion; the compiler selects the implementation. The following declarations are for a library exposing an existing macro plugin named `ValidationMacros`:

```swift
@attached(peer)
public macro RequireEquatable<T: Equatable>(type: T.Type) =
    #externalMacro(module: "ValidationMacros", type: "Accept")
@attached(peer)
public macro RequireEquatable<T>(type: T.Type) =
    #externalMacro(module: "ValidationMacros", type: "Reject")
@attached(peer)
public macro RequireEquatable<T: Equatable>(value: T) =
    #externalMacro(module: "ValidationMacros", type: "Accept")
@attached(peer)
public macro RequireEquatable<T>(value: T) =
    #externalMacro(module: "ValidationMacros", type: "Reject")
```

These are the corresponding implementation types inside that plugin target; register both with the target's `CompilerPlugin`. This fragment omits package setup and registration, and is not a complete macro package.

```swift
import SwiftSyntax
import SwiftSyntaxMacros

public enum Accept: PeerMacro {
    public static func expansion(
        of node: AttributeSyntax,
        providingPeersOf declaration: some DeclSyntaxProtocol,
        in context: some MacroExpansionContext
    ) throws -> [DeclSyntax] { [] }
}

public enum Reject: PeerMacro {
    public static func expansion(
        of node: AttributeSyntax,
        providingPeersOf declaration: some DeclSyntaxProtocol,
        in context: some MacroExpansionContext
    ) throws -> [DeclSyntax] {
        throw MacroExpansionErrorMessage(
            "The field's static type must conform to Equatable"
        )
    }
}
```

Use the type form for an annotated field, for example `@RequireEquatable(type: Int.self)`. For an inferred field, pass its initializer expression to the value form, such as `@RequireEquatable(value: "Untitled")`. Distinct argument labels keep metatypes out of value-overload selection. The empty peer expansion does not evaluate that argument at runtime.

A type-level macro can use the **member-attribute** role to attach these helpers to supported stored properties. It does not gain that role merely by being an extension macro. Select the helper from syntax: an annotation supplies the metatype expression; otherwise the initializer supplies the value expression. Handle multiple bindings, computed/static properties, wrappers, missing initializers, and contextual expressions explicitly rather than claiming general property support. [MemberAttributeMacro](https://github.com/swiftlang/swift-syntax/blob/main/Sources/SwiftSyntaxMacros/MacroProtocols/MemberAttributeMacro.swift), [SE-0389 composition](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0389-attached-macros.md#member-attribute-macros)

This checks constraints visible at the expansion site. An unconstrained generic `T` takes the fallback even if a particular caller later uses `Int`; provide the intended constraint or change the macro's contract. Do not add independent `Equatable` and `AnyObject` branches without resolving their overlap: an `Equatable` class made the tested overload set ambiguous. Equality by identity is a domain decision, not a general replacement for value equality.

## Name an inferred field type through a witness

When generated code needs a type spelling, express a protocol requirement whose associated type can be inferred from a concrete witness. Refer to that witness's associated type in the generated signature. This uses ordinary associated-type inference; it does not obtain an inferred type string inside SwiftSyntax. [Swift generics](https://github.com/swiftlang/swift-book/blob/main/TSPL.docc/LanguageGuide/Generics.md#associated-types)

Complete expanded-code example, written without a macro dependency:

```swift
import Foundation

public struct Entry {
    public var timestamp = Date.now

    public protocol FieldTypes {
        associatedtype Timestamp
        static var seed: Timestamp { get }
    }

    public enum FieldWitness: FieldTypes {
        public static let seed = Date.now
    }

    public init() {}

    public func replacingTimestamp(_ value: FieldWitness.Timestamp) -> Self {
        var copy = self
        copy.timestamp = value
        return copy
    }
}
```

Only the associated type is used by `replacingTimestamp`; it does not read `seed`. Stored type properties initialize lazily on first access. Reading a static witness as an initializer default would instead share its stored value, which is wrong for defaults intended to run afresh, such as a timestamp or identifier. Preserve the original default's evaluation semantics when generating an initializer. [Swift type properties](https://github.com/swiftlang/swift-book/blob/main/TSPL.docc/LanguageGuide/Properties.md#type-properties)

Keep these limits beside the generator:

- A nested protocol cannot be emitted inside a generic context under SE-0404. Require an annotation or design and validate a different representation for that case.
- Static witness storage still obeys isolation and Sendable rules. A nonisolated static `let` holding a non-Sendable reference was rejected in Swift 6 mode; do not add unsafe isolation annotations just to obtain a type projection. [SE-0412](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0412-strict-concurrency-for-global-variables.md)
- Public signatures cannot depend on private helper types. The example intentionally makes the helpers public; that expands the exposed API. Validate access levels and name collisions, or keep the field's explicit type in the public API. [Swift access control](https://github.com/swiftlang/swift-book/blob/main/TSPL.docc/LanguageGuide/AccessControl.md#guiding-principle-of-access-levels)
- Moving an initializer expression into a nested witness can change name lookup, contextual typing, access, and isolation. Do not infer universal support from constructor-shaped expressions, and do not promise support for property wrappers or arbitrary defaults without generator-specific checks.

## Test compiler-enforced contracts

Expansion snapshots check generated syntax and parser/macro diagnostics. They do not establish that overload selection, generated conformances, access control, or isolation satisfy the Swift type checker. For those contracts, also compile client fixtures against the actual library and macro plugin. The inspected [SwiftSyntax assertion](https://github.com/swiftlang/swift-syntax/blob/c161c39b9687945370b430708e8a3795b707291a/Sources/SwiftSyntaxMacrosGenericTestSupport/Assertions.swift) parses and expands syntax; the compiler separately checks macro arguments and results under [SE-0382](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md#type-checked-macro-arguments-and-results).

Keep deliberately invalid code in snippet strings passed to a compiler-backed harness, or in separately compiled fixture files, so it does not prevent the normal test target from building. Pair a supported use with a rejected use. Check the intended diagnostic as well as compilation status: a missing import or unloaded plugin is not proof that the library rejected the misuse. When recording diagnostic snapshots, review the output and rerun against the approved expectation.

Record the compiler, SDK, target, language mode, strict-concurrency level, default isolation, and relevant feature flags. A successful compiler exit can still emit warnings; require empty diagnostics only when that is the intended contract. Swift 5 versus Swift 6 alone does not determine concurrency diagnostics: [global-storage checks](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0412-strict-concurrency-for-global-variables.md) and [default isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md) also matter. Runtime promises, such as preserving default-expression evaluation, still need execution checks.

## Evidence and validation limits

Official sources checked 2026-10-22–23; local validation completed 2026-10-23. Local validation used macOS 27.0.1 arm64, Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), complete strict concurrency, default isolation explicitly `nonisolated`, and no additional upcoming features.

In Swift 6 and Swift 5 language modes on that compiler, a locally built macro plugin selected passing annotated/inferred fields and rejected nonconforming fields; a member-attribute harness attached the helpers to simple stored properties. A side-effect trap confirmed the helper's erased argument was not evaluated at runtime. Constrained versus unconstrained generic checks and the overlapping class constraints behaved as described. The expanded witness example compiled and ran; projections inferred from a factory, a static constant, and an immediately invoked closure worked. A trap confirmed using only a projected type did not evaluate its witness. A separate-module client used the public projection. Nested generic protocols and private helper types in public signatures were rejected. Non-Sendable static storage was an error in Swift 6 and a warning in Swift 5 with complete checking.

These are macro/compiler and local execution checks, not a complete derivation or memberwise macro implementation. Older compilers, other platforms, editor diagnostic placement, property-wrapper inference, arbitrary default relocation, library evolution, compile-time performance, and application behavior were not validated.

Compiler-test guidance checked 2026-10-26 against the linked SwiftSyntax assertion and the [MacroTesting implementation](https://github.com/pointfreeco/swift-macro-testing/blob/47751a42d88ae202b245e111807c2ad6929211c1/Sources/MacroTesting/AssertMacro.swift). Separate `swiftc -typecheck` checks used macOS 27.0.1 arm64, Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), the macOS 27.0 SDK, and an arm64 macOS 27.0 target, with no added feature flags. For a class with mutable instance storage and `static let shared = Model()`, explicit nonisolated default isolation produced no diagnostics in Swift 5 with minimal checking, a warning in Swift 5 with complete checking, and an error in Swift 6 with complete checking. Swift 6 with complete checking and MainActor default isolation produced no diagnostics. These are configuration-sensitive typechecks, not executions of CompilationTesting; its private beta, dependency discovery, plugin loading, snapshot recording, fix-its, and CI behavior were not validated.

## Sources

- [Advanced macro technique #1: Static type information](https://www.pointfree.co/blog/posts/232-advanced-macro-technique-1-static-type-information) (2026-10-07)
- [Advanced macro technique #2: Type inference](https://www.pointfree.co/blog/posts/233-advanced-macro-technique-2-type-inference) (2026-10-08)
- [Beta Preview: CompilationTesting](https://www.pointfree.co/blog/posts/234-beta-preview-compilationtesting) (2026-10-26)
