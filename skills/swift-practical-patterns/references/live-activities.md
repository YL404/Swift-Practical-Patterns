# Local Live Activity lifecycle

Use for an iPhone app that starts a standard Live Activity and supplies local state updates. The `ActivityContent` request, update, and end APIs below require iOS 16.2+. Live Activities began in iOS 16.1; supporting that release requires its older API variants. This reference does not implement push updates, scheduled/transient activities, or interactive App Intents. [Request availability](https://developer.apple.com/documentation/activitykit/activity/request(attributes:content:pushtype:))

## Configure the two targets

Add a Widget Extension with Include Live Activity, set the app target's Boolean `NSSupportsLiveActivities` to `YES`, and include the attributes definition in both targets. The extension supplies an `ActivityConfiguration`, registered in its `WidgetBundle`; it needs Lock Screen, expanded, compact leading/trailing, and minimal presentations. The system chooses the presentation, so multiple activities do not guarantee the minimal presentation on every device. [App configuration](https://developer.apple.com/documentation/activitykit/displaying-live-data-with-live-activities), [Info.plist key](https://developer.apple.com/documentation/bundleresources/information-property-list/nssupportsliveactivities), [ActivityConfiguration](https://developer.apple.com/documentation/widgetkit/activityconfiguration)

A Live Activity uses ActivityKit updates rather than a widget timeline. Its presentation cannot fetch network or location data. Local updates require the app to have execution time; starting an activity does not grant continuous background execution. For updates while the app is not running, evaluate ActivityKit push notifications separately. [Execution and data constraints](https://developer.apple.com/documentation/activitykit/displaying-live-data-with-live-activities), [Push update workflow](https://developer.apple.com/documentation/activitykit/starting-and-updating-live-activities-with-activitykit-push-notifications)

## Separate event identity from changing display state

Keep fixed event identifiers and labels in the attributes. Put replaceable display values in the `Codable`, `Hashable` content state. Use a durable business identifier to correlate activities after relaunch; the activity's own `id` identifies the system activity, not the journey. The correlation choice is implementation advice. Keep the combined static and dynamic data within 4 KB; omit large assets and complete domain models. [Attributes and size limits](https://developer.apple.com/documentation/activitykit/displaying-live-data-with-live-activities)

This complete shared model uses app-supplied strings; `journeyID` must be stable for the same journey.

```swift
import ActivityKit
import Foundation

@available(iOS 16.2, *)
struct JourneyAttributes: ActivityAttributes {
    struct ContentState: Codable, Hashable, Sendable {
        var currentStop: String
        var nextStop: String?
    }

    let journeyID: String
    let line: String
    let destination: String
}
```

## Start, update, and finish from the app

Check `areActivitiesEnabled` to choose the app's starting UI, and still handle request errors: authorization can change and system limits can prevent creation. The ordinary request below must originate while the app is foregrounded. `pushType: nil` selects local updates; it does not configure remote delivery. [Authorization](https://developer.apple.com/documentation/activitykit/activityauthorizationinfo/areactivitiesenabled), [Request contract](https://developer.apple.com/documentation/activitykit/activity/request(attributes:content:pushtype:))

These complete API helpers compile with the shared model. They do not coordinate competing events. The caller validates and orders snapshots, handles thrown errors, retains the returned activity, and serializes updates and ending for each activity. Do not fire independent tasks for every tap and assume their completion order matches the tap order.

```swift
@available(iOS 16.2, *)
enum JourneyActivityError: Error {
    case disabled
}

@available(iOS 16.2, *)
func startJourney(
    attributes: JourneyAttributes,
    state: JourneyAttributes.ContentState,
    staleDate: Date?
) throws -> Activity<JourneyAttributes> {
    guard ActivityAuthorizationInfo().areActivitiesEnabled else {
        throw JourneyActivityError.disabled
    }
    return try Activity.request(
        attributes: attributes,
        content: ActivityContent(state: state, staleDate: staleDate),
        pushType: nil
    )
}

@available(iOS 16.2, *)
func updateJourney(
    _ activity: Activity<JourneyAttributes>,
    state: JourneyAttributes.ContentState,
    staleDate: Date?
) async {
    await activity.update(ActivityContent(state: state, staleDate: staleDate))
}

@available(iOS 16.2, *)
func finishJourney(
    _ activity: Activity<JourneyAttributes>,
    finalState: JourneyAttributes.ContentState,
    dismissalPolicy: ActivityUIDismissalPolicy = .default
) async {
    await activity.end(
        ActivityContent(state: finalState, staleDate: nil),
        dismissalPolicy: dismissalPolicy
    )
}
```

An update supplies the new content state. `staleDate` marks when that snapshot becomes outdated; the widget reads `context.isStale` to change its presentation. It does not schedule a fetch, grant background time, or end the event. The system ignores updates to an ended activity. [Freshness](https://developer.apple.com/documentation/activitykit/activitycontent/staledate), [Update behavior](https://developer.apple.com/documentation/activitykit/activity/update(_:))

End with the final state even when retaining the completed presentation. Ending and removal are different: `.immediate` requests removal immediately, `.default` lets the ended activity remain on the Lock Screen for up to four hours, and `.after(date)` is bounded by that four-hour window. Earlier user removal remains possible; `.after(now + 60)` is not a guaranteed minute of visibility. [Final content](https://developer.apple.com/documentation/activitykit/activity/end(_:dismissalpolicy:)), [Dismissal policies](https://developer.apple.com/documentation/activitykit/activityuidismissalpolicy), [Custom date](https://developer.apple.com/documentation/activitykit/activityuidismissalpolicy/after(_:))

## Recover handles and reconcile lifecycle state

A view's optional activity handle is not a durable inventory. On app launch, enumerate the current activities and match their attributes to the business event. This complete lookup returns all active or stale matches rather than silently choosing one:

```swift
@available(iOS 16.2, *)
func ongoingJourneys(for journeyID: String) -> [Activity<JourneyAttributes>] {
    Activity<JourneyAttributes>.activities.filter { activity in
        activity.attributes.journeyID == journeyID
            && (activity.activityState == .active || activity.activityState == .stale)
    }
}
```

Before creating a replacement, reconcile these matches with the app's journey records. Reuse a relevant activity, and end activities for completed events; choose an explicit policy if duplicates exist. This lookup is a snapshot, not an atomic uniqueness guarantee. The start helper itself does not deduplicate. [Current activities](https://developer.apple.com/documentation/activitykit/activity/activities), [Relaunch reconciliation](https://developer.apple.com/documentation/activitykit/displaying-live-data-with-live-activities#Observe-active-Live-Activities)

Observe `activityStateUpdates` while the app runs to clear stale UI handles after ending or dismissal; tie the observer task's cancellation to its owner. Removing a Live Activity does not cancel its underlying journey or order. A standard activity can be active for up to eight hours, with up to four further hours on the Lock Screen; it is unsuitable as an indefinite tracker. [State updates](https://developer.apple.com/documentation/activitykit/activity/activitystateupdates-swift.property), [Duration and removal](https://developer.apple.com/documentation/activitykit/displaying-live-data-with-live-activities)

## Evidence and limits

Official documentation checked 2026-10-25. The shared model and all helpers were typechecked together with Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), iPhoneOS 27.1 SDK, Swift 6, complete strict concurrency, default nonisolated isolation, and no additional feature flags, targeting iOS 16.2 and iOS 27. A separate unguarded availability probe was rejected at iOS 16.1 and accepted at 16.2.

No signed app or widget extension was built or launched. Authorization changes, activity creation/recovery, lifecycle timing, background execution, APNs delivery, Lock Screen/Dynamic Island rendering, accessibility, older OS execution, and paired-device presentations were not tested. Typechecking establishes API use only; the lifecycle behavior above comes from the linked Apple contracts. The helpers are not a complete app, duplicate reconciler, event serializer, or background service.

## Sources

- [Getting started with Live Activities in SwiftUI](https://tanaschita.com/activitykit-live-activities-swiftui/) (2026-09-07).
