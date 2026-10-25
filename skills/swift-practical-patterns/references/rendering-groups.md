# Geometry and compositing boundaries

Use this reference when changing descendants separate during a parent animation, overlapping content fades incorrectly, a cutout erases too much, a decorative outline needs a soft halo without blurring its content, or a graphic is being considered for rasterization.

Apple documentation verified 2026-10-23; the static-glow section was checked 2026-10-25. `geometryGroup()` requires iOS 17 / macOS 14. `compositingGroup()`, `drawingGroup(opaque:colorMode:)`, and the blend mode below require iOS 13 / macOS 10.15. The overlay example uses iOS 14 / macOS 11 for `accessibilityHidden`. The examples were checked with Swift 6.4; validation details appear below.

## Match the boundary to the problem

- Use `geometryGroup()` to resolve an ancestor's position and size animation at a view boundary before passing the geometry to its descendants. It is useful when descendants are inserted, removed, or replaced during the movement and must remain together. Place it around the complete moving element, inside the ancestor modifier that changes placement. It does not request a rasterized snapshot or synchronize unrelated state changes. [Geometry contract](https://developer.apple.com/documentation/swiftui/view/geometrygroup())
- Use `compositingGroup()` to separate graphical effects into stages. For a combined fade, put the group before the outer opacity modifier: `content.compositingGroup().opacity(amount)`. Reversing them puts the opacity inside the boundary. [Compositing contract](https://developer.apple.com/documentation/swiftui/view/compositinggroup())
- Use `drawingGroup(opaque:colorMode:)` when an offscreen image of supported graphical content is required. It explicitly flattens the subtree; it is not an interchangeable remedy for geometry animation. Neither a group name nor a complex effect proves a performance benefit. [Drawing contract](https://developer.apple.com/documentation/swiftui/view/drawinggroup(opaque:colormode:))

This complete view definition places the changing label and its background inside a geometry boundary, with the alignment-changing frame outside it. Callers targeting earlier systems need an availability branch with ordinary layout; that fallback does not reproduce the geometry-group guarantee.

```swift
import SwiftUI

@available(iOS 17.0, macOS 14.0, *)
struct MovingStatus: View {
    @State private var delivered = false

    var body: some View {
        VStack {
            Text(delivered ? "Delivered" : "Out for delivery")
                .padding()
                .background(.blue.opacity(0.15), in: Capsule())
                .geometryGroup()
                .frame(
                    maxWidth: .infinity,
                    alignment: delivered ? .trailing : .leading
                )
            Button("Update status") {
                withAnimation(.easeInOut(duration: 1)) {
                    delivered.toggle()
                }
            }
        }
    }
}
```

## Keep an eraser inside the intended destination

For `.destinationOut`, the view carrying the blend modifier is the source; previously drawn content is the destination. Source alpha removes destination coverage, so an opaque source color erases completely and partial alpha erases partially. Its hue does not determine the cutout. A plain `ZStack` arranges views without necessarily isolating their blending from surrounding content. Group only the destination you want to erase together with the eraser, leaving the photograph or other underlying content outside. [Blend API](https://developer.apple.com/documentation/swiftui/view/blendmode(_:)), [destination-out equation](https://developer.apple.com/documentation/coregraphics/cgblendmode/destinationout)

This complete decorative overlay definition cuts a circular opening in a dimming layer. Place it above the content to reveal in an outer `ZStack`; leave that content outside the compositing group. The fixed diameter is a demo design choice. A production spotlight should receive the target's actual geometry.

```swift
import SwiftUI

@available(iOS 14.0, macOS 11.0, *)
struct SpotlightOverlay: View {
    var body: some View {
        ZStack {
            Color.black.opacity(0.6)
            Circle()
                .fill(.black)
                .frame(width: 80, height: 80)
                .blendMode(.destinationOut)
        }
        .compositingGroup()
        .allowsHitTesting(false)
        .accessibilityHidden(true)
    }
}
```

Transparency alone does not create a hit-testing hole. This overlay [disables hit testing](https://developer.apple.com/documentation/swiftui/view/allowshittesting(_:)) for the entire decoration; a design that blocks input outside the opening needs a separate interaction policy. Keep essential text and controls legible independently of blending, and inspect the actual background, opacity, drawing order, and supported appearances.

## Blur the decorative stroke, not the content

For a multicolor halo, layer a broad, blurred `strokeBorder` behind a narrower sharp stroke using the same `AngularGradient`. The border is an inner stroke (the shape is inset by half the line width before stroking); the later blur is a separate rendering effect. It does not clip its output to that inner border. Keep the default transparent blur output. [Inner-stroke contract](https://developer.apple.com/documentation/swiftui/insettableshape/strokeborder(_:linewidth:antialiased:)), [blur contract](https://developer.apple.com/documentation/swiftui/view/blur(radius:opaque:))

This complete static decoration definition targets iOS 17 / macOS 14. The two widths and blur radius are fixed visual choices. Apply it with `.overlay { GlowOutline(shape: Capsule()) }` after padding the host; change the shape to match the host. Blur only the broad stroke so the host's text or controls stay sharp.

```swift
import SwiftUI

@available(iOS 17.0, macOS 14.0, *)
struct GlowOutline<Outline: InsettableShape>: View {
    let shape: Outline

    var body: some View {
        let gradient = AngularGradient(
            colors: [.purple, .pink, .orange, .purple],
            center: .center
        )
        ZStack {
            shape.strokeBorder(gradient, lineWidth: 10)
                .blur(radius: 8)
            shape.strokeBorder(gradient, lineWidth: 2)
        }
        .allowsHitTesting(false)
        .accessibilityHidden(true)
    }
}
```

If the host needs clipping, clip the host before adding this overlay; a clip after the overlay also cuts off the halo. Ancestor clips still apply, and an overlay does not reserve extra layout space for its glow. Leave room in the enclosing layout when the halo must remain visible. The decoration is excluded from hit testing and accessibility; keep the real control or status label accessible. [Overlay sizing](https://developer.apple.com/documentation/swiftui/view/overlay(alignment:content:)), [clipping](https://developer.apple.com/documentation/swiftui/view/clipshape(_:style:)), [hit testing](https://developer.apple.com/documentation/swiftui/view/allowshittesting(_:)), [accessibility](https://developer.apple.com/documentation/swiftui/view/accessibilityhidden(_:))

This static version needs no timer or task and can serve as a motion-free alternative. It does not establish interpolation of random gradient stops, appropriate refresh intervals, or rendering cost. Use the scheduling and motion decisions in [animation-control.md](animation-control.md) if animation is actually required; setting an animation duration to zero does not stop a task from changing colors.

## Limit rasterization to supported graphics

`drawingGroup()` includes content rendered directly by SwiftUI, such as text, shapes, and images. Core Animation layer content, including web views, media players, more complex controls/containers, and most UIKit/AppKit views, is replaced by a placeholder. Do not flatten an arbitrary screen merely because one graphic is expensive. [Supported content](https://developer.apple.com/documentation/swiftui/view/drawinggroup(opaque:colormode:))

Keep `opaque: false` for content with cutouts, rounded transparent corners, or any other nonopaque pixels. Passing `true` requires alpha equal to one throughout the image; it is a precondition, not a request to fill transparency. The default color mode is `.nonLinear`; changing it changes the working color space/storage format and can change results. Measure the specific scene before adopting rasterization for speed, including changing content and the size of the offscreen image. Do not assume it caches a dynamic subtree for free. [Parameters](https://developer.apple.com/documentation/swiftui/view/drawinggroup(opaque:colormode:))

## Verification and remaining boundaries

The `MovingStatus` and `SpotlightOverlay` blocks passed `swiftc -typecheck` with Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), the iPhoneOS 27.1 SDK targeting arm64 iOS 14, and the macOS 27.0 SDK targeting arm64 macOS 11; `MovingStatus` retains its iOS 17 / macOS 14 annotation. Checks used complete strict concurrency and warnings as errors: Swift 6 with both `nonisolated` and `MainActor` default isolation, and Swift 5 with `nonisolated` default isolation.

Separate Swift 6 ImageRenderer checks ran on arm64 macOS 27.0.1 at scale 1, reading interior pixels into 8-bit premultiplied sRGB. The exact overlay left the center transparent and the outer scrim at alpha 153/255. With explicit sRGB red behind the group, the opening retained opaque red; removing the boundary also erased that background. A half-alpha source left approximately half the scrim alpha, and changing an opaque eraser from black to green did not change the hole. Overlapping shapes faded once at alpha 128/255 with a group, versus 191/255 without it. A supported shape flattened with `opaque: false` retained its transparent exterior and opaque center.

These checks establish local static output for those inputs. No animated interaction, device display, older runtime, unsupported-content placeholder, hit testing, VoiceOver, appearance matrix, or performance gain was tested. The geometry-group recommendation follows Apple's contract; the source article's specific animation recording was not reproduced.

The exact `GlowOutline` block passed typechecking with the same Xcode/Swift toolchain, the iPhoneOS 27.1 SDK targeting arm64 iOS 17, and the macOS 27.0 SDK targeting arm64 macOS 14. Checks used Swift 6, complete strict concurrency, warnings as errors, and both default-isolation settings above.

A separate macOS 27.0.1 ImageRenderer check at scale 1 placed `GlowOutline(shape: Rectangle())` over a clear 100 × 60 host with 20 points of outer padding and read 8-bit premultiplied sRGB pixels. The rendered size stayed 140 × 100 with or without the overlay; a sample four points outside the host had alpha 74/255, and the center stayed transparent. Clipping the host before the overlay preserved those results. Clipping after the overlay removed the halo beyond a one-pixel boundary band and made the same outside sample transparent; four nonzero pixels remained in the boundary band. These are local static measurements, not a guarantee of an exact blur footprint. Capsule/rounded shapes, device display, motion, interaction, VoiceOver, older systems, and performance were not exercised.

## Sources

- https://nilcoalescing.com/blog/GeometryCompositingAndDrawingGroupsInSwiftUI/
- https://nilcoalescing.com/blog/BlendModesInSwiftUI/
- https://livsycode.com/swiftui/an-apple-intelligence-style-glow-effect-in-swiftui/
