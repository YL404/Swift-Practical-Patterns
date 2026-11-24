# Live Activity lifecycle and push-to-start

Use for an iPhone or iPad app that manages a standard Live Activity locally (iOS 16.2+) or starts one remotely with ActivityKit push-to-start (iOS 17.2+) and downloads assets in the background. Live Activities began in iOS 16.1; supporting 16.1 requires its older non-`ActivityContent` API variants. This reference does not implement APNs server infrastructure, broadcast channel creation, scheduled/transient activities, or interactive App Intents. [Request availability](https://developer.apple.com/documentation/activitykit/activity/request(attributes:content:pushtype:)), [Push notifications overview](https://developer.apple.com/documentation/activitykit/starting-and-updating-live-activities-with-activitykit-push-notifications)

## Configure the two targets

Add a Widget Extension with Include Live Activity, set the app target's Boolean `NSSupportsLiveActivities` to `YES`, and include the attributes definition in both targets. The extension supplies an `ActivityConfiguration`, registered in its `WidgetBundle`; it needs Lock Screen, expanded, compact leading/trailing, and minimal presentations. The system chooses the presentation, so multiple activities do not guarantee the minimal presentation on every device. [App configuration](https://developer.apple.com/documentation/activitykit/displaying-live-data-with-live-activities), [Info.plist key](https://developer.apple.com/documentation/bundleresources/information-property-list/nssupportsliveactivities), [ActivityConfiguration](https://developer.apple.com/documentation/widgetkit/activityconfiguration)

A Live Activity uses ActivityKit updates rather than a widget timeline, and its extension presentation cannot fetch network or location data directly. Local updates require the app to have execution time; starting an activity locally does not grant continuous background execution. To start or update while the app is not running, add the Push Notifications capability and use ActivityKit push tokens rather than registering the Live Activity with UserNotifications. [Execution and data constraints](https://developer.apple.com/documentation/activitykit/displaying-live-data-with-live-activities), [Push update workflow](https://developer.apple.com/documentation/activitykit/starting-and-updating-live-activities-with-activitykit-push-notifications)

## Separate event identity from changing display state

Keep fixed event identifiers and labels in the attributes. Put replaceable display values in the `Codable`, `Hashable` content state. Use a durable business identifier to correlate activities after relaunch; the activity's own `id` identifies the system activity, not the journey. The correlation choice is implementation advice. Keep the combined static and dynamic data within 4 KB; omit large assets and complete domain models. When supporting remote push-to-start or push updates, do not rely on custom JSON encoding/decoding strategies: the system always decodes ActivityKit push payloads with default strategies, and custom strategies cause updates or starts to fail. In targets using Swift 6 `MainActor` default isolation, mark the attributes type `nonisolated` (adding `Sendable` alone still leaves an isolated `ActivityAttributes` conformance), because ActivityKit's `update`, `end`, and async update sequences use the conformance from `@concurrent` contexts. [Attributes and size limits](https://developer.apple.com/documentation/activitykit/displaying-live-data-with-live-activities), [Payload decoding contract](https://developer.apple.com/documentation/activitykit/starting-and-updating-live-activities-with-activitykit-push-notifications)

This complete shared model uses app-supplied strings; `journeyID` must be stable for the same journey.

```swift
import ActivityKit
import Foundation

@available(iOS 16.2, *)
nonisolated struct JourneyAttributes: ActivityAttributes, Sendable {
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

Check `areActivitiesEnabled` to choose the app's starting UI, and still handle request errors: authorization can change and system limits can prevent creation. The ordinary local request below must originate while the app is foregrounded. `pushType: nil` selects local updates; pass `.token` (or `.channel(channelID)` on iOS 18+) when the activity will receive ActivityKit push updates. [Authorization](https://developer.apple.com/documentation/activitykit/activityauthorizationinfo/areactivitiesenabled), [Request contract](https://developer.apple.com/documentation/activitykit/activity/request(attributes:content:pushtype:))

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

## Start remotely with push-to-start and hydrate assets

Remote push-to-start requires **iOS / iPadOS 17.2+** (on iOS 17.1 and earlier, ActivityKit pushes can only update or end an activity already started in the app). Broadcast channels cannot start a Live Activity. [Push-to-start workflow](https://developer.apple.com/documentation/activitykit/starting-and-updating-live-activities-with-activitykit-push-notifications), [pushToStartTokenUpdates](https://developer.apple.com/documentation/activitykit/activity/pushtostarttokenupdates)

- **Observe the push-to-start token early:** iterate `Activity<Attributes>.pushToStartTokenUpdates` at app launch and upload the hex-encoded token to your server. You do not need to start a Live Activity locally to receive this token. Rotate and invalidate outdated tokens on the server whenever a new token arrives.
- **Construct valid APNs start payloads:** send `apns-push-type: liveactivity`, `apns-topic: <bundleID>.push-type.liveactivity`, and `apns-priority` `5` or `10` (prefer `5` when immediate delivery is unnecessary to preserve the hourly push budget; opt into `NSSupportsLiveActivitiesFrequentUpdates` only for genuinely frequent updates). Inside `aps`, include `timestamp`, `"event": "start"`, `attributes-type`, `attributes`, `content-state`, and a required `alert` so the user is notified. On iOS 18+, add `"input-push-token": 1` to request a per-activity update token or `"input-push-channel": "<channelID>"` to subscribe to channel updates; do not send `input-push-channel` to pre-iOS 18 devices or the activity fails to start.
- **Hydrate assets beyond the 4 KB payload limit in the app:** per Apple's documentation, delivering a push-to-start notification starts the Live Activity, wakes the app, and grants background runtime specifically so the app can download assets the Live Activity needs. Configure a shared App Group between the app and widget extension, check both already-present `Activity<Attributes>.activities` and incoming `Activity<Attributes>.activityUpdates`, write downsized files atomically into the App Group container (keeping widget memory limits in mind), and call `activity.update(...)` to mark the content state ready so the widget reloads from disk. Deduplicate concurrent downloads by `activity.id` so overlapping startup checks and `activityUpdates` events do not race. Also observe `activity.pushTokenUpdates` when the server needs the per-activity update token; do not read `activity.pushToken` synchronously right after creation because token generation is asynchronous.

These complete helpers compile against the iOS 17.2 SDK for observing push-to-start tokens and preparing remote assets for incoming activities. The caller supplies validated download URLs, App Group file paths, server upload handlers, per-activity deduplication, and task-lifecycle cancellation.

```swift
import ActivityKit
import Foundation

@available(iOS 17.2, *)
nonisolated struct MediaCardAttributes: ActivityAttributes, Sendable {
    struct ContentState: Codable, Hashable, Sendable {
        var assetReady: Bool
    }

    let cardID: String
    let assetURL: URL?
}

@available(iOS 17.2, *)
func observePushToStartTokens(
    uploadHexToken: @Sendable (String) async -> Void
) async {
    for await tokenData in Activity<MediaCardAttributes>.pushToStartTokenUpdates {
        let hexToken = tokenData.map { String(format: "%02x", $0) }.joined()
        await uploadHexToken(hexToken)
    }
}

@available(iOS 17.2, *)
func hydrateRemoteActivities(
    prepareAsset: @Sendable (URL, String) async throws -> Void
) async {
    for activity in Activity<MediaCardAttributes>.activities {
        await hydrateIfNeeded(activity, prepareAsset: prepareAsset)
    }
    for await activity in Activity<MediaCardAttributes>.activityUpdates {
        await hydrateIfNeeded(activity, prepareAsset: prepareAsset)
    }
}

@available(iOS 17.2, *)
private func hydrateIfNeeded(
    _ activity: Activity<MediaCardAttributes>,
    prepareAsset: @Sendable (URL, String) async throws -> Void
) async {
    guard !activity.content.state.assetReady,
          let url = activity.attributes.assetURL else {
        return
    }
    do {
        try await prepareAsset(url, activity.attributes.cardID)
        await activity.update(
            ActivityContent(state: .init(assetReady: true), staleDate: nil)
        )
    } catch {
        // Handle or report failure according to app policy without marking ready.
    }
}
```

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

Official documentation and SDK interfaces rechecked 2026-11-24. All code blocks were extracted verbatim and passed `swiftc -typecheck` with Xcode 27.2 (27B52), Apple Swift 6.4.2 (swiftlang-6.4.2.1.9), iPhoneOS 27.2 SDK targeting arm64 iOS 16.2 (with `@available(iOS 17.2, *)` on push-to-start helpers) under Swift 6 (`nonisolated` and `MainActor` default isolation) and Swift 5 (`nonisolated` default isolation), complete strict concurrency, and warnings as errors. Separate unguarded probes verified that `ActivityContent` request APIs are rejected at iOS 16.1 and `pushToStartTokenUpdates` is rejected at iOS 17.1 and accepted at iOS 17.2.

No signed app or widget extension was built or launched. Authorization changes, APNs push-to-start delivery, background wake runtime budgets, App Group file sharing under extension memory limits, activity creation/recovery, lifecycle timing, Lock Screen/Dynamic Island rendering, accessibility, older OS execution, and paired-device presentations were not tested locally. Typechecking establishes API use only; the lifecycle and push-to-start wake behavior above comes from the linked Apple contracts.

## Sources

- [Getting started with Live Activities in SwiftUI](https://tanaschita.com/activitykit-live-activities-swiftui/) (2026-09-07).
- [How I sent a video to your lock screen](https://blog.jacobstechtavern.com/p/videos-on-your-lock-screen) (2026-11-18; public Act I push-to-start asset-hydration pattern verified against Apple documentation).
