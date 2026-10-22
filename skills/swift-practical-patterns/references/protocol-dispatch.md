# Protocol requirements and extension method selection

Use this when a protocol-typed value or generic function calls an extension implementation instead of a conforming type's same-named method. The examples use plain Swift instance methods, direct conformances, and unconstrained extensions. Explicit `any P` syntax requires Swift 5.6+; no framework or OS-specific API is involved. [SE-0335](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md)

## Choose the customization point

Put a method in the protocol declaration when callers using that protocol must select each conforming type's implementation. An extension can still supply the fallback body. Keep extension-only methods for shared helpers whose implementation is chosen from the declarations available to the caller. A same-named method on a concrete type does not turn an extension-only helper into a protocol requirement. [Method requirements and protocol extensions](https://github.com/swiftlang/swift-book/blob/main/TSPL.docc/LanguageGuide/Protocols.md#protocol-extensions)

Before changing the protocol, check the receiver's static type, the method's full signature, and which declarations are visible at the call site. The following complete command-line example demonstrates the distinction:

```swift
protocol MessageSource {}

extension MessageSource {
    func message() -> String { "default" }
}

struct LocalMessage: MessageSource {
    func message() -> String { "local" }
}

struct FallbackMessage: MessageSource {}

func throughGeneric<T: MessageSource>(_ source: T) -> String {
    source.message()
}

let concrete = LocalMessage()
let erased: any MessageSource = concrete
precondition(concrete.message() == "local")
precondition(erased.message() == "default")
precondition(throughGeneric(concrete) == "default")
precondition(throughGeneric(FallbackMessage()) == "default")
```

The generic body's constraint exposes the extension method, not `LocalMessage.message()`. Passing a concrete value to that generic function does not change this selection. These results were observed in both unoptimized and optimized builds under the conditions below.

If `message()` is intended to vary by conformance, replace the empty protocol declaration above with:

```swift
protocol MessageSource {
    func message() -> String
}
```

Keep the extension and conforming types. The protocol-typed and generic calls for `LocalMessage` now return `"local"`; update their two preconditions accordingly. `FallbackMessage` still returns `"default"`. The extension body now supplies a default for a requirement, and the concrete implementation satisfies that requirement. [Default implementations](https://github.com/swiftlang/swift-book/blob/main/TSPL.docc/LanguageGuide/Protocols.md#providing-default-implementations)

This is a change to the protocol contract and may change existing clients' behavior. Make it only when per-conformance customization is intended. Recheck inherited class conformances, constrained extensions, overloads, and access or isolation differences separately; this example does not establish their selection rules.

## Separate semantics from generated calls

Witness tables support protocol-requirement calls for value types as well as reference types; a `struct` receiver does not by itself prove that a protocol call has no indirection. Unspecialized generic code can also use witnesses. [Apple's explanation of protocol and generic dispatch](https://developer.apple.com/videos/play/wwdc2016/416/)

Conversely, specialization and devirtualization can remove indirect calls without changing which implementation the program must select. `final` rules out class overriding, while access control and whole-module visibility can enable optimization; none is a repair for an extension-only method that needs to be a protocol requirement. Preserve required customization and inspect the actual build when investigating overhead. [Swift optimization guidance](https://github.com/swiftlang/swift/blob/main/docs/OptimizationTips.rst#reducing-dynamic-dispatch), [generic specialization](https://github.com/swiftlang/swift/blob/main/docs/OptimizationTips.rst#generics)

## Evidence and validation limits

Official sources checked 2026-10-22.

Compiled and executed the complete example and its requirement-based alternative on macOS 27.0.1 arm64, Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1). Both Swift 5 and Swift 6 language modes used complete strict concurrency, explicitly nonisolated default isolation, and no additional feature flags. Each version ran with `-Onone` and with `-O -whole-module-optimization`. All checked concrete, existential, generic, and fallback results matched the expectations above. Unoptimized SIL generation showed a direct reference to the extension method in the original generic body and a witness-method lookup after adding the requirement. A direct-conforming final class was also checked through a concrete value, existential, and generic call in Swift 6 mode.

These checks establish example behavior on that toolchain. No older compiler, other platform, Objective-C/C++ interoperability, cross-module or library-evolution build, inherited class conformance, or constrained extension was tested. No timing benchmark, Instruments profile, or claim about optimized machine-code shape is provided.

## Sources

- [Method dispatch mechanisms in Swift: static and dynamic dispatch](https://nilcoalescing.com/blog/MethodDispatchMechanismsInSwift/) (2025-03-28)
