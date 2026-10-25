# Selection foregrounds and custom row backgrounds

Use this when a custom icon keeps its accent color over a selected row, or when a Mac selection must distinguish focus from selection. `backgroundProminence` requires iOS/iPadOS 17+ or macOS 14+.

## Resolve foregrounds inside the row

[`backgroundProminence`](https://developer.apple.com/documentation/swiftui/environmentvalues/backgroundprominence) describes the background behind content. Standard foreground styles already adapt; custom colors can switch to a semantic style when prominence increases. Read the environment in a descendant inside the row, rather than in the view constructing the whole list. A custom `ShapeStyle.resolve(in:)` can resolve at the styled element instead.

Keep the selected ID or selected-ID set as the source of truth. Prominence is a styling input, not a selection Boolean or a replacement for keyboard focus. [`BackgroundProminence`](https://developer.apple.com/documentation/swiftui/backgroundprominence) describes focused selected rows as one example of increased prominence.

## Choose the selection background before tracking focus manually

For conventional selection colors, use [`.selection`](https://developer.apple.com/documentation/swiftui/shapestyle/selection) only on selected items. On macOS and iPadOS it accounts for the key window and nearest focusable element; on iPhone it uses the accent color. Applied as a background, it also supplies the matching prominence to foreground content.

Complete view example for iOS/iPadOS 17+ and macOS 14+. The caller owns selection and focus; this row only renders them:

```swift
import SwiftUI

struct SelectionRow: View {
    let title: String
    let isSelected: Bool

    var body: some View {
        HStack {
            Image(systemName: "bookmark.fill")
                .foregroundStyle(BookmarkStyle())
            Text(title)
            Spacer()
        }
        .padding(8)
        .background(
            isSelected ? AnyShapeStyle(.selection) : AnyShapeStyle(.clear),
            in: .rect(cornerRadius: 6)
        )
    }
}

private struct BookmarkStyle: ShapeStyle {
    func resolve(in environment: EnvironmentValues) -> some ShapeStyle {
        environment.backgroundProminence == .increased
            ? AnyShapeStyle(.secondary)
            : AnyShapeStyle(.orange)
    }
}
```

Inside a native `List` or `Table`, start with its supplied selection background and prominence rather than adding another fill. With a fully custom background, explicitly set `.environment(\.backgroundProminence, .increased)` or `.standard` on the foreground subtree according to the background actually drawn. Writing that environment value does not draw a background or establish focus. Retain manual focus/window tracking only when the requested custom presentation needs it; the article's blanket recommendation for non-List containers is broader than the documented `.selection` behavior.

## Evidence and limits

Verified 2026-10-25 against Apple's documentation and the Xcode 27.1 public SDK interfaces. The exact view example passed Swift 6 typechecking with complete strict concurrency and warnings as errors, using both nonisolated and MainActor default isolation, targeting iOS 17 with the iPhoneOS 27.1 SDK and macOS 14 with the macOS 27.0 SDK. The same source targeting iOS 16 or macOS 13 was rejected for availability. Checks used Apple Swift 6.4 on macOS 27.0.1 arm64, without extra upcoming features.

No window or device UI was run. Selection changes, focus transfer, inactive windows, contrast, appearance variants, and accessibility remain runtime checks for the actual container. In particular, the example does not implement keyboard navigation or click selection.

## Sources

- [A WWDC 26 Update on Building a Mac-assed App with SwiftUI](https://pfandrade.me/blog/swiftui-mac-assed-wwdc27-update/) (2026-06-23)
