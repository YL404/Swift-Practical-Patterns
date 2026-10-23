# macOS toolbar title, background, and visibility

Use this reference when a Mac window should hide its title, expose content behind the titlebar, use a deliberate toolbar color, or reveal its toolbar only on full-screen hover. Choose the scope before changing modifiers. Basic `.unified`, `.unifiedCompact`, and `.expanded` layout selection is separate from visibility.

## Choose the smallest scope

Scene modifiers configure the window; toolbar view preferences flow up to the nearest container that renders the bar, including the root of a macOS `WindowGroup`. Put a window-wide preference on that window's root content. A preference attached to a child is not necessarily confined to that child's rectangle. Apple's [toolbar visibility contract](https://developer.apple.com/documentation/swiftui/view/toolbarvisibility(_:for:)) describes this propagation and notes that a requested visibility may not be fulfilled for every bar.

The minimum versions below are for macOS, independently of the article's macOS 26 examples. Wrap Mac-specific scene code in `#if os(macOS)` in a shared source file.

| Requested change | Apply | Minimum macOS and scope |
|---|---|---|
| Remove the default title/subtitle item without choosing a different toolbar layout | `.toolbar(removing: .title)` | 15; root content view |
| Hide the title while explicitly choosing a unified layout | `.windowToolbarStyle(.unified(showsTitle: false))` or `.unifiedCompact(showsTitle: false)` | 11; scene |
| Hide the title and the titlebar's backing so more content can show | `.windowStyle(.hiddenTitleBar)` | 11; scene |
| Hide the bar background while retaining the title/action configuration | `.toolbarBackgroundVisibility(.hidden, for: .windowToolbar)` | 15; root content view |
| Choose a preferred toolbar background style | `.toolbarBackground(.black, for: .windowToolbar)` | 13; root content view; visibility is a separate preference |
| Hide the entire titlebar | `.toolbarVisibility(.hidden, for: .windowToolbar)` | 15; root content view |
| Reveal the toolbar on hover while the window is full screen | `.windowToolbarFullScreenVisibility(.onHover)` | 15; root content view; not a windowed-mode visibility setting |

The [`.title` default item](https://developer.apple.com/documentation/swiftui/toolbardefaultitemkind/title) includes **both title and subtitle**. Although [`toolbar(removing:)`](https://developer.apple.com/documentation/swiftui/view/toolbar(removing:)) starts at macOS 14, its `.title` argument requires macOS 15. Do not infer availability from the modifier alone. On macOS 11–14, the [unified style's `showsTitle` parameter](https://developer.apple.com/documentation/swiftui/windowtoolbarstyle/unified(showstitle:)) is an earlier option for hiding the title, if explicitly selecting that layout suits the window; it is not a claim of identical subtitle or placement behavior.

[`hiddenTitleBar`](https://developer.apple.com/documentation/swiftui/windowstyle/hiddentitlebar) hides the title and backing, rather than requesting removal of the entire titlebar. Hiding the entire titlebar can also remove access to its window controls; use it only when that is the requested experience, and check how people close, move, and manage the window. Apple's visibility contract also says that forcing a nonautomatic toolbar visibility disables the visibility command supplied by `ToolbarCommands`.

## Separate the bar background from item glass

[`toolbarBackgroundVisibility`](https://developer.apple.com/documentation/swiftui/view/toolbarbackgroundvisibility(_:for:)) expresses whether the **bar's** background is preferred to be visible. A `.clear` shape style instead requests a transparent background style; do not treat these as interchangeable state controls or assume identical separator behavior on every OS.

On macOS 26+ and iOS 26+, [`sharedBackgroundVisibility(.hidden)`](https://developer.apple.com/documentation/swiftui/toolbarcontent/sharedbackgroundvisibility(_:)) is a different operation: applied to `ToolbarItem` or other toolbar content, it hides that item's shared glass and puts it in its own logical group. It does not request removal of the window's titlebar backing. Use it for an item with its own presentation, such as an avatar; use the bar preference when changing the window toolbar's background. Do not hide the whole titlebar to remove one item's glass.

For deployment below macOS 15, the older `.toolbarBackground(.hidden, for: .windowToolbar)` visibility overload is available from macOS 13. It is soft-deprecated and renamed to `toolbarBackgroundVisibility`; keep it only in an earlier-target fallback. The `ShapeStyle` overload of `toolbarBackground`, such as `.black`, remains the method for choosing a background style.

## Keep a deliberate background and color scheme together

[`toolbarColorScheme`](https://developer.apple.com/documentation/swiftui/view/toolbarcolorscheme(_:for:)) is available from macOS 13. Apple specifies that the requested scheme is respected only while the bar's background is visible, and that fulfillment depends on the bar. Choosing black alone does not express an always-visible background.

Complete compilable demonstration app for macOS 15+; Refresh is an intentional no-op, not an implementation of data loading:

```swift
import SwiftUI

@main
struct ToolbarAppearanceDemo: App {
    var body: some Scene {
        WindowGroup("Reading") {
            Text("Document")
                .frame(minWidth: 320, minHeight: 240)
                .toolbar {
                    ToolbarItem {
                        Button("Refresh", systemImage: "arrow.clockwise") {
                            // Demonstration only; connect the app's refresh action.
                        }
                    }
                }
                .toolbarBackground(.black, for: .windowToolbar)
                .toolbarBackgroundVisibility(.visible, for: .windowToolbar)
                .toolbarColorScheme(.dark, for: .windowToolbar)
        }
    }
}
```

Use a custom color only when the task calls for it. If the intent is to expose the window background while retaining its title and actions, replace the three appearance preferences with `.toolbarBackgroundVisibility(.hidden, for: .windowToolbar)` and let the system resolve foreground contrast; a forced toolbar scheme is not a substitute for checking the underlying content.

[`windowToolbarFullScreenVisibility(.onHover)`](https://developer.apple.com/documentation/swiftui/view/windowtoolbarfullscreenvisibility(_:)) separately makes the toolbar reveal when the mouse enters the menu-bar area in full screen. Use it instead of removing the entire titlebar when full-screen auto-reveal is the goal. The modifier's declaration is cross-platform, but `.onHover` and `.windowToolbar` are macOS-only; gate the expression by platform as well as deployment version.

## Validation and limits

Verified on 2026-10-23 against Apple's documentation and the Xcode 27.1 public SwiftUI interface. The demonstration and the table's API combinations were typechecked with Apple Swift 6.4, complete strict concurrency, explicit default nonisolated isolation, and no extra upcoming features, using the macOS 27.0 SDK on macOS 27.0.1 arm64. Availability checks covered the documented earlier deployment boundaries; this does not execute those older systems.

No window was launched. Title/subtitle positioning, separators, traffic-light visibility, full-screen hover, appearance transitions, keyboard/window-control access, and accessibility were not locally exercised. For a real window, check these under its actual scene style, toolbar content, minimum size, light/dark appearance, and full-screen mode. The article's visually similar combinations on macOS 26 do not establish that all combinations are equivalent or that `.automatic` always resolves to one layout.

## Sources

- [A guide to macOS window toolbar styles in SwiftUI](https://nilcoalescing.com/blog/AGuideToMacOSToolbarStylesInSwiftUI)
