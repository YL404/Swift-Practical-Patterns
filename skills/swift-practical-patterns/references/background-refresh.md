# Background content refresh in SwiftUI apps

For short-lived background content refresh in apps using the SwiftUI app lifecycle on iOS/iPadOS 16+: configuration, registration, submission, rescheduling, and cancellation. The system schedules the work and decides when it runs, so time-critical execution or mandatory completion must not depend on this mechanism. Evaluate background `URLSession` for long file transfers and `BGProcessingTask` for heavy computation separately; this document does not cover other background mechanisms or platforms. [Apple's strategy guidance](https://developer.apple.com/documentation/backgroundtasks/choosing-background-strategies-for-your-app)

Official material verified on 2026-10-08. Examples were statically checked with Xcode 27.0, the iOS 27.0 SDK, Apple Swift 6.4, and the Swift 6 language mode; device behavior has not been verified.

## Configuration and registration

1. Enable **Background fetch** in the target's Background Modes; this corresponds to `fetch` in `UIBackgroundModes`.
2. List the permitted task identifiers in the target's `Info.plist` under the `BGTaskSchedulerPermittedIdentifiers` array, matching the code character for character.
3. Register one handler for each listed identifier, and finish registration before the end of the app launch sequence. In SwiftUI use `Scene.backgroundTask(_:action:)`, available from iOS 16. [Apple configuration guide](https://developer.apple.com/documentation/uikit/using-background-tasks-to-update-your-app), [SwiftUI modifier](https://developer.apple.com/documentation/swiftui/scene/backgroundtask(_:action:))

The official registration rules require each identifier to be registered exactly once; the system kills the app on a second registration. When using the SwiftUI path, have a single scene in the normal app launch path host the handler — the main `WindowGroup` in the example below. Other scenes can share the scheduling method without duplicating a modifier for the same identifier, and you should not register a second set of handlers through `BGTaskScheduler.register`. This setup advice follows the official single-registration rule; this document has not tested how SwiftUI treats the same modifier on repeated scenes, and it does not infer that every scene body re-evaluation registers again. [Apple registration rule](https://developer.apple.com/documentation/backgroundtasks/bgtaskscheduler/register(fortaskwithidentifier:using:launchhandler:))

Once `BGTaskSchedulerPermittedIdentifiers` is present, the legacy `application(_:performFetchWithCompletionHandler:)` and `setMinimumBackgroundFetchInterval(_:)` paths are disabled; do not mix the two refresh mechanisms. [Apple configuration guide](https://developer.apple.com/documentation/uikit/using-background-tasks-to-update-your-app)

## Registration and scheduling example

This is a compilable registration and scheduling example compatible with an iOS 16 deployment target. For newer targets, apply the migration notes below. `ContentProvider.refresh()` is deliberately a business placeholder stub: it currently neither fetches nor caches anything. When you wire it up, replace it with the app's real refresh, which must handle failure, cancellation, and persistence. Do not treat the stub's normal return as a successful refresh.

```swift
import BackgroundTasks
import SwiftUI

struct ContentProvider {
    let taskID = "com.example.app.fetch"

    func scheduleRefresh() {
        let request = BGAppRefreshTaskRequest(identifier: taskID)
        request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60)
        do {
            try BGTaskScheduler.shared.submit(request)
        } catch let error as BGTaskScheduler.Error {
            switch error.code {
            case .notPermitted:
                print("Check identifier and Background fetch configuration: \(error)")
            case .unavailable:
                print("Background refresh is unavailable: \(error)")
            case .tooManyPendingTaskRequests:
                print("Review pending task requests: \(error)")
            default:
                print("Scheduling failed: \(error)")
            }
        } catch {
            print("Scheduling failed: \(error)")
        }
    }

    func refresh() async {
        if Task.isCancelled { return }
        // Replace this stub with cancellation-aware refresh, failure handling, and persistence.
    }
}

@main
struct MyApp: App {
    @Environment(\.scenePhase) private var scenePhase
    private let provider = ContentProvider()

    var body: some Scene {
        WindowGroup { Text("MyApp") }
            .backgroundTask(.appRefresh(provider.taskID)) {
                provider.scheduleRefresh()
                await provider.refresh()
            }
            .onChange(of: scenePhase) { phase in
                if phase == .background { provider.scheduleRefresh() }
            }
    }
}
```

The example uses the single-parameter `onChange(of:perform:)` for iOS 16 compatibility; it carries a deprecation notice from iOS 17 onward, so use the zero- or two-parameter replacements when the deployment target allows. `BGTaskScheduler.submit(_:)` carries a deprecation notice from iOS 27; the new API is `submitTaskRequest(_:completionHandler:)` / `try await submitTaskRequest(_:)`. When migrating, also honor the official requirements about calling context and callback delay instead of only swapping the method name: the completion handler may run on an arbitrary queue after an arbitrary delay, and Apple directs you not to call the method from the main thread or performance-critical contexts. This document has not verified a complete integration with the new API. [Apple's new submission API](https://developer.apple.com/documentation/backgroundtasks/bgtaskscheduler/submittaskrequest(_:completionhandler:))

## Scheduling semantics and error diagnosis

- `earliestBeginDate` means only **no earlier than** this date; `nil` means no start delay. The 15-minute value in the example is not an execution period, and the system does not guarantee a launch at that time. [Apple date semantics](https://developer.apple.com/documentation/backgroundtasks/bgtaskrequest/earliestbegindate)
- Submitting a request again for the same identifier while the previous one is still unexecuted replaces it. At most 1 refresh and 10 processing requests can be scheduled at any time, and this quota is per app: multiple refresh identifiers do not give each one an independent refresh slot. [Apple submission semantics](https://developer.apple.com/documentation/backgroundtasks/bgtaskscheduler/submit(_:))
- Moving to the background is one trigger point for submitting a request. Apple's sample reschedules the next refresh at the start of the handler, before doing the current work; submitting a new request still provides no guarantee of continuous execution. [Apple handler sample](https://developer.apple.com/documentation/uikit/using-background-tasks-to-update-your-app)

Keep the error when a submission fails and distinguish the following causes instead of discarding it with `try?`:

| Error | Check first |
|---|---|
| `notPermitted` | whether `UIBackgroundModes` contains `fetch`; whether the identifier is registered and matches |
| `unavailable` | whether Background Refresh was turned off; whether the app runs on a Simulator that does not support background processing |
| `tooManyPendingTaskRequests` | the current pending requests and the type quota; identify requests that are safe to cancel before resubmitting |
| Other errors | keep the full error and check the current SDK documentation |

[Apple notPermitted](https://developer.apple.com/documentation/backgroundtasks/bgtaskscheduler/error/code/notpermitted), [Apple unavailable](https://developer.apple.com/documentation/backgroundtasks/bgtaskscheduler/error/code/unavailable), [Apple submission limits](https://developer.apple.com/documentation/backgroundtasks/bgtaskscheduler/submit(_:))

## Completion and cancellation

Apple's strategy guidance grants **up to 30 seconds** of background runtime for a refresh; do not treat it as a fixed budget available on every launch. In SwiftUI, the action closure returning is what completes the task, and the system then puts the app back into a suspended state; you do not call `BGTask.setTaskCompleted(success:)` manually — that belongs to the path where you register a `BGTask` launch handler directly. [Apple runtime guidance](https://developer.apple.com/documentation/backgroundtasks/choosing-background-strategies-for-your-app), [SwiftUI completion semantics](https://developer.apple.com/documentation/swiftui/scene/backgroundtask(_:action:))

If the time limit runs out before the action returns, the system cancels the task. The business implementation should use cancellation-aware APIs and check `Task.isCancelled` or `Task.checkCancellation()` between long-running steps; `withTaskCancellationHandler` responds to cancellation, it does not announce how many seconds remain. Save results in resumable increments, stop follow-up work promptly once cancelled, and do not stake required persistence on extra time after cancellation. [SwiftUI cancellation semantics](https://developer.apple.com/documentation/swiftui/scene/backgroundtask(_:action:))

Do not spawn `Task` or `Task.detached` work to escape this lifecycle. It follows from the documented rule — the app suspends once the closure returns — that spawned tasks cannot gain extra background execution time this way; the exact runtime timing of such spawned tasks has not been verified.

## Verification and boundaries

1. Type-check the final delivered code against the declared targets, then check the target configuration and identifiers. The example above was extracted verbatim and checked in the Swift 6 language mode: the iOS 16 target passed with no warnings; the iOS 27 target also passed, with deprecation notices for `submit` and the legacy `onChange`.
2. After submitting, call `await BGTaskScheduler.shared.pendingTaskRequests()` from an async context and check `identifier` and `earliestBeginDate`. It returns only unexecuted requests; seeing a request does not mean the system will run it on time. [Apple pending-requests API](https://developer.apple.com/documentation/backgroundtasks/bgtaskscheduler/getpendingtaskrequests(completionhandler:))
3. Log submission, actual wake-ups, work results, and cancellations on a physical device, and check that the app still recovers in the foreground when a refresh fails. Apple's official sample must be run on a physical device; the Simulator cannot prove real background scheduling. [Apple sample](https://developer.apple.com/documentation/backgroundtasks/refreshing-and-maintaining-your-app-using-background-tasks)

Verification here covered API documentation and compilation only: no request submission, no debug-simulated launch, and no on-device acceptance were executed. If a debug trigger is adopted, it proves only that the handler path ran for that one launch — not the system's scheduling frequency, the actual time budget, or overall reliability. This document does not include unverified private debug selectors.

The scheduling, registration, error, and cancellation conclusions rest on the Apple official material linked above.

## Sources

- [Scheduling and handling background app refresh in SwiftUI](https://nilcoalescing.com/blog/SchedulingAndHandlingBackgroundAppRefreshInSwiftUI/)
