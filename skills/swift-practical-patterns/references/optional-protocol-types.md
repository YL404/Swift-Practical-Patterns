# Optional opaque and existential types

Use this when diagnosing optional `some` / `any` syntax, including `some View?`, delegate properties, and protocol compositions. The shorter spelling requires a **Swift 6.4 compiler**; the language mode is a separate setting. This syntax change introduces no new OS requirement. A runtime `#available` check cannot make an older compiler parse it.

## Choose a spelling for the supported toolchains

[SE-0521, implemented in Swift 6.4](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0521-improved-optional-opaque-and-any.md) makes `some P?` equivalent to `(some P)?`, and `any P?` equivalent to `(any P)?`. Repeated optionality such as `any P??` follows the same rule. The equivalent `!` spelling is accepted only where implicitly unwrapped optionals are already permitted.

Protocol compositions still need grouping: use `any (P & Q)?` or `(any P & Q)?`, and similarly for `some`. Bare `any P & Q?` and `some P & Q?` remain errors. Keep the parentheses around the whole opaque/existential type when sharing source with older compilers. Existing parenthesized code needs no migration; SE-0521 specifies identical types and manglings.

Optional opaque results themselves predate this shorthand: [SE-0328, implemented in Swift 5.7](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md), introduced structural opaque results such as `(some P)?`. Its mandatory-parentheses rule is superseded by SE-0521 on Swift 6.4.

## Supply enough information to infer the opaque type

Complete command-line example for Swift 6.4:

```swift
protocol Labelled {
    var title: String { get }
}

struct Badge: Labelled {
    let title: String
}

func makeBadge(_ visible: Bool) -> some Labelled? {
    visible ? Badge(title: "Verified") : nil
}

assert(makeBadge(false) == nil)
assert(makeBadge(true)?.title == "Verified")
```

The conditional expression supplies the concrete optional type. In local Swift 6.4 checks, replacing the body with `guard visible else { return nil }` followed by `return Badge(title: "Verified")` failed to infer the opaque type at the bare `nil`. Both return-type spellings failed. Keeping that control flow and writing `return nil as Badge?` passed. Diagnose this inference failure separately from optional syntax; removing or adding parentheses does not fix it.

An opaque result still needs a consistent underlying type for each generic substitution. Returning two unrelated conforming types from separate branches remains invalid. Switching to `any P?` changes the type contract; choose it only when existential storage or varying concrete types are intended. [Swift language guide: opaque and boxed protocol types](https://github.com/swiftlang/swift-book/blob/main/TSPL.docc/LanguageGuide/OpaqueTypes.md#returning-an-opaque-type)

## Evidence and validation limits

Official sources checked 2026-10-22. The source's SwiftUI guard/nil body failed macOS typechecking with either return-type spelling; replacing it with a single conditional expression passed. The shorter spelling alone does not resolve the inference failure.

Validated on macOS 27.0.1 arm64, Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), complete strict concurrency, explicit default `nonisolated` isolation, and no additional feature flags. The example compiled and ran in Swift 6 and Swift 5 language modes on that compiler. Additional Swift 6 checks accepted both spellings, grouped compositions, nested optionals, an existing IUO property, `any ~Copyable?` in noncopyable storage, and a class-bound weak existential delegate; releasing its owner cleared the weak reference. Bare compositions, incompatible opaque return branches, and an untyped nil-only result were rejected.

These are compiler and local execution checks. Older compilers, other platforms, and SwiftUI rendering/lifecycle were not tested. The compiler feature does not establish runtime availability of APIs used inside a real view.

## Sources

- [some View? without parentheses in Swift 6.4](https://sarunw.com/posts/optional-opaque-and-existential-types/) (2026-10-16)
