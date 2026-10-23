# Downloaded notification audio and AlarmKit boundaries

Use this when an iOS notification service extension downloads audio before delivery, or when deciding whether a remotely triggered alert belongs in UserNotifications or AlarmKit. The notification APIs below require iOS 10+; AlarmKit requires iOS 26+. Check each target's deployment, signing, and App Group membership separately. These are separate alert mechanisms with separate contracts.

## Choose the alert mechanism before choosing storage

For a remote message with a custom sound, a notification service extension can assign a `UNNotificationSound` to the notification content. This is an explicitly documented use of UserNotifications. [Apple notification sounds](https://developer.apple.com/documentation/usernotifications/unnotificationsound)

Use AlarmKit for an actual alarm or countdown, with the app's alarm authorization and `NSAlarmKitUsageDescription` configured. Notification permission does not establish alarm authorization. Apple's AlarmKit session says alarms are intended for scheduled alerts and countdowns, and should not replace critical or time-sensitive notifications. A prototype that schedules an alarm for one second after a push is not, by itself, a reason to choose AlarmKit for message delivery. [Apple alarm authorization](https://developer.apple.com/documentation/alarmkit/scheduling-an-alarm-with-alarmkit), [WWDC25 AlarmKit guidance](https://developer.apple.com/videos/play/wwdc2025/230/)

## Use the sound API's own lookup rules

`UNNotificationSound(named:)` searches the app container's `Library/Sounds`, then shared App Group containers' `Library/Sounds`, then the current executable's main bundle. It uses the first matching filename. Configure the same App Group on the containing app and extension before using shared storage; obtain its actual URL with `FileManager.containerURL(forSecurityApplicationGroupIdentifier:)` and handle failure. An App Group container is separate from the app's own data container. [Apple sound lookup](https://developer.apple.com/documentation/usernotifications/unnotificationsound/init(named:)), [App Group configuration](https://developer.apple.com/documentation/xcode/configuring-app-groups)

AlarmKit's `sound:` parameter instead uses `ActivityKit.AlertConfiguration.AlertSound`. Its `named(_:)` documentation specifies the app's main bundle or its data container's `Library/Sounds`; it does not establish App Group lookup. Do not transfer UserNotifications' documented shared-container behavior to this different type. The source article reports scheduling AlarmKit from a service extension with a sound stored in an App Group, but that combination remains an author observation, not a verified recipe here. [Apple ActivityKit sound contract](https://developer.apple.com/documentation/activitykit/alertconfiguration/alertsound/named(_:)), [WWDC25 sound configuration](https://developer.apple.com/videos/play/wwdc2025/230/)

For ordinary notification sounds, the file must exist locally before delivery, use a supported audio encoding in an AIFF, WAV, or CAF container, and be shorter than 30 seconds. Longer files use the default sound. A `.caf` suffix alone does not validate downloaded bytes. Validate the actual media before selecting it. Use an app-generated, per-notification filename and retain the file until it is no longer needed; repeatedly overwriting `alarm.caf` can replace another pending alert's audio. The filename and retention choices are implementation advice, not an OS guarantee. Do not infer AlarmKit's format or duration limits from these UserNotifications rules. [Apple sound resource requirements](https://developer.apple.com/documentation/usernotifications/unnotificationsound)

## Bound the service extension's work

Send an alert notification with `aps.mutable-content` set to `1` and an `aps.alert` dictionary containing title and body. The service extension is not a handler for silent, sound-only, or badge-only pushes. Preserve visible alert text in the returned content. [Apple service extension](https://developer.apple.com/documentation/usernotifications/unnotificationserviceextension), [payload configuration](https://developer.apple.com/library/archive/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/ModifyingNotifications.html)

The extension has at most 30 seconds to supply content, not a guaranteed 30-second download window. On download, validation, or storage failure, return usable original or fallback content promptly. If completion misses the deadline, the system displays the original notification. Keep that original content meaningful without the download. [Apple receive callback](https://developer.apple.com/documentation/usernotifications/unnotificationserviceextension/didreceive(_:withcontenthandler:))

`serviceExtensionTimeWillExpire()` runs on a separate thread and must supply fallback content quickly. Implementation advice: coordinate success, failure, and expiry through one synchronized terminal state that consumes the completion callback once; call the consumed callback outside its lock. Retain the download task so expiry can cancel it. Cancellation is cooperative: prevent late results from replacing fallback content or initiating further side effects, and do not assume an already-started external operation is rolled back. A `defer` inside an unstructured task only runs when that task exits; it does not provide the expiry path or extend the process lifetime. See [synchronous shared state](synchronous-shared-state.md) for locking boundaries, checking its platform minimum before choosing `Mutex`. This reference does not supply a complete lifecycle implementation. [Apple expiry callback](https://developer.apple.com/documentation/usernotifications/unnotificationserviceextension/serviceextensiontimewillexpire())

Once a supported file is saved and validated, this complete helper selects it for an ordinary notification while preserving the original alert. `preparedSound` must be a trusted filename generated by the app, not an unchecked remote path. This is content preparation only; download, media validation, shared storage, callback arbitration, and file cleanup belong to the caller.

```swift
import UserNotifications

func notificationContent(
    from original: UNNotificationContent,
    preparedSound: UNNotificationSoundName
) -> UNNotificationContent {
    guard let content = original.mutableCopy() as? UNMutableNotificationContent else {
        return original
    }
    content.sound = UNNotificationSound(named: preparedSound)
    return content
}
```

## Evidence and limits

Official sources checked 2026-10-23. The helper was typechecked with Xcode 27.1 (27A9275), Apple Swift 6.4, the iPhoneOS 27.1 SDK, Swift 6, complete strict concurrency, application-extension restrictions, and explicit default nonisolated isolation for iOS 10 and iOS 27 targets. This checks API use, not sound playback or extension execution.

The article's public prototype section was read; its subsequent recording, server, storage, and pairing implementation was behind a paywall. No APNs delivery, signed App Group access, audio decoding/playback, permission transitions, expiry races, AlarmKit scheduling from an extension, or device behavior was executed. Do not claim that the AlarmKit/App Group combination is unsupported solely because these documents do not specify it, or that the author's experiment proves reliable remote alarm delivery. Revisit that combination only with explicit framework support or a reproducible, signed device test that records OS, entitlements, authorization, sound lookup, expiry, and failure behavior.

## Sources

- [Building apps is a system design exercise](https://blog.jacobstechtavern.com/p/apps-system-design) — public prototype section only.
