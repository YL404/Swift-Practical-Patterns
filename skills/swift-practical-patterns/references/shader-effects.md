# SwiftUI Metal shader boundaries

Use this reference for procedural color filters, parameters passed from SwiftUI to MSL, missing shader functions, or choosing a shader effect's input contract. Prefer a built-in gradient or ordinary view modifier when it already expresses the requested effect.

Verified against Apple documentation on 2026-10-23. These shader APIs require iOS/iPadOS 17 or macOS 14. The examples cover those platforms; older deployment targets need an availability branch with a conventional SwiftUI fallback.

## Match the effect to its function

The first arguments below are supplied by SwiftUI. Append custom uniform arguments in the same order and compatible MSL types as the Swift call.

| Modifier | Stitchable MSL signature | Use |
|---|---|---|
| `colorEffect` | `half4 name(float2 position, half4 color, args...)` | Transform the current pixel's color. |
| `distortionEffect` | `float2 name(float2 position, args...)` | Return the source position for a destination pixel. |
| `layerEffect` | `half4 name(float2 position, SwiftUI::Layer layer, args...)` | Sample the rasterized source layer to compute a color. |

`position` uses user-space coordinates, measured in points for these SwiftUI views, not physical display pixels. Do not multiply a geometry proxy's size by display scale to normalize it against this position. Source colors are premultiplied in the destination color space. [Color contract](https://developer.apple.com/documentation/swiftui/view/coloreffect(_:isenabled:)), [distortion contract](https://developer.apple.com/documentation/swiftui/view/distortioneffect(_:maxsampleoffset:isenabled:)), [layer contract](https://developer.apple.com/documentation/swiftui/view/layereffect(_:maxsampleoffset:isenabled:))

For distortion and layer effects, `maxSampleOffset` must cover the maximum displacement in **each axis** over all pixels and parameter values. A horizontal displacement bounded by 12 points and no vertical displacement needs `CGSize(width: 12, height: 0)`. `.zero` is appropriate only when sampling never moves from the destination position. A sampling bound does not define layout or replace deliberate padding/clipping decisions. Views backed by UIKit/AppKit may produce a placeholder rather than enter the filtered layer. [Sampling and content limits](https://developer.apple.com/documentation/swiftui/view/layereffect(_:maxsampleoffset:isenabled:))

## Compile the resource and bind its uniforms

Include the `.metal` file in the app target's compiled sources. `ShaderLibrary` is SwiftUI's dynamic lookup type, not a generated class with Swift declarations for each MSL function. `ShaderLibrary.name(...)` refers to the main bundle's default library; use `ShaderLibrary.bundle(resourceBundle).name(...)` when the compiled library belongs to another bundle. A Swift typecheck cannot establish that the named function exists or that its uniform sequence matches. [Dynamic member lookup](https://developer.apple.com/documentation/swiftui/shaderlibrary/subscript(dynamicmember:)-swift.type.subscript), [bundle selection](https://developer.apple.com/documentation/swiftui/shaderlibrary/bundle(_:)), [dynamic function calls](https://developer.apple.com/documentation/swiftui/shaderfunction)

The following pair provides complete function/view definitions, not an app. It generates decorative brightness bands with a caller-supplied phase in `0..<1` cycles. The color uniform is already premultiplied; preserve that representation and apply the source alpha to all four components. Do not replace transparent source pixels with an unconditional opaque color. [Color uniform conversion](https://developer.apple.com/documentation/swiftui/shader/argument/color(_:))

```metal
#include <metal_stdlib>
using namespace metal;

[[ stitchable ]] half4 tintedBands(
    float2 position, half4 source, float2 size, float phase, half4 tint
) {
    float2 uv = position / max(size, float2(1.0));
    half level = half(0.625 + 0.375 * sin(6.2831853 * (uv.x + phase)));
    return half4(tint.rgb * level, tint.a) * source.a;
}
```

```swift
import SwiftUI

@available(iOS 17.0, macOS 14.0, *)
struct BandPanel: View {
    let phase: Double

    var body: some View {
        let phase = phase
        RoundedRectangle(cornerRadius: 16)
            .fill(.white)
            .visualEffect { effect, proxy in
                effect.colorEffect(
                    ShaderLibrary.tintedBands(
                        .float2(proxy.size),
                        .float(phase),
                        .color(.blue.opacity(0.7))
                    )
                )
            }
            .frame(minHeight: 120)
            .accessibilityHidden(true)
    }
}
```

Only the three custom arguments are passed from Swift. `.float2(proxy.size)` keeps dimensions local to the view; `.float(phase)` converts the scalar to MSL `float`; `.color` resolves the SwiftUI color for the destination. `import SwiftUI` suffices for this call site; the SwiftUI Metal header is needed when using `SwiftUI::Layer`, which this color-only function does not use. [Geometry access](https://developer.apple.com/documentation/swiftui/view/visualeffect(_:)), [uniform types](https://developer.apple.com/documentation/swiftui/shader/argument)

For animation, derive phase from elapsed time supplied through `TimelineView` or an appropriate animation driver. Use a stable time origin and a bounded periodic phase when the effect repeats; `.animation` does not promise 60 updates per second. Decide whether resuming should jump with wall time or continue a paused logical clock, and provide a static decorative alternative for Reduce Motion. The schedule, cadence, and lifecycle decisions are covered in [animation-control.md](animation-control.md#treat-timelineview-as-scheduled-rendering).

## Verification boundary

The exact Swift view passed warnings-as-errors typechecking with Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), macOS 27.0 SDK targeting arm64 macOS 14, and iPhoneOS 27.1 SDK targeting arm64 iOS 17. Checks used complete strict concurrency: Swift 6 with nonisolated and MainActor default isolation, and Swift 5 with nonisolated default isolation. The exact MSL function compiled with warnings as errors for `air64-apple-macos14.0` and `air64-apple-ios17.0` and linked into separate Metal libraries. A deliberately nonexistent dynamic shader name also passed Swift typechecking, confirming that this check does not validate library contents.

No SwiftUI app or GPU rendering was run. Resource-bundle loading, Swift/MSL argument binding at runtime, pixel output and transparency, resized display behavior, animated cadence, Reduce Motion, sampling limits, UIKit/AppKit placeholders, older runtimes, and performance were not exercised. The effect contracts above follow Apple documentation; compilation alone does not verify those behaviors.

## Sources

- https://www.createwithswift.com/taking-first-steps-into-metal-shaders/
- https://www.createwithswift.com/custom-parameters-and-animation-with-metal-shaders/
