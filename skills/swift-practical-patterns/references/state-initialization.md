# Observable state: construction, seeds, and replacement

Use this when a view-owned observable model is repeatedly constructed, needs a parent-supplied initial value, or must survive repeated appearances without losing edits. Scope: iOS 17 / macOS 14 and corresponding Observation-capable platforms. Verified 2026-10-23 with Xcode 27.1 / Swift 6.4; older toolchains use the property-wrapper behavior described below.

## Distinguish stored state from construction work

SwiftUI retaining the original state does not mean every expression used to construct a new view is lazy.

- With Xcode 27, prefer a private inline default such as `@State private var model = EditorDraft(title: "")` when no parent input is needed. The macro defers the default instead of allocating a redundant instance on each reconstruction. This is a build-toolchain change, back-deployed to the Observation-era systems; an `if #available(iOS 27, *)` check is not the selector. [Apple State macro](https://developer.apple.com/documentation/swiftui/state()), [WWDC26 data-flow discussion](https://developer.apple.com/videos/play/wwdc2026/269/?time=1313)
- An expression explicitly evaluated in `init`, including `model = EditorDraft(title: title)` or `State(wrappedValue: EditorDraft(title: title))`, still runs each time that initializer runs. Retaining the first object does not remove those allocations. Use this for an intentional inexpensive seed, not as a way to defer expensive construction.
- With Xcode 26 or earlier, even the inline property-wrapper default can repeat construction. Deferred optional state is a fallback when construction is costly; the current macro removes that need for the private inline-default case. Older-compiler behavior was not executed in this verification.

## Supply one initial value in init

For an intentional one-time seed, omit the declaration default and initialize ordinary stored properties before assigning to state. This complete view declaration uses a cheap value seed:

```swift
import SwiftUI

struct SeededCounter: View {
    let name: String
    @State private var count: Int

    init(name: String, initialCount: Int) {
        self.name = name
        self.count = initialCount
    }

    var body: some View {
        Stepper("\(name): \(count)", value: $count)
    }
}
```

Distinguish two cases described in [TN3211](https://developer.apple.com/documentation/technotes/tn3211-resolving-swiftui-source-incompatibilities-for-state-and-contentbuilder). For its use-before-initialization diagnostic with state lacking a default, putting ordinary-property assignments first is Apple's documented remedy. Assigning in `init` when the state declaration already has a default is separately unsupported: remove that default rather than relying on assignment order. Both minimal cases compiled in the verified 27.1 toolchain; the installed double-default view nevertheless read its declaration value, not the value assigned in `init`. Do not make the presence of a compiler error the correctness test.

The seed is used for the view identity's initial state. Later arguments do not replace child-owned state. This is appropriate for an independently edited draft; use ordinary inputs or an externally owned observable model when later parent updates must propagate.

## Defer a parent-configured model without rebuilding it on every appearance

When construction needs runtime inputs and should happen only if the current state is empty, keep optional state and guard the lifecycle action. These complete model and view declarations implement synchronous draft creation; they do not implement asynchronous data loading.

```swift
import SwiftUI
import Observation

@MainActor
@Observable
final class EditorDraft {
    var title: String

    init(title: String) {
        self.title = title
    }
}

struct DeferredDraftEditor: View {
    let initialTitle: String
    @State private var draft: EditorDraft? = nil

    var body: some View {
        Group {
            if let draft {
                DraftFields(draft: draft)
            } else {
                ProgressView()
            }
        }
        .task { @MainActor in
            guard draft == nil else { return }
            draft = EditorDraft(title: initialTitle)
        }
    }
}

private struct DraftFields: View {
    @Bindable var draft: EditorDraft

    var body: some View {
        TextField("Title", text: $draft.title)
    }
}
```

The seed is read when this task first creates the draft, not when the view struct is constructed. Later `initialTitle` changes deliberately do not overwrite edits. The guard prevents another invocation from replacing a non-nil draft. Resetting it to nil permits creation on a later task invocation; it does not itself restart `.task`. Destroying the view's identity also destroys its state. Unwrapping the object in a child with `@Bindable` preserves normal projected bindings without inventing a `Binding(get:set:)` for optional storage. [State object sharing](https://developer.apple.com/documentation/swiftui/state())

Attach the task to the stable outer content, not only to the empty-state branch that disappears when the model is assigned. A view task follows appearance/removal and identity; it is not an app-wide or identity-wide once token. The check and assignment here are on `MainActor` with no suspension between them. Adding an `await` requires a separate policy for in-flight work, cancellation, and stale results. Moving a synchronous constructor into `.task` does not make that constructor run off the main actor. [View.task lifecycle](https://developer.apple.com/documentation/swiftui/view/task(name:priority:file:line:_:))

## Decide what an input change means

Do not replace every parent-configured model merely because an input changed. Choose whether the existing draft survives, a model operation reloads data, or a new editing session starts:

- To preserve edits for the current view identity, retain the one-time seed semantics above.
- To update content within the same identity, use the new input in the model's update/load operation. `task(id:)` restarts its action when the ID changes; it does not reset state automatically. Cancellation is cooperative, so asynchronous results and errors need an appropriate stale-write policy. [View.task(id:)](https://developer.apple.com/documentation/swiftui/view/task(id:name:priority:file:line:_:))
- To intentionally replace the whole editing session, a caller can change `.id(sessionID)`. This resets the view's state, including local edits; it is broader than restarting one task. [View.id](https://developer.apple.com/documentation/swiftui/view/id(_:))

## Validation scope

The two code blocks compiled for macOS 14 and iOS 17 deployment with the macOS 27.0 and iPhoneOS 27.1 SDKs, Apple Swift 6.4 (swiftlang-6.4.0.34.1), Xcode 27.1 (27A9275). Both Swift 5 and Swift 6 language modes used complete strict concurrency, explicit nonisolated default isolation, and no extra feature flags. UI models and the task action have explicit `MainActor` isolation.

A separate Swift 6 AppKit/SwiftUI hosting probe ran on macOS 27.0.1 arm64. After three parent revisions under the same identity, the private inline default and guarded deferred model each had one construction; the direct-init and explicit-State models each had four. All retained the original edited object. The deferred probe deliberately restarted `task(id:)` four times to check the guard; this was not an actual disappearance/reappearance test. Changing the outer identity created fresh models with the latest dynamic seed. Separate installed views confirmed that the no-default reversed assignment order compiled and produced 42, while a declaration default of 0 plus an init assignment of 42 compiled but produced 0.

These are local construction/state checks, not timing benchmarks or validation of network loads, navigation, multiwindow sharing, editing focus, or older runtimes. No older compiler, iOS execution, simulator, or device behavior was tested. The deferred example's initializer must remain cheap enough for `MainActor`; no background execution claim is made.

## Sources

- https://nilcoalescing.com/blog/InitializingObservableClassesWithinTheSwiftUIHierarchy/
- https://nilcoalescing.com/blog/InitializingObservableClassesWithTheStateMacroInXcode27/
