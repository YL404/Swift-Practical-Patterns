# Symbol rendering and effect selection

Use this for standalone SF Symbols whose layers, variable value, or animation do not match the intended state. For a symbol inside a localized sentence, use [inline-symbols.md](inline-symbols.md). Symbol names and supported layers/effects have their own availability; an available modifier does not make every symbol support it.

## Choose the color mechanism

Rendering modes and multiple foreground styles require iOS/iPadOS 15+ or macOS 12+.

- Specify `.monochrome` when a uniform foreground style is required; choose palette when hierarchy levels need separate styles.
- In [palette mode](https://developer.apple.com/documentation/swiftui/symbolrenderingmode/palette), the symbol's authored hierarchy maps to primary, secondary, and tertiary foreground styles. These are hierarchy levels, not arbitrary paths that the app can recolor individually. Supplying [two](https://developer.apple.com/documentation/swiftui/view/foregroundstyle(_:_:)) or [three styles](https://developer.apple.com/documentation/swiftui/view/foregroundstyle(_:_:_:)) selects palette mode unless a different mode is explicitly specified.
- [Multicolor](https://developer.apple.com/documentation/swiftui/symbolrenderingmode/multicolor) can combine inherent styles and the foreground style. Do not assume that every layer ignores `foregroundStyle`, or that applying it can replace every inherent color. Use palette mode when the app needs explicit hierarchy colors; inspect the chosen symbol in the SF Symbols app.

Complete view definition, not a standalone app:

```swift
import SwiftUI

@available(iOS 15.0, macOS 12.0, *)
struct PaletteWarning: View {
    var body: some View {
        Image(systemName: "exclamationmark.triangle.fill")
            .symbolRenderingMode(.palette)
            .foregroundStyle(.yellow, .blue)
            .font(.title)
            .accessibilityLabel("Warning")
    }
}
```

For a state magnitude, use [Image(systemName:variableValue:)](https://developer.apple.com/documentation/swiftui/image/init(systemname:variablevalue:)) on iOS/iPadOS 16+ or macOS 13+. Supply a finite value in `0...1`, normalizing application units first. Unsupported symbols ignore the value; check support rather than treating it as a generic progress fill. `variableValue` represents state, whereas a `.variableColor` symbol effect is an animation and does not report progress.

## Match the animation to its trigger

The basic symbol-effect modifiers and transitions require iOS/iPadOS 17+ or macOS 14+. Choose the API by intent and the effect's protocol conformance:

- A state that stays active: [symbolEffect(_:options:isActive:)](https://developer.apple.com/documentation/swiftui/view/symboleffect(_:options:isactive:)) requires `IndefiniteSymbolEffect`; omitted `isActive` defaults to `true`. Indefinite does not always mean looping: [scale](https://developer.apple.com/documentation/symbols/scalesymboleffect) holds a changed scale, while pulse can repeat. Bind activity to the operation's real lifecycle.
- A transient response to a new event: [symbolEffect(_:options:value:)](https://developer.apple.com/documentation/swiftui/view/symboleffect(_:options:value:)) requires `DiscreteSymbolEffect` and an `Equatable` trigger. Change the trigger for each event; assigning the same value is not a new trigger. This API is not an event queue or a completion callback.
- A change of symbol or variant within a retained view: use [contentTransition(.symbolEffect(.replace))](https://developer.apple.com/documentation/swiftui/contenttransition/symboleffect(_:options:)), which requires `ContentTransitionSymbolEffect`. Keep the view's identity; forcing a new `.id` changes the problem to insertion/removal. The default `.symbolEffect` requests an automatic choice, not a particular morph.
- Insertion/removal of a conditional symbol: use [transition(.symbolEffect)](https://developer.apple.com/documentation/swiftui/transition/symboleffect) or its [effect overload](https://developer.apple.com/documentation/swiftui/transition/symboleffect(_:options:)), which requires `TransitionSymbolEffect`. Other view content is unaffected. Do not substitute this for a retained image's content transition.

Effect availability is separate: [breathe](https://developer.apple.com/documentation/symbols/breathesymboleffect) requires iOS 18/macOS 15, and [drawOn](https://developer.apple.com/documentation/symbols/drawonsymboleffect) requires iOS/macOS 26 and suitable draw data. Gate them separately from the iOS 17/macOS 14 modifiers. Do not infer an effect's supported overloads from its name or from another effect.

This complete demonstration view separates ongoing state, a discrete event, and content replacement. The toggle simulates activity; it does not perform a transfer. It preserves static state feedback when Reduce Motion is enabled. The inner label removes its inherited bounce using [symbolEffectsRemoved](https://developer.apple.com/documentation/swiftui/view/symboleffectsremoved(_:)); the replacement uses an identity content transition instead.

```swift
import SwiftUI

@available(iOS 17.0, macOS 14.0, *)
struct SymbolFeedbackControls: View {
    @Environment(\.accessibilityReduceMotion) private var reduceMotion
    @State private var sending = false
    @State private var event = 0
    @State private var muted = false

    var body: some View {
        VStack {
            Toggle("Sending", isOn: $sending)
            Image(systemName: "paperplane")
                .symbolEffect(.pulse, isActive: sending && !reduceMotion)
                .accessibilityLabel(sending ? "Sending" : "Idle")

            Button {
                event += 1
            } label: {
                Label("Add item", systemImage: "basket")
                    .symbolEffectsRemoved(reduceMotion)
            }
            .symbolEffect(.bounce, options: .repeat(2), value: event)

            Button {
                muted.toggle()
            } label: {
                Label(
                    muted ? "Unmute" : "Mute",
                    systemImage: muted ? "speaker.slash" : "speaker.wave.2"
                )
            }
            .contentTransition(
                reduceMotion ? .identity : .symbolEffect(.replace)
            )
        }
    }
}
```

## Evidence and validation limits

Official documentation checked 2026-10-23. Color rules correct the source's claim that multicolor always ignores foreground styles. Basic typography and conventional labels remain covered by broader SwiftUI guidance.

The two exact Swift blocks were typechecked with Xcode 27.1 (27A9275), Apple Swift 6.4, complete strict concurrency, and warnings as errors. Together with the chart example, they passed eight configurations: Swift 5/6 × nonisolated/MainActor default isolation on arm64 macOS 12.0 (SDK 27.0) and arm64 iOS 15.0 (SDK 27.1). Availability annotations protect the newer declarations; no additional upcoming features were enabled. Five separate macOS probes confirmed rejection of scale as a discrete effect, bounce as a content-transition effect, drawOn as a discrete effect, and breathe/drawOn below macOS 15/26 respectively. These checks use the current compiler, not historical SDKs. No visual animation, rapid retriggering, automatic morph, Reduce Motion behavior, VoiceOver output, symbol fallback, or older runtime behavior is established by typechecking. Verify those in the target interface; do not promise that symbol effects synchronize application work or deliver every rapid event.

## Sources

- [Customizing the appearance of symbol images in SwiftUI](https://nilcoalescing.com/blog/CustomizingTheAppearanceOfSymbolImagesInSwiftUI/) (2024-07-22)
- [Animating SF Symbols in SwiftUI](https://nilcoalescing.com/blog/AnimatingSFSymbolsInSwiftUI/) (2025-12-18)
