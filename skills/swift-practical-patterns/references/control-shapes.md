# Button shapes and concentric corners

Use this reference when changing a system button's outer shape or making nested corners follow their container. These are separate decisions: clipping content does not configure a button's system platter; use an explicit container shape when choosing custom reference geometry.

Apple documentation verified 2026-10-23. The concentric APIs and glass button styles below require an SDK exposing iOS 26 / macOS 26 APIs. Runtime availability checks do not make those declarations available to an older SDK.

## Configure a system button through its style

For a circular glass button, keep `Button` and use `.buttonStyle(.glass)` with `.buttonBorderShape(.circle)`. Apple recommends the glass button style plus the border-shape preference for this case, rather than applying a raw glass effect to a button. Buttons in system toolbars already receive the appropriate glass treatment; avoid adding another glass layer there. [Apple's SwiftUI Group Lab](https://developer.apple.com/videos/play/wwdc2026/8120/?time=441)

[`buttonBorderShape(_:)`](https://developer.apple.com/documentation/swiftui/view/buttonbordershape(_:)) can apply to one button or a containing view's buttons. It configures the system platter. [`clipShape`](https://developer.apple.com/documentation/swiftui/view/clipshape(_:style:)) instead removes rendered content outside a shape. Use clipping when that removal is the goal, not as a substitute for configuring a system button. For a custom glass surface, [`glassEffect(_:in:)`](https://developer.apple.com/documentation/swiftui/view/glasseffect(_:in:)) has its own shape argument.

Availability depends on the argument as well as the modifier: `buttonBorderShape` starts at iOS 15 / macOS 12, but [`.circle`](https://developer.apple.com/documentation/swiftui/buttonbordershape/circle) needs iOS 17 / macOS 14, and [`.glass`](https://developer.apple.com/documentation/swiftui/primitivebuttonstyle/glass) needs iOS 26 / macOS 26. Apple also notes that on macOS 15 and earlier, some border shapes only apply to bordered buttons in widgets. Do not promise an identical circular fallback on those Mac runtimes.

Complete view definition for iOS 17+ / macOS 14+, using a current SDK. The caller supplies the real action. The earlier-system branch retains a bordered system control; it does not reproduce glass or guarantee the same shape on older macOS.

```swift
import SwiftUI

@available(iOS 17.0, macOS 14.0, *)
struct CircularSettingsButton: View {
    let action: () -> Void

    var body: some View {
        if #available(iOS 26.0, macOS 26.0, *) {
            button.buttonStyle(.glass)
        } else {
            button.buttonStyle(.bordered)
        }
    }

    private var button: some View {
        Button("Settings", systemImage: "gearshape", action: action)
            .labelStyle(.iconOnly)
            .buttonBorderShape(.circle)
    }
}
```

## Resolve corners against the intended container

[`ConcentricRectangle`](https://developer.apple.com/documentation/swiftui/concentricrectangle) and `.rect(corners: .concentric)` require iOS 26 / macOS 26. They derive corner radii from a system-provided container, such as a screen or sheet, or a custom container supplied with [`containerShape`](https://developer.apple.com/documentation/swiftui/view/containershape(_:)-3br47). Use a `RoundedRectangularShape`, such as `RoundedRectangle`, for corner-based resolution. For a container that does not conform, Apple documents an inset-shape fallback like `ContainerRelativeShape`; do not assume four independently resolved corner radii.

Choose the policy deliberately:

- Default concentric corners resolve independently. A corner far from its container corner can become square. The system subtracts the distance between corners from the container's radius; one fixed radius minus padding is not a general rule for every layout.
- [`isUniform: true`](https://developer.apple.com/documentation/swiftui/concentricrectangle/init(corners:isuniform:)) selects the largest resolved radius for all four corners. Use it for a symmetric inset element when independently matching each corner is not the intended design.
- [`.concentric(minimum: 8)`](https://developer.apple.com/documentation/swiftui/edge/corner/style/concentric(minimum:)) supplies a lower bound when the resolved radius would be smaller. The minimum is a design choice; corners using it need not share the container's exact center.

`containerShape` supplies geometry to descendants; draw a background or clip content separately when needed. The corner-aware overload requires iOS 26 / macOS 26. The older [`InsettableShape` overload](https://developer.apple.com/documentation/swiftui/view/containershape(_:)-qn9q) starts at iOS 15 / macOS 12, but does not make `ConcentricRectangle` available there. For an earlier target, branch to an ordinary fixed-radius shape without claiming equivalent adaptation.

Complete iOS 26+ / macOS 26+ view definition. The colored rectangle stands in for artwork. The dimensions, spacing, and minimum radius illustrate a particular card design rather than universal constants; the caller determines the card's width.

```swift
import SwiftUI

@available(iOS 26.0, macOS 26.0, *)
struct ConcentricCard: View {
    let title: String

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            Rectangle()
                .fill(.indigo.gradient)
                .frame(height: 120)
                .clipShape(.rect(
                    corners: .concentric(minimum: 8),
                    isUniform: true
                ))
            Text(title)
        }
        .padding(12)
        .background(.background, in: .rect(cornerRadius: 24))
        .containerShape(.rect(cornerRadius: 24))
    }
}
```

Use `.rect(corners: .concentric)` or `ConcentricRectangle()`, not `.rect(cornerRadius: 12, style: .concentric)`. The latter expects [`RoundedCornerStyle`](https://developer.apple.com/documentation/swiftui/roundedcornerstyle), whose cases are `.circular` and `.continuous`; concentricity belongs to [`Edge.Corner.Style`](https://developer.apple.com/documentation/swiftui/edge/corner/style). This distinction also applies when shaping a button's label: clipping the label is still separate from its system platter.

Safe-area insets and placement change the distance to container corners. Do not add `ignoresSafeArea()` merely to force round corners. Inspect the intended layout, choose independent or uniform corners, and supply a minimum only if the design calls for one.

## Validation and limits

The exact examples were typechecked with Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), iPhoneOS 27.1 and macOS 27.0 SDKs, Swift 6, complete strict concurrency, and warnings as errors. Checks covered both nonisolated and MainActor default isolation, targeting iOS 17 / macOS 14 with each view's availability annotation preserved. Separate checks confirmed the iOS 26 / macOS 26 concentric boundary, the newer `.circle` argument boundary, the older container-shape overload, and rejection of `.concentric` as a `RoundedCornerStyle`.

No UI was launched. The geometry policies above follow Apple's documented contract; rendered corner radii, sheet/popover adaptation, glass press/hover feedback, clipping of highlights, older-system fallback appearance, hit areas, VoiceOver, and Dynamic Type were not locally exercised. Check those in the actual control and container before treating appearance or interaction as accepted. A compiled shape preference is not evidence of identical rendering across contexts or platforms.

## Sources

- https://livsycode.com/swiftui/changing-the-shape-of-glass-buttons-in-swiftui/
- https://nilcoalescing.com/blog/ConcentricRectangleInSwiftUI/
