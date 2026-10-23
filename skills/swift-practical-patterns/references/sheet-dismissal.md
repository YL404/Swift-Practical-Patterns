# Protecting unsaved sheet edits

Use when a sheet must keep a draft until the user explicitly discards it. The native policy below requires iOS/iPadOS 15+ or macOS 12+; detent selection requires iOS/iPadOS 16+ or macOS 13+. Documentation checked 2026-10-23.

## Separate blocking, confirmation, and dismissal

`interactiveDismissDisabled(hasUnsavedChanges)` blocks nonprogrammatic dismissal. It neither asks for confirmation nor stops code from calling `dismiss()` or clearing the presentation binding. Apply the modifier inside the sheet hierarchy and make every app-controlled close path follow the draft policy. [Apple: interactiveDismissDisabled](https://developer.apple.com/documentation/swiftui/view/interactivedismissdisabled(_:))

For an explicit Cancel action, ask before discarding a changed draft, then dismiss only after confirmation. Read `dismiss` in the sheet content's environment, where it targets that presentation. The presenting view's dismiss action can target a different container. [Apple: DismissAction](https://developer.apple.com/documentation/swiftui/dismissaction)

This complete sheet-content declaration starts a new, empty text draft. It demonstrates cancellation only; it is not a persistence editor or a save implementation.

```swift
import SwiftUI

struct NewDraftSheet: View {
    @Environment(\.dismiss) private var dismiss
    @State private var text = ""
    @State private var confirmDiscard = false

    private var hasUnsavedChanges: Bool { !text.isEmpty }

    var body: some View {
        VStack {
            TextField("Draft", text: $text)
            Button("Cancel") {
                if hasUnsavedChanges {
                    confirmDiscard = true
                } else {
                    dismiss()
                }
            }
            .confirmationDialog(
                "Discard draft?",
                isPresented: $confirmDiscard,
                titleVisibility: .visible
            ) {
                Button("Discard", role: .destructive) { dismiss() }
                Button("Keep Editing", role: .cancel) { }
            }
        }
        .interactiveDismissDisabled(hasUnsavedChanges)
    }
}
```

For an existing document, compare the draft with its saved baseline rather than checking for empty text. Keep edits in separate draft state until save succeeds; closing a sheet does not undo writes already made to a live model or autosaved context. [Apple's editor sample](https://developer.apple.com/documentation/swiftdata/adding-and-editing-persistent-data-in-your-app) uses local editable values for this reason. A save failure must leave the draft available. The example provides no protection against a presenter independently clearing its binding.

## A selected detent is not a dismissal-attempt event

`presentationDetents(_:selection:)` exposes a resizable sheet's current detent and allows programmatic selection. The selected value must belong to the supplied set. A selection change alone therefore does not identify a user close attempt. [Apple: detent selection](https://developer.apple.com/documentation/swiftui/view/presentationdetents(_:selection:))

Treat a small detent used as a confirmation threshold as an app-specific resize interaction, not as interception of every system dismissal attempt. The article's snap-back implementation is not validated here: selected and default detents may coincide, programmatic changes can also reach the threshold, and a blocked drag need not select a different detent. Validate those cases, supported detents after adaptation, repeated gestures, and dialog timing on the target runtime before adopting it. An unstructured `Task` around the callback is not an ordering guarantee.

If the requirement is specifically to react to a refused interactive dismissal, UIKit has a separate [presentationControllerDidAttemptToDismiss](https://developer.apple.com/documentation/uikit/uiadaptivepresentationcontrollerdelegate/presentationcontrollerdidattempttodismiss(_:)) callback. This establishes the event's meaning; it does not establish a safe way to replace a SwiftUI-owned presentation controller delegate. No bridge implementation is provided or verified here.

## Validation scope

The declaration was typechecked with Xcode 27.1 / Apple Swift 6.4, Swift 6, complete strict concurrency and warnings as errors, for iOS 15 and macOS 12 deployment with nonisolated and MainActor default isolation. No sheet was presented. Gesture prevention, dialogs, focus, accessibility, adaptation, older runtimes, and UIKit delegate integration remain untested; compilation does not validate the article's threshold behavior.

## Sources

- https://livsycode.com/swiftui/intercepting-swiftui-sheet-dismissal/
