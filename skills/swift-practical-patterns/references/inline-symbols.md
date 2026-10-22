# Inline SF Symbols in localized SwiftUI text

Use this when an icon belongs inside a sentence. For a conventional icon-and-title label, keep `Label`. Image and `Text` interpolation require iOS/iPadOS 14+ or macOS 11+; the styled example below requires iOS/iPadOS 17+ or macOS 14+. Check the chosen symbol's availability separately.

## Keep the rich interpolation builder

Pass the literal directly to `Text`, for example `Text("Tap \(Image(systemName: "arrow.down.circle")) to download.")`. SwiftUI selects `LocalizedStringKey`, whose builder has an [Image interpolation overload](https://developer.apple.com/documentation/swiftui/localizedstringkey/stringinterpolation/appendinterpolation(_:)-5m52e). If this rich literal needs storage, give it an explicit `LocalizedStringKey` type before interpolation occurs.

A prebuilt `String` uses ordinary string interpolation and then `Text`'s nonlocalizing initializer. Wrapping that completed string in `LocalizedStringKey` cannot recover the original image value. [Text initializer selection](https://developer.apple.com/documentation/swiftui/text/init(_:tablename:bundle:comment:))

## Preserve Image or Text when styling

`Image.foregroundStyle(_:)` uses the general [View modifier](https://developer.apple.com/documentation/swiftui/view/foregroundstyle(_:)), which returns `some View`. It no longer matches the image overload. Compilation alone does not establish that an interpolated view will render as an inline image.

For a separately colored symbol, first construct `Text(Image(...))`, then apply the [Text-specific foregroundStyle](https://developer.apple.com/documentation/swiftui/text/foregroundstyle(_:)), which returns `Text`. The localized builder also has a [Text interpolation overload](https://developer.apple.com/documentation/swiftui/localizedstringkey/stringinterpolation/appendinterpolation(_:)-4qyfo).

This is a complete helper function, not a standalone app:

```swift
import SwiftUI

@available(iOS 17.0, macOS 14.0, *)
func styledInstruction() -> Text {
    let symbol: Text = Text(Image(systemName: "arrow.down.circle"))
        .foregroundStyle(.green)
    let message: LocalizedStringKey = "Tap \(symbol) to download."
    return Text(message)
}
```

Do not generalize the restriction to every image modifier. For example, [Image.renderingMode(_:)](https://developer.apple.com/documentation/swiftui/image/renderingmode(_:)) returns `Image` and can retain the image interpolation path. Check the actual return type after the whole modifier chain. Likewise, adding a general view modifier after the styled `Text` may lose its `Text` type.

Markdown image syntax is not a replacement for this mechanism: `Text` does not support the Markdown `imageURL` attribute. Do not infer a broader attachment capability from `AttributedString` accepting Markdown. [Supported Markdown](https://developer.apple.com/documentation/swiftui/text/init(_:tablename:bundle:comment:))

## Evidence and validation limits

Official declarations and documentation checked 2026-10-22. The source's blanket statement about image modifiers is narrowed here to modifiers that change the required type.

Typechecked on macOS 27.0.1 arm64, Xcode 27.1 (27A9275), Apple Swift 6.4, Swift 6 mode, complete strict concurrency, default isolation explicitly nonisolated, and no additional upcoming features. Using the iPhoneOS 27.1 and macOS 27.0 SDKs with arm64 iOS 14.0/macOS 11.0 deployment targets, checks covered direct image interpolation, the availability-annotated helper above, and an `Image`-preserving `renderingMode` chain. Assigning a foreground-styled image to `Image` and using the Text-specific style without its availability annotation were rejected as expected.

Compiler AST inspection confirmed that a prebuilt `String` uses `DefaultStringInterpolation` and the nonlocalizing `Text` initializer; the directly styled image instead selects a generic localized interpolation overload. No app was run. Inline placement, colors, the source's debug-description output, Dynamic Type, VoiceOver, localization extraction/translation, older SDKs, and older OS runtimes were not tested.

## Sources

- [Embedding SF Symbols in SwiftUI Text](https://nilcoalescing.com/blog/EmbeddingSFSymbolsInSwiftUIText/) (2026-04-06)
