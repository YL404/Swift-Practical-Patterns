# Custom formatting in plain Swift strings

Use this when a project repeatedly needs labeled, type-safe formatting inside an ordinary `String` literal. The interpolation mechanism is available in Swift 5+; the Foundation `FormatStyle` example needs iOS/iPadOS 15+ or macOS 12+. This reference covers those Apple platforms.

## Choose whether to add an overload

- For one-off formatting, keep the existing platform API: `"Total: \(amount.formatted(style))"` or `"Total: \(style.format(amount))"` needs no extension. [FormatStyle usage](https://developer.apple.com/documentation/foundation/formatstyle)
- Add an `appendInterpolation` overload only when a shared interpolation convention improves repeated call sites. Check the project and imported dependencies for an existing overload first.
- For localized interface text, keep the literal in the project's localization API. SwiftUI's `LocalizedStringKey.StringInterpolation` already supports format styles; extending `String.StringInterpolation` does not extend that separate builder. Passing a prebuilt `String` variable to `Text` selects its nonlocalizing initializer. [Localized interpolation](https://developer.apple.com/documentation/swiftui/localizedstringkey/stringinterpolation), [Text initializer selection](https://developer.apple.com/documentation/swiftui/text/init(_:tablename:bundle:comment:))

## Add only the required constraints

This is a complete command-line example. In an app, put the example calls inside the consuming function and gate them for the deployment target.

```swift
import Foundation

@available(iOS 15.0, macOS 12.0, *)
extension String.StringInterpolation {
    mutating func appendInterpolation<F: FormatStyle>(
        _ value: F.FormatInput,
        format: F
    ) where F.FormatOutput == String {
        appendLiteral(format.format(value))
    }
}

let amount = 1234.5
let style = FloatingPointFormatStyle<Double>.number
    .precision(.fractionLength(2))
    .locale(Locale(identifier: "en_US"))
let message: String = "Total: \(amount, format: style)"
print(message) // Total: 1,234.50
```

The compiler turns the labeled interpolation into an `appendInterpolation(_:format:)` call on the literal's interpolation builder. `String` uses `DefaultStringInterpolation`; Apple supports extending it with mutating interpolation methods. Do not copy the builder or capture it in an escaping closure. [DefaultStringInterpolation](https://developer.apple.com/documentation/swift/defaultstringinterpolation), [SE-0228](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0228-fix-expressiblebystringinterpolation.md)

`F.FormatInput` ties the value to the style's accepted type. `F.FormatOutput == String` is needed by this implementation's `appendLiteral` call; an attributed-output style needs a different approach. The article additionally requires `F.FormatInput: Equatable`. That constraint is unnecessary here: neither the implementation nor `FormatStyle.FormatInput` requires equality. Keep it only if the actual implementation compares inputs. [FormatInput](https://developer.apple.com/documentation/foundation/formatstyle/formatinput), [FormatOutput](https://developer.apple.com/documentation/foundation/formatstyle/formatoutput)

The style controls value formatting, including locale-sensitive separators. The `en_US` locale above makes the example explicit; choose the caller's intended locale in production. Formatting a value does not translate the surrounding sentence.

## Evidence and validation limits

Official sources checked 2026-10-22.

Compiled and executed on macOS 27.0.1 arm64 with Xcode 27.1 (27A9275), Apple Swift 6.4, Swift 6 mode, complete strict concurrency, default isolation explicitly nonisolated, and no additional upcoming features. Checks covered dates, numbers with both signs and zero under `en_US`/`fr_FR`, and a custom style accepting a non-`Equatable` class. Numeric and date interpolations matched direct style formatting. Missing-overload, mismatched-input, and attributed-output cases were rejected by the compiler as expected. The ordinary `String` call without the extension failed on this toolchain. No iOS runtime, older OS runtime, SwiftUI rendering, or localization-resource extraction was tested; no performance claim is made.

## Sources

- [Defining custom string interpolation behavior in Swift](https://nilcoalescing.com/blog/DefiningCustomStringInterpolationBehaviorInSwift/) (2026-01-13)
