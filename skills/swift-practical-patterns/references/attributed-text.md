# Semantic styling in read-only SwiftUI text

Use this for distinct measurement-unit typography or app-defined Markdown attributes and link handling. The attributed-text examples need iOS/iPadOS 15+ or macOS 12+; the in-app browser preference needs iOS/iPadOS 26+ or macOS 26+. This reference covers those Apple platforms.

## Style a measurement's unit without splitting its formatted string

Format with `Measurement.FormatStyle.attributed`, then target the `.unit` measurement attribute. This preserves the formatter's localized characters, unit order, and spacing instead of finding a hardcoded suffix. The semantic component distinguishes `.value` from `.unit`. [Attributed measurement formatting](https://developer.apple.com/documentation/foundation/measurement/attributedstyle), [measurement components](https://developer.apple.com/documentation/foundation/attributescopes/foundationattributes/measurementattribute/component)

Complete helper and consuming view:

```swift
import SwiftUI

func unitStyledMeasurement<U: Dimension>(
    _ value: Measurement<U>,
    format: Measurement<U>.FormatStyle,
    unitFont: Font
) -> AttributedString {
    format.attributed.format(value).transformingAttributes(
        AttributeScopes.FoundationAttributes.MeasurementAttribute.self,
        AttributeScopes.SwiftUIAttributes.FontAttribute.self
    ) { component, font in
        if component.value == .unit {
            font.value = unitFont
        }
    }
}

struct DistanceLabel: View {
    @Environment(\.locale) private var locale
    let distance: Measurement<UnitLength>

    var body: some View {
        let style = Measurement<UnitLength>.FormatStyle(
            width: .abbreviated, locale: locale, usage: .asProvided
        )
        Text(unitStyledMeasurement(distance, format: style, unitFont: .caption))
            .font(.headline)
    }
}
```

The helper sets only the unit's attributed font; the value can inherit the outer font. Attributed styles take precedence over equivalent view styling. [Attribute transformation](https://developer.apple.com/documentation/foundation/attributedstring/transformingattributes(_:_:_:)-7kw1o), [Text attribute precedence](https://developer.apple.com/documentation/swiftui/text/init(_:)-1a4oh)

Choose conversion policy separately from typography. `.asProvided` retains the supplied unit; other usage/locale combinations can select another unit. Read the SwiftUI environment locale when formatting for that view. `.narrow` means the shortest unit representation, not a universal promise to remove whitespace; preserve locale-specific separators. [Measurement formatting](https://developer.apple.com/documentation/foundation/measurement/formatstyle), [asProvided](https://developer.apple.com/documentation/foundation/measurementformatunitusage/asprovided), [unit widths](https://developer.apple.com/documentation/foundation/measurement/formatstyle/unitwidth)

## Decode custom Markdown attributes, then map them to rendering attributes

For ordinary localized inline Markdown, keep the app's existing localized `Text` path. For runtime document content, explicitly parse an `AttributedString`; wrapping a runtime `String` in `LocalizedStringKey` does not extract translations. A single `Text` renders only supported inline attributes, even if a parser recognizes block structure. [Text Markdown and supported attributes](https://developer.apple.com/documentation/swiftui/text/init(_:)-1a4oh)

Add a custom attribute only when the document format needs an app-specific semantic value. Supply a scope containing the key and make the key Markdown-decodable. The example explicitly enables extended attributes as documented and nests the SwiftUI scope, which also includes Foundation attributes. Custom attribute syntax uses JSON5 values. [MarkdownDecodableAttributedStringKey](https://developer.apple.com/documentation/foundation/markdowndecodableattributedstringkey), [extended parsing](https://developer.apple.com/documentation/foundation/attributedstring/markdownparsingoptions/allowsextendedattributes), [scopes and JSON5](https://developer.apple.com/videos/play/wwdc2021/10109/)

Complete decoder and rendering mapping; include `import SwiftUI`:

```swift
enum NoticeToneAttribute: CodableAttributedStringKey,
    MarkdownDecodableAttributedStringKey {
    enum Value: String, Codable, Sendable { case warning, success }
    static let name = "noticeTone"
}

struct NoticeTextScope: AttributeScope {
    let noticeTone: NoticeToneAttribute
    let swiftUI: AttributeScopes.SwiftUIAttributes
}

func noticeMarkdown(_ source: String) -> AttributedString {
    do {
        var text = try AttributedString(
            markdown: source,
            including: NoticeTextScope.self,
            options: .init(
                allowsExtendedAttributes: true,
                interpretedSyntax: .inlineOnlyPreservingWhitespace
            )
        )
        for run in text.runs {
            let color: Color
            switch run[NoticeToneAttribute.self] {
            case .warning?: color = .orange
            case .success?: color = .green
            case nil: continue
            }
            text[run.range][AttributeScopes.SwiftUIAttributes.ForegroundColorAttribute.self] = color
        }
        return text
    } catch {
        return AttributedString(source) // This fallback displays the input literally.
    }
}
```

For example, `Text(noticeMarkdown("^[Check the address](noticeTone: 'warning')"))` uses an explicit mapping from document values to SwiftUI colors. Decoding the custom attribute alone does not draw it. Invalid enum values or incompatible payload types can fail decoding; choose a fallback or surface the error according to the document's needs. A throwing parser is not a strict Markdown validator. If color communicates status, retain explanatory words or other non-color cues. [Attribute decoding errors](https://developer.apple.com/documentation/foundation/markdowndecodableattributedstringkey/decodemarkdown(from:))

## Apply a link policy separately from styling

`Text` renders a link attribute as an actionable link. An `OpenURLAction` can handle it, discard it, or pass it to the system. `.handled` reports that the app already handled the URL; it does not open the link. On iOS/macOS 26+, `.systemAction(prefersInApp: true)` expresses a preference, not proof of a browser presentation. [Text link behavior](https://developer.apple.com/documentation/swiftui/text/init(_:)-1a4oh), [OpenURLAction](https://developer.apple.com/documentation/swiftui/openurlaction), [systemAction(_:prefersInApp:)](https://developer.apple.com/documentation/swiftui/openurlaction/result/systemaction(_:prefersinapp:))

Complete policy function; `docs.example.com` is a placeholder for this example's permitted host. Adapt it to the app's actual URL policy:

```swift
func documentLinkPolicy(_ url: URL) -> OpenURLAction.Result {
    guard url.scheme == "https", url.host == "docs.example.com" else {
        return .discarded
    }
    if #available(iOS 26.0, macOS 26.0, *) {
        return .systemAction(prefersInApp: true)
    }
    return .systemAction
}
```

Attach it at the consuming text with `.environment(\.openURL, OpenURLAction(handler: documentLinkPolicy))`. If only the browser preference is needed on 26+, `.onOpenURL(prefersInApp: true)` avoids a custom handler. [Preference-only modifier](https://developer.apple.com/documentation/swiftui/view/onopenurl(prefersinapp:))

## Evidence and validation limits

Official sources checked 2026-10-23.

The exact helpers and consuming views passed typechecking with warnings treated as errors using Xcode 27.1 (27A9275), Apple Swift 6.4, complete strict concurrency, explicit nonisolated default isolation, and no additional upcoming features in Swift 6 and Swift 5 modes. Targets were arm64 iOS 15 with the iPhoneOS 27.1 SDK and arm64 macOS 12 with the macOS 27.0 SDK. These deployment-target checks do not test older compilers or older OS runtimes.

On macOS 27.0.1 arm64, Swift 6 execution covered 36 combinations of four locales (`en_US`, `fr_FR`, `ar_EG`, `ja_JP`), three unit widths, and negative/zero/positive kilometer values. All preserved the formatter's characters and assigned the caption font only to semantic unit ranges. Narrow output retained U+202F in French and U+0020 in Arabic for the checked value. Markdown checks covered both tone mappings, emphasis, links, newline preservation, missing scope, invalid enum/type/JSON5 payloads, literal fallback, and tolerated unclosed emphasis.

Apple documents extended attributes as disabled by default. In this local runtime, the scoped initializer decoded the custom key with omitted options and with `allowsExtendedAttributes: false` as well as with `true`. This is an observed discrepancy, not a cross-version guarantee. Do not rely on that flag alone to reject custom document values; use the declared scope and decoder. The example uses the documented explicit `true` configuration.

No SwiftUI view was hosted. Font rendering, Dynamic Type, VoiceOver, iOS execution, URL activation, browser presentation, localization extraction, older runtimes, and performance were not tested.

## Sources

- [Styling measurement unit fonts in SwiftUI](https://nilcoalescing.com/blog/StylingMeasurementUnitFontsInSwiftUI/) (2026-06-07)
- [Rendering Markdown in SwiftUI](https://artemnovichkov.com/blog/rendering-markdown-in-swiftui) (2026-01-13)
