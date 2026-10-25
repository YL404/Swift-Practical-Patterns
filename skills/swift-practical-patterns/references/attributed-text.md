# Fonts, semantic styles, and paragraphs in read-only SwiftUI text

Use this for bundled custom-font failures, distinct measurement-unit typography, app-defined Markdown attributes and link handling, or paragraph baseline distance. The measurement and Markdown examples need iOS/iPadOS 15+ or macOS 12+; the in-app browser preference and line-height API need iOS/iPadOS 26+ or macOS 26+. The paragraph view includes an older-system fallback. This reference covers those Apple platforms.

## Diagnose a custom font before adjusting its styling

A successful build or visible text does not establish that the requested font loaded. SwiftUI falls back to the system font when it cannot retrieve the named font. Check the resource, registration, and name before changing sizes or weights. [Apple custom-font setup and fallback](https://developer.apple.com/documentation/swiftui/applying-custom-fonts-to-text)

For fonts shipped with an iOS/iPadOS app, verify target membership and the actual files in the built app bundle, then list their bundle-relative paths, including extensions, in the `UIAppFonts` array. A native macOS app instead uses `ATSApplicationFontsPath` with the folder containing its bundled fonts. Installing a font on the development Mac does not package it for another target. Keep these platform configurations separate. [Platform-specific registration](https://developer.apple.com/documentation/swiftui/applying-custom-fonts-to-text)

Use the font's **PostScript name** in `Font.custom`, not its resource filename or a family name guessed from Font Book. For example, `Font.custom("BrandText-Regular", size: 17, relativeTo: .body)` uses a placeholder PostScript name; replace it with the name from the licensed font actually bundled with the app. This initializer requires iOS/iPadOS 14+ or macOS 11+ and preserves scaling relative to the chosen text style. [Initializer and availability](https://developer.apple.com/documentation/swiftui/font/custom(_:size:relativeTo:))

If the base font loads but bold or italic styling has no effect, check whether that font supplies the corresponding variant. SwiftUI does not synthesize missing bold or italic styles. Verify the bundled font on the target platform, including larger Dynamic Type sizes; a font present in the Mac's catalog is not evidence of availability on iOS. [Font variants and scaling](https://developer.apple.com/documentation/swiftui/applying-custom-fonts-to-text)

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

## Choose baseline distance separately from the gap between lines

On iOS/iPadOS 26+ and macOS 26+, `lineHeight(_:)` sets the default paragraph line height in the view hierarchy; `nil` lets SwiftUI choose for the context. `AttributedString.LineHeight` describes the distance between successive baselines. `lineSpacing(_:)` instead specifies the gap from the bottom of one line to the top of the next, so the numbers are not interchangeable. [Default line height](https://developer.apple.com/documentation/swiftui/view/lineheight(_:)), [baseline definition](https://developer.apple.com/documentation/foundation/attributedstring/lineheight), [line spacing](https://developer.apple.com/documentation/swiftui/view/linespacing(_:))

Choose the rule for the paragraph:

- `.multiple(factor:)` uses the largest font point size multiplied by the factor. It can grow with a Dynamic Type-aware font.
- `.leading(increase:)` adds a fixed point increase to the largest font point size. The increase itself does not scale.
- `.exact(points:)` fixes the total baseline distance, independently of the fonts' ascent and descent. A literal fixed value does not grow with Dynamic Type; check for overlapping or clipped lines as text grows.
- `.normal`, `.tight`, and `.loose` are point-size-based presets; `.variable` uses font metrics. Keep the default when no explicit paragraph rule is required.

The three numeric rules above ignore effective ascent and descent. A multiple therefore still needs checks with mixed font sizes, scripts, and accessibility text sizes; it is not a guarantee that all glyphs fit. [Multiple](https://developer.apple.com/documentation/foundation/attributedstring/lineheight/multiple(factor:)), [fixed increase](https://developer.apple.com/documentation/foundation/attributedstring/lineheight/leading(increase:)), [exact height](https://developer.apple.com/documentation/foundation/attributedstring/lineheight/exact(points:)), [presets](https://developer.apple.com/documentation/foundation/attributedstring/lineheight)

Complete view definition for paragraph text supplied as a runtime string:

```swift
struct ParagraphText: View {
    let text: String

    var body: some View {
        if #available(iOS 26.0, macOS 26.0, *) {
            Text(text)
                .font(.body)
                .lineHeight(.multiple(factor: 1.3))
        } else {
            Text(text)
                .font(.body.leading(.loose))
        }
    }
}
```

Include `import SwiftUI`. The factor is an example design choice. The older-system branch uses a font's leading adjustment as an approximate fallback, not an equivalent baseline distance. Some fonts do not support that adjustment and return the original font. Do not combine leading, line spacing, and line height expecting their numeric effects to match or add without layout checks. [Font leading behavior](https://developer.apple.com/documentation/swiftui/font/leading(_:))

## Evidence and validation limits

Official sources for the existing styling examples checked 2026-10-23. Custom-font setup, fallback, variants, and initializer availability checked against Apple documentation on 2026-10-25. The placeholder font expression passed Swift 6 typechecking for iOS 14/macOS 11 using the toolchain below; no font resource, app bundle, registration, or rendered variant was tested.

The exact helpers and consuming views passed typechecking with warnings treated as errors using Xcode 27.1 (27A9275), Apple Swift 6.4, complete strict concurrency, explicit nonisolated default isolation, and no additional upcoming features in Swift 6 and Swift 5 modes. Targets were arm64 iOS 15 with the iPhoneOS 27.1 SDK and arm64 macOS 12 with the macOS 27.0 SDK. These deployment-target checks do not test older compilers or older OS runtimes.

On macOS 27.0.1 arm64, Swift 6 execution covered 36 combinations of four locales (`en_US`, `fr_FR`, `ar_EG`, `ja_JP`), three unit widths, and negative/zero/positive kilometer values. All preserved the formatter's characters and assigned the caption font only to semantic unit ranges. Narrow output retained U+202F in French and U+0020 in Arabic for the checked value. Markdown checks covered both tone mappings, emphasis, links, newline preservation, missing scope, invalid enum/type/JSON5 payloads, literal fallback, and tolerated unclosed emphasis.

Apple documents extended attributes as disabled by default. In this local runtime, the scoped initializer decoded the custom key with omitted options and with `allowsExtendedAttributes: false` as well as with `true`. This is an observed discrepancy, not a cross-version guarantee. Do not rely on that flag alone to reject custom document values; use the declared scope and decoder. The example uses the documented explicit `true` configuration.

The exact paragraph view and the complete set of fenced Swift definitions in this reference were typechecked again under the Swift 5/6 configurations and iOS 15/macOS 12 deployment targets above. Separate Swift 6 checks of all seven line-height rules and an unguarded modifier passed for iOS/macOS 26; unguarded use was rejected for iOS 18/macOS 15 as expected. These checks establish signatures and availability, not rendered baseline distances or Dynamic Type behavior.

No SwiftUI view was hosted. Font rendering, Dynamic Type, VoiceOver, iOS execution, URL activation, browser presentation, localization extraction, older runtimes, and performance were not tested.

## Sources

- [Styling measurement unit fonts in SwiftUI](https://nilcoalescing.com/blog/StylingMeasurementUnitFontsInSwiftUI/) (2026-06-07)
- [Rendering Markdown in SwiftUI](https://artemnovichkov.com/blog/rendering-markdown-in-swiftui) (2026-01-13)
- [Adjusting line height in SwiftUI on iOS 26](https://nilcoalescing.com/blog/AdjustingLineHeightInSwiftUIOniOS26/) (2026-03-04)
- [SwiftUI Font and Texts](https://www.swiftyplace.com/blog/swiftui-font-and-texts) (2023-06-01)
