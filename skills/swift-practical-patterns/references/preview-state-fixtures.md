# Preview input states and partially generated content

Use this reference when a component needs a selectable catalog of input states, or when a Foundation Models view must display unfinished output without making a model request. Ordinary mock injection and `@Previewable` are separate concerns; keep using the project's existing preview setup for those.

## Select a preview mechanism

With an Xcode 27 SDK, [`#Preview(arguments:)`](https://developer.apple.com/documentation/swiftui/preview(_:traits:arguments:body:)) maps an array of inputs to a group of previews. It requires iOS/iPadOS/Mac Catalyst 26, macOS 27, tvOS 26, watchOS 26, or visionOS 26. The generic argument has no `Identifiable`, `Hashable`, `Equatable`, or `CaseIterable` constraint. Apple's [Xcode 27 release notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27-release-notes) describe the Canvas grid and opening a variant in Interactive mode.

- Use arguments when several inputs share the same rendering setup. A stack of sample views is one composed preview; it does not give each child its own preview variant.
- Keep a small set of inputs that reveal meaningful differences. Test device, locale, color scheme, and Dynamic Type separately when combining them with every state would obscure the review.
- Arguments supply inputs. If the view needs mutable bindings or an editable model, give each variant its own state through a wrapper or the project's supported `@Previewable` setup.
- Keep existing deployment targets. Annotate the preview with its availability; older SDKs still need separate preview declarations because a runtime guard cannot add an unknown macro overload.

This complete source fragment demonstrates an input type without extra conformances. The preview can be compiled in an iOS 17 or macOS 14 target using Xcode 27, but the argument group itself requires the newer runtime above.

```swift
import SwiftUI

struct RowInput {
    let title: String
    let isLoading: Bool
}

@available(iOS 26.0, macOS 27.0, *)
#Preview("Row states", traits: .sizeThatFitsLayout, arguments: [
    RowInput(title: "Waiting", isLoading: true),
    RowInput(title: "Download complete", isLoading: false),
    RowInput(title: "A long title that may need wrapping", isLoading: false)
]) { input in
    HStack {
        if input.isLoading { ProgressView() }
        Text(input.title)
    }
    .padding()
}
```

## Build offline fixtures for unfinished generated output

For a completed `@Generable` value, [`asPartiallyGenerated()`](https://developer.apple.com/documentation/foundationmodels/generable/aspartiallygenerated()) gives the representation expected by a view that consumes partial output. To exercise missing fields or a growing string, construct [`GeneratedContent(json:)`](https://developer.apple.com/documentation/foundationmodels/generatedcontent/init(json:)) and decode the generated `PartiallyGenerated` type. Apple explicitly allows incomplete JSON; this is not a guarantee that arbitrary malformed input or a value with the wrong field type will decode.

The following complete source fragment requires Foundation Models on iOS/iPadOS/Mac Catalyst 26 or macOS 26; its argument preview additionally requires macOS 27. It creates no session and makes no generation request. The explicit error view distinguishes a broken fixture from a legitimate missing field.

```swift
import SwiftUI
import FoundationModels

@available(iOS 26.0, macOS 26.0, *)
@Generable
struct Draft {
    let title: String
    let summary: String
}

@available(iOS 26.0, macOS 26.0, *)
struct DraftFixtureView: View {
    let json: String

    var body: some View {
        if let draft = try? Draft.PartiallyGenerated(GeneratedContent(json: json)) {
            VStack(alignment: .leading) {
                Text(draft.title ?? "Generating title…")
                    .font(.headline)
                if let summary = draft.summary {
                    Text(summary)
                }
            }
            .padding()
        } else {
            Text("Invalid preview fixture")
                .foregroundStyle(.red)
        }
    }
}

@available(iOS 26.0, macOS 27.0, *)
#Preview("Partial drafts", traits: .sizeThatFitsLayout, arguments: [
    #"{}"#,
    #"{"title":"Trip"#,
    #"{"title":"Trip","summary":"A short"#,
    #"{"title":"Trip","summary":"A short itinerary."}"#
]) { json in
    DraftFixtureView(json: json)
}
```

For an older toolchain that supports Foundation Models but not argument groups, use separate `#Preview` declarations around `DraftFixtureView`. Construct hand-picked snapshots in the property's intended generation order: Apple's [guided-generation documentation](https://developer.apple.com/documentation/foundationmodels/generating-swift-data-structures-with-guided-generation) states that properties are generated in declaration order. These fixtures test rendering inputs, not model correctness, real stream chunk boundaries, latency, or animation timing. The static grid is not a replay of successive updates to one view identity.

## Verification scope

Verified 2026-10-23 against Apple documentation and Xcode 27.1 (27A9275), Apple Swift 6.4. Both complete fragments typechecked together in Swift 6 mode with complete strict concurrency and default nonisolated declarations: macOS SDK 27.0 targeting macOS 14 and 27; iOS Simulator SDK 27.1 targeting iOS 17 and 26. No upcoming concurrency features were enabled.

On an arm64 Mac running macOS 27.0.1, a separate command-line check decoded all four JSON fixtures, checked the missing and populated fields, compared the final fixture with `asPartiallyGenerated()`, and confirmed rejection of a numeric `title` and an invalid JSON token. It created no model session. This validates those fixture conversions, not all possible JSON prefixes. Canvas display, variant labels, interactive state isolation, animation, model execution, and older toolchains/runtimes have not been tested.

## Sources

- https://artemnovichkov.com/blog/previewing-swiftui-states-with-preview-arguments
- https://artemnovichkov.com/blog/working-with-partially-generated-content-in-xcode-previews
