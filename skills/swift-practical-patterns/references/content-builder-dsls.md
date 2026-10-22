# ContentBuilder for non-View DSLs

Use this reference when deciding whether to reuse SwiftUI's builder for an app-local rule table or other non-View composition. It is not a general replacement for a domain-owned result builder.

## Choose the dependency and deployment boundary

With the Xcode 27 SDK, `ContentBuilder` aliases `ViewBuilder`; changing the attribute name alone does not change compilation or runtime behavior. Apple documents structural assembly followed by conditional protocol conformance, with shared container initializers replacing domain-specific overload sets. This addresses type-checking work in nested builders; it is not a rendering or runtime speed guarantee. See [ContentBuilder](https://developer.apple.com/documentation/swiftui/contentbuilder) and [TN3211](https://developer.apple.com/documentation/technotes/tn3211-resolving-swiftui-source-incompatibilities-for-state-and-contentbuilder).

For a custom DSL, prefer one structural construction path when domains share the same grammar, then require the relevant capability on the resulting container. Keep constraints that are needed to disambiguate expressions or produce useful diagnostics. This is an API design recommendation, not a guarantee about every constraint solver workload.

Distinguish the SDK that recognizes the declarations from the runtime that supports the selected overload:

- The alias is an SDK 27 spelling that can be used with older deployment targets. Earlier SDKs recognize `ViewBuilder`, but do not gain these unconstrained declarations by changing the spelling.
- `TupleContent` itself is available on iOS/macOS 26, but the stock unconstrained, multi-value `buildBlock` that produces it requires iOS/macOS 27. Do not infer the latter's availability from the alias or container. These are the Xcode 27.1 SDK declarations; [TupleContent](https://developer.apple.com/documentation/swiftui/tuplecontent) documents its own availability and intended use for custom DSL protocol conformances.
- The domain-owned pack container below supports an iOS 17/macOS 14 deployment target with Xcode 27. Parameter-pack types need runtime support ([SE-0398](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0398-variadic-types.md)); pack iteration is a Swift 6 compiler feature with no additional deployment requirement ([SE-0408](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0408-pack-iteration.md)). Compilation in Swift 5 language mode with this newer compiler is also possible.

Use a domain-owned `@resultBuilder` when the module must avoid SwiftUI, needs a different grammar, or would otherwise accumulate framework extensions and storage dependencies. Keep the protocol and conformances internal for the app-local approach; exporting this as a reusable library requires a separate API design review.

## Give the structural outputs domain behavior

The common definitions and either deployment path below form a complete, non-UI example when assembled as directed. They implement an ordered lookup, not URL parsing or navigation. The common definitions support a leaf, an optional rule, and dynamic entries. `Entry.key` must be unique within the `ForEach` input and remain its stable identity. For data without meaningful identity, use a domain-owned collection container instead.

```swift
import SwiftUI

protocol LookupRule {
    func value(for key: String) -> Int?
}

struct Entry: LookupRule, Identifiable {
    let key: String
    let number: Int
    var id: String { key }

    func value(for key: String) -> Int? {
        self.key == key ? number : nil
    }
}

extension Optional: LookupRule where Wrapped: LookupRule {
    func value(for key: String) -> Int? {
        self?.value(for: key)
    }
}

extension ForEach: LookupRule where Content: LookupRule {
    func value(for key: String) -> Int? {
        for element in data {
            if let result = content(element).value(for: key) {
                return result
            }
        }
        return nil
    }
}
```

For the stock iOS/macOS 27 path, add this snippet. The tuple traversal implements declaration order and first-match selection; the builder itself does not supply those domain semantics. `ForEach`'s closure is evaluated during lookup, so keep rule construction free of effects. Pack members have already been constructed before the lookup starts; returning early skips later lookups, not earlier construction.

```swift
@available(iOS 26.0, macOS 26.0, *)
extension TupleContent: LookupRule where repeat each Content: LookupRule {
    func value(for key: String) -> Int? {
        for rule in repeat each content {
            if let result = rule.value(for: key) {
                return result
            }
        }
        return nil
    }
}

@available(iOS 27.0, macOS 27.0, *)
@ContentBuilder
func makeLookup(includePreview: Bool, entries: [Entry]) -> some LookupRule {
    Entry(key: "home", number: 1)
    if includePreview {
        Entry(key: "preview", number: 2)
    }
    ForEach(entries) { $0 }
}
```

[`TupleContent.content`](https://developer.apple.com/documentation/swiftui/tuplecontent/content) and [`ForEach`](https://developer.apple.com/documentation/swiftui/foreach) expose the values used here. `some LookupRule` preserves one concrete composite type; flags change its stored values, not that return type. The composite acquires the local protocol only through the conformances above.

## Back-deploy the multi-value block deliberately

For an iOS 17/macOS 14 target, replace the `TupleContent` extension with the following snippet, and remove the OS 27 availability annotation from `makeLookup`. Keep the common definitions, `@ContentBuilder`, and the function body. This is a module-level overload on the underlying `ViewBuilder`, visible through either alias; it is not back-deployment supplied by the stock multi-value overload.

```swift
struct RuleGroup<each Rule: LookupRule>: LookupRule {
    let rules: (repeat each Rule)

    func value(for key: String) -> Int? {
        for rule in repeat each rules {
            if let result = rule.value(for: key) {
                return result
            }
        }
        return nil
    }
}

extension ContentBuilder {
    static func buildBlock<each Rule: LookupRule>(
        _ rules: repeat each Rule
    ) -> RuleGroup<repeat each Rule> {
        RuleGroup(rules: (repeat each rules))
    }
}
```

On the verified compiler, this constrained overload also wraps single-statement branches in `RuleGroup`. Review other builder overloads in the importing module when adopting it; avoid adding overlapping domain overloads merely to preserve this spelling. A dedicated builder keeps that overload surface under your control.

## Check the grammar before adopting it

- A bare `if` needs the `Optional` conformance. `ForEach` is an ordinary container value; it does not add support for a raw `for` loop. Stock `ViewBuilder` has no `buildArray`; a custom builder can implement it ([SE-0289](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0289-result-builders.md)).
- `if/else` and `switch` produce `_ConditionalContent`. The examples intentionally do not traverse its underscored storage. Prefer a domain-owned builder and conditional type when those constructs are required, rather than extending this example with an unverified storage contract.
- Empty-block resolution depends on the expected result and the available overloads. In the verified `some LookupRule` context, the compiler selects the zero-length pack overload: `TupleContent` on the stock path and `RuleGroup` on the fallback path, each returning `nil`. This does not give `EmptyContent` a `LookupRule` conformance. If imports or other overloads make an empty block ambiguous, supply an explicit domain empty value instead of assuming it means `EmptyView`.
- `if #available` invokes `buildLimitedAvailability`. The Xcode 27.1 stock overloads require framework content conformances that these rules do not have. Check availability outside the builder and call separately gated functions, or design a dedicated availability operation; do not assume a bare-`if` conformance covers this case.
- Extending the tuple does not make every SwiftUI container traversable. Do not assume `Group` or `Section` offers equivalent public child storage.

For ordinary SwiftUI ambiguity after upgrading SDKs, use the targeted fixes in [TN3211](https://developer.apple.com/documentation/technotes/tn3211-resolving-swiftui-source-incompatibilities-for-state-and-contentbuilder); this DSL technique is not a reason to rewrite working view builders.

## Verification scope

Verified 2026-10-23 against Apple documentation and Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1). The exact snippets, assembled separately for each path, compiled in Swift 5 and Swift 6 language modes with complete concurrency checking and nonisolated default isolation. Both ran on macOS 27.0.1 arm64, including enabled/disabled optional rules, empty dynamic data, misses, duplicate-key precedence across groups, zero/single-element packs, and lookup short-circuiting. The stock path targeted macOS 27; the fallback targeted macOS 14. Both modes also passed iOS typechecking for targets 27 and 17 respectively.

Swift 6 negative checks confirmed rejection of raw loops, `if/else` without the conditional conformance, and in-block availability for these rules on both paths. An ungated stock multi-value block failed iOS 26 typechecking; the fallback failed on iOS 16 because parameter packs in generic types require iOS 17. The fallback's single-statement wrapping and both zero-pack results were inspected during local execution. Older runtimes, other platforms, full URL routing/navigation, public-library integration, UI rendering, and compile-time performance were not tested. Recheck overload selection against the target SDK and the actual module's imports and extensions.

## Sources

- https://fatbobman.com/en/posts/contentbuilder-explained/
- https://artemnovichkov.com/blog/using-swiftui-contentbuilder-with-non-view-types
