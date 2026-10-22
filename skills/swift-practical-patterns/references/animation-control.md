# Custom timing, phase feedback, and timeline updates

Use this reference when an effect needs custom interpolation, a sequence of temporary visual states, or rendering derived from time. Ordinary implicit/explicit animation and basic curve selection need no additional machinery.

## Choose what controls time

| Requirement | Approach | Minimum iOS / macOS |
|---|---|---|
| A reusable progress curve with a fixed duration | `Animation.timingCurve(_:duration:)` with `UnitCurve` | 17 / 14 |
| Interpolation that built-in curves cannot express | `CustomAnimation` | 17 / 14 |
| A temporary sequence of complete visual states | Triggered `phaseAnimator` | 17 / 14 |
| Independent timing tracks for different properties | Keyframe animations | 17 / 14 |
| Content computed from scheduled dates | `TimelineView` | 15 / 12 |

These minimums apply to the selected APIs, not every animation factory. Keep an older target and use availability branches if needed. A runtime check does not supply missing SDK declarations.

## Bound custom interpolation and its interruption policy

Prefer a [UnitCurve-backed animation](https://developer.apple.com/documentation/swiftui/animation/timingcurve(_:duration:)) when a progress curve is enough. Use [CustomAnimation](https://developer.apple.com/documentation/swiftui/customanimation) when its interpolation logic is necessary. Its `animate(value:time:context:)` receives a target vector and elapsed time; return a scaled vector for that time, and return `nil` when finished so SwiftUI can remove the animation. A permanently non-nil result does not signal completion. [Method contract](https://developer.apple.com/documentation/swiftui/customanimation/animate(value:time:context:))

This complete definition supplies a fixed-duration pullback curve. Its negative initial progress is intentional; changing the duration into caller input would also require validating that it is finite and positive.

```swift
import SwiftUI

@available(iOS 17, macOS 14, *)
struct PullbackAnimation: CustomAnimation {
    func animate<V: VectorArithmetic>(
        value: V,
        time: TimeInterval,
        context: inout AnimationContext<V>
    ) -> V? {
        let duration = 0.6
        guard time <= duration else { return nil }
        let t = max(0, time / duration)
        let progress = t * t * (t * (12 - 7 * t) - 4)
        return value.scaled(by: progress)
    }
}
```

Use `Animation(PullbackAnimation())` with `withAnimation` or `.animation(_:value:)` around the intended state change. Honor Reduce Motion at that call site by choosing static feedback or a less motion-heavy alternative. This definition supplies neither a custom velocity nor a merge policy: the defaults return `nil` and `false`, respectively. With the default merge result, overlapping animations run together and SwiftUI combines their results. Do not assume spring-like velocity continuity or replacement semantics for rapid repeated changes. [Velocity](https://developer.apple.com/documentation/swiftui/customanimation/velocity(value:time:context:)), [merging](https://developer.apple.com/documentation/swiftui/customanimation/shouldmerge(previous:value:time:context:))

Keep any per-run mutable data in `AnimationContext.state`, rather than global state or progress stored in the view. The context also exposes the originating view's environment. [AnimationContext](https://developer.apple.com/documentation/swiftui/animationcontext)

For springs, `duration` describes perceptual pace, not an exact deadline at which all motion has settled. This matters particularly for bouncy springs. Use animation completion rather than sleeping for that duration. The default `.logicallyComplete` criterion can fire while the animation has a remaining tail; `.removed` waits for the entire animation, including subsequent animations on the affected properties. Choose the criterion for the actual dependency. [Spring duration](https://developer.apple.com/documentation/swiftui/animation/spring(duration:bounce:blendduration:)), [completion](https://developer.apple.com/documentation/swiftui/withanimation(_:completioncriteria:_:completion:)), [logical completion](https://developer.apple.com/documentation/swiftui/animationcompletioncriteria/logicallycomplete), [removal](https://developer.apple.com/documentation/swiftui/animationcompletioncriteria/removed)

## Keep phase feedback separate from lasting state

The first phase supplies the initial appearance. A triggered sequence advances through the phases and returns to that first appearance; store a lasting result such as “saved” separately. The animation closure selects the animation **into its destination phase**, including the return to the first phase. A delay on that animation delays entry into that destination. [Triggered initializer](https://developer.apple.com/documentation/swiftui/phaseanimator/init(_:trigger:content:animation:)), [WWDC23 phase explanation and sample](https://developer.apple.com/videos/play/wwdc2023/10157/)

This complete view definition accepts an event counter and durable saved state from its owner. Increment `saveEvent` after a successful save if each occurrence should produce feedback; assigning `true` repeatedly to an already-true Boolean does not produce a changed trigger. The counter is an event signal, not a phase index.

```swift
import SwiftUI

@available(iOS 17, macOS 14, *)
struct SaveFeedback: View {
    let saveEvent: Int
    let isSaved: Bool
    @Environment(\.accessibilityReduceMotion) private var reduceMotion

    var body: some View {
        if reduceMotion {
            Text(isSaved ? "Saved" : "Not saved")
        } else {
            Text(isSaved ? "Saved" : "Not saved")
                .phaseAnimator([1.0, 1.08, 0.96], trigger: saveEvent) {
                    content, scale in
                    content.scaleEffect(scale)
                } animation: { destination in
                    destination == 1.0
                        ? .easeOut(duration: 0.2)
                        : .easeInOut(duration: 0.15)
                }
        }
    }
}
```

No `withAnimation` is needed just to change the trigger; the animator owns its transitions. This view only renders feedback and performs no save. Do not infer a durable operation's success from animation completion. Closely spaced triggers, removal/reinsertion, and changing the phase collection need separate runtime checks; this example does not promise to queue one full playback per event.

Use a nonempty phase sequence. The untriggered overload cycles continuously; to stop an acknowledged repeating alert, replace its animated branch with static content. Preserve a text or accessibility indication of the alert. For independent timing per property, choose keyframes instead. [Looping modifier](https://developer.apple.com/documentation/swiftui/view/phaseanimator(_:content:animation:)), [Reduce Motion](https://developer.apple.com/documentation/swiftui/environmentvalues/accessibilityreducemotion)

## Treat TimelineView as scheduled rendering

[TimelineView](https://developer.apple.com/documentation/swiftui/timelineview) supplies scheduled dates to a content closure. Derive displayed values from `context.date`; counting callbacks or adding a fixed frame delta would tie correctness to how often rendering occurs. An animation schedule reevaluates content; it does not by itself interpolate a state change.

| Schedule | Appropriate use and limit |
|---|---|
| `.everyMinute` | Clock content aligned to minute boundaries. |
| `.periodic(from: start, by: interval)` | A custom date origin and interval. With a past origin the initial context uses the latest interval boundary; with a future origin the view initially renders the current date. |
| `.animation(minimumInterval: interval, paused: flag)` | Continuously time-derived visuals. The interval limits how quickly the schedule generates updates; it does not guarantee a frame rate. `nil` lets the system choose. |

[Minute schedule](https://developer.apple.com/documentation/swiftui/timelineschedule/everyminute), [periodic schedule](https://developer.apple.com/documentation/swiftui/timelineschedule/periodic(from:by:)), [animation schedule](https://developer.apple.com/documentation/swiftui/timelineschedule/animation(minimuminterval:paused:))

The system can use a slower cadence than the requested schedule. `live < seconds < minutes`; hide detail that updates faster than the current cadence. This complete view definition uses fractional seconds only at live cadence and hides seconds at minute cadence. [Cadence](https://developer.apple.com/documentation/swiftui/timelineview/context/cadence-swift.enum)

```swift
import SwiftUI

@available(iOS 15, macOS 12, *)
struct CadenceClock: View {
    var body: some View {
        TimelineView(.animation(minimumInterval: 1.0 / 30.0)) { context in
            let format: Date.FormatStyle = switch context.cadence {
            case .live:
                .dateTime.hour().minute().second()
                    .secondFraction(.fractional(2))
            case .seconds:
                .dateTime.hour().minute().second()
            default:
                .dateTime.hour().minute()
            }
            Text(context.date, format: format)
                .monospacedDigit()
        }
    }
}
```

This is an illustrative live clock, not a recommendation to redraw a minute-only clock at animation cadence; use `.everyMinute` for that task. The switch expression requires a Swift 5.9+ compiler. Keep the timeline around the smallest time-dependent subtree and keep its content free of network requests or business side effects.

Pausing the animation schedule stops it generating updates. It does not freeze wall-clock time: a function of `context.date` will use the later date when updates resume. If an effect must resume from a frozen position, manage its logical elapsed time separately. This is a consequence of the chosen date-based calculation, not a guarantee about exact pause/resume callback timing. Avoid large motion when Reduce Motion is enabled; a static decorative branch can avoid installing the animation schedule at all.

On iOS, neither TimelineView nor a run-loop timer grants execution while the app is suspended; separate view rendering from data refresh and background work. [iOS background execution](https://developer.apple.com/documentation/uikit/extending-your-app-s-background-execution-time). For app background refresh, read [background-refresh.md](background-refresh.md).

## Verification boundary

Verified on 2026-10-23 against the linked Apple documentation and Xcode 27.1 public SDK declarations. The phase behavior above follows Apple's initializer discussion and WWDC sample. The modifier page's wording that each trigger change advances one phase differs from that explanation; a local macOS check also advanced through the full sequence from one trigger change. Do not build manual per-phase trigger stepping from that wording.

All three exact Swift blocks passed warnings-as-errors typechecking in Swift 5 and Swift 6 language modes with Apple Swift 6.4 (swiftlang-6.4.0.34.1), Xcode 27.1 (27A9275): macOS SDK 27.0 targeting arm64 macOS 14/12 as annotated, and iOS SDK 27.1 targeting arm64 iOS 17/15. These deployment targets were typechecked, not run on those older systems.

On macOS 27.0.1 arm64, a temporary AppKit/SwiftUI host using the phase example's sequence and timing logged initial scale 1.0 and, for each of two separated trigger changes, 1.08 → 0.96 → 1.0 without further trigger changes. This checks local phase callbacks, not visible interpolation, rapid triggers, or iOS behavior. The custom curve's runtime interpolation and merging, device cadence reduction, pause/resume lifecycle, accessibility presentation, and background behavior were not exercised.

## Sources

- https://nilcoalescing.com/blog/AnimationTimingInSwiftUI/
- https://nilcoalescing.com/blog/PhaseAnimationsInSwiftUI/
- https://nilcoalescing.com/blog/TimelineViewInSwiftUI/
