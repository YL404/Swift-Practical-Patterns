# In-app translation in SwiftUI

Use this guidance to choose the system translation UI or implement a custom translation flow, repeat requests, and handle language assets. It covers text translation on iOS/iPadOS and macOS, rather than app localization.

Official documentation verified 2026-10-09. Validation used Xcode 27.0, Apple Swift 6.4, Swift 6 language mode, and the iOS/macOS 27 SDKs. The examples were typechecked; no translation, download UI, or device behavior was executed.

## Choose the integration and deployment target

| Need | API | Minimum version |
|---|---|---|
| Let the system present a translation of a short text | `translationPresentation(isPresented:text:attachmentAnchor:arrowEdge:replacementAction:)` | iOS/iPadOS 17.4, macOS 14.4 |
| Translate into your own UI, check languages, or translate batches | `translationTask(_:action:)`, `TranslationSession`, `LanguageAvailability` | iOS/iPadOS 18, macOS 15 |
| Translate outside a view with languages already installed | `TranslationSession(installedSource:target:)` | iOS/iPadOS 26, macOS 26 |

The direct initializer requires an explicit source language and cannot request language downloads. Creating it does not ensure a later translation succeeds; translation throws if the required languages are missing. Use a view-provided session when permission/download UI is needed. The view-driven session APIs also support Mac Catalyst 26+, while the presentation modifier is unavailable there; Catalyst was not compiled in this verification. [Presentation modifier](https://developer.apple.com/documentation/swiftui/view/translationpresentation(ispresented:text:attachmentanchor:arrowedge:replacementaction:)), [TranslationSession](https://developer.apple.com/documentation/translation/translationsession), [Direct initializer](https://developer.apple.com/documentation/translation/translationsession/init(installedsource:target:))

## System UI and replacement

Set `isPresented` to `true` to show the translation UI. Supplying `replacementAction` adds a “Replace with Translation” button; the closure receives a translated string when the user selects that button. It is not an automatic translation-completion callback. Without the closure, the UI displays the translation without returning it to your app. Changing `text` after presentation has no effect on the displayed translation. iOS ignores `arrowEdge`; Apple recommends short input, roughly a few sentences or phrases. [Apple modifier semantics](https://developer.apple.com/documentation/swiftui/view/translationpresentation(ispresented:text:attachmentanchor:arrowedge:replacementaction:))

This is a compilable view definition; a containing app must present it. `Text(verbatim:)` displays the input content without treating it as an app localization key.

```swift
import SwiftUI
import Translation

@available(iOS 17.4, macOS 14.4, *)
struct SystemTranslationView: View {
    @State private var showTranslation = false
    @State private var text = "Hallo, Welt!"

    var body: some View {
        VStack {
            Text(verbatim: text)
            Button("Translate") { showTranslation = true }
        }
        .translationPresentation(isPresented: $showTranslation, text: text) { translated in
            text = translated
        }
    }
}
```

## Custom sessions, repeat requests, and result validity

Keep an optional `TranslationSession.Configuration` in state. A non-`nil` initial configuration runs the action when the view appears; changing a non-`nil` configuration runs it again. For new text with the same language pair, call `configuration.invalidate()`. When the language pair changes, assign a configuration for the new pair instead. Changing the input text alone is not a configuration change. [translationTask](https://developer.apple.com/documentation/swiftui/view/translationtask(_:action:)), [invalidate](https://developer.apple.com/documentation/translation/translationsession/configuration/invalidate())

For this view-driven pattern, await translation inside the provided action and retain the configuration rather than a session to reuse from later button tasks. Apple documents a `fatalError` if a view-provided session is used after its view disappears or after its configuration changes. Language equality is not a session-validity check. This lifetime restriction concerns view-provided sessions; the direct initializer has the separate installed-language requirements above. [Apple session lifetime](https://developer.apple.com/documentation/swiftui/view/translationtask(_:action:))

Do not assume a new configuration run immediately stops every old operation. The checked modifier documentation does not specify that guarantee. Snapshot the request, reject obsolete results, and keep error handling. The compilable view below uses a fixed German-to-English pair, disables input and repeated taps while a request is pending, and invalidates the request when the view disappears. The counter prevents an old response or error from updating a later request; it does not claim to cancel the translation engine.

```swift
import SwiftUI
import Translation

@available(iOS 18.0, macOS 15.0, *)
struct CustomTranslationView: View {
    @State private var sourceText = "Hallo, Welt!"
    @State private var targetText: String?
    @State private var errorMessage: String?
    @State private var isTranslating = false
    @State private var requestID = 0
    @State private var configuration: TranslationSession.Configuration?

    var body: some View {
        VStack {
            TextField("German text", text: $sourceText)
                .disabled(isTranslating)
            Button("Translate to English") {
                requestID += 1
                isTranslating = true
                targetText = nil
                errorMessage = nil
                if configuration == nil {
                    configuration = .init(
                        source: Locale.Language(identifier: "de"),
                        target: Locale.Language(identifier: "en")
                    )
                } else {
                    configuration?.invalidate()
                }
            }
            .disabled(isTranslating || sourceText.isEmpty)
            if let targetText { Text(verbatim: targetText) }
            if let errorMessage { Text(verbatim: errorMessage) }
        }
        .translationTask(configuration) { session in
            let id = requestID
            let input = sourceText
            do {
                try Task.checkCancellation()
                let response = try await session.translate(input)
                try Task.checkCancellation()
                await MainActor.run {
                    guard id == requestID else { return }
                    targetText = response.targetText
                    isTranslating = false
                }
            } catch {
                await MainActor.run {
                    guard id == requestID else { return }
                    isTranslating = false
                    if !Task.isCancelled {
                        errorMessage = error.localizedDescription
                    }
                }
            }
        }
        .onChange(of: sourceText) {
            targetText = nil
            errorMessage = nil
        }
        .onDisappear {
            requestID += 1
            isTranslating = false
            configuration = nil
        }
    }
}
```

If your app allows editing, changing languages, or starting a new request while work is pending, adapt the request identity to all those inputs and control configuration changes. This example verifies compilation, not the framework's cancellation or navigation timing. `TranslationSession.cancel()` is available from iOS/macOS 26; it attempts to stop ongoing session work, and future requests on that cancelled session fail. Gate it for older targets. [cancel](https://developer.apple.com/documentation/translation/translationsession/cancel())

## Language support and downloads

Use `LanguageAvailability.supportedLanguages` for available language choices and `await status(from:to:)` for the selected pair. An individually supported language does not imply every pairing is supported. The statuses mean:

- `.installed`: the pair is supported and its assets are ready.
- `.supported`: the pair is supported, but assets still need downloading/installing.
- `.unsupported`: do not offer that pairing. Same-language variants such as en-GB to en-US are unsupported.

[Supported languages](https://developer.apple.com/documentation/translation/languageavailability/supportedlanguages), [Pair status](https://developer.apple.com/documentation/translation/languageavailability/status(from:to:)), [Status cases](https://developer.apple.com/documentation/translation/languageavailability/status)

A `nil` source in the configuration lets the framework detect the language and ask the user if detection fails. A `nil` target lets it choose from the user's preferred languages. Prefer the `Locale.Language` values returned by `supportedLanguages` when building a picker; the framework attempts to match other values, but that match can fail. If you need a preflight check using text, `status(for:to:)` is throwing and can fail to identify the source; the documentation recommends a sample of at least 20 characters. [Configuration initializer](https://developer.apple.com/documentation/translation/translationsession/configuration/init(source:target:)), [Text-based status](https://developer.apple.com/documentation/translation/languageavailability/status(for:to:))

Translation runs on the device. A view-provided session can ask permission to download missing language assets during `translate(_:)`; this can take minutes. Installed assets avoid that download requirement, but an availability check does not guarantee the following call succeeds. Handle download refusal, progress-view dismissal, invalid session, and translation errors. Dismissing download progress throws `CocoaError.userCancelled` while the download continues in the background. [TranslationSession](https://developer.apple.com/documentation/translation/translationsession), [Translation and errors](https://developer.apple.com/documentation/translation/translationsession/translate(_:)-4m20l)

`prepareTranslation()` is optional: it requests language-download permission without translating. It needs a known source language; a `nil` source throws `unableToIdentifyLanguage` because there is no sample text. It returns without prompting if assets are already installed **or downloading**, so a return is not a guarantee that downloads have finished. Keep error handling on the actual translation. [Preparation semantics](https://developer.apple.com/documentation/translation/translationsession/preparetranslation())

## Batch requests and ordering

Batch strings must share a source language. `translations(from:)` returns all responses in request order. `translate(batch:)` yields responses incrementally, potentially in a different order; assign each `TranslationSession.Request` a `clientIdentifier` and use the returned identifier to correlate it. If the displayed collection can change, use durable item IDs and a request snapshot rather than applying responses to current array offsets. This correlation rule is an implementation recommendation based on the documented ordering difference. [All-at-once translation](https://developer.apple.com/documentation/translation/translationsession/translations(from:)), [Incremental translation](https://developer.apple.com/documentation/translation/translationsession/translate(batch:))

## Verification and remaining boundaries

The two view definitions above were extracted verbatim and typechecked with `swiftc -typecheck -swift-version 6`, the iOS/macOS 27 SDKs, and arm64 deployment targets iOS 17.4/18 and macOS 14.4/15. Their availability annotations permit the presentation view on the older target while keeping the custom view gated. All checks passed without diagnostics. No app was launched, and no translation/session behavior was inferred from these checks.

Apple's sample requires macOS or physical iOS devices and states that it does not translate in iOS/iPadOS simulators. The article author's observation that the system overlay worked in a simulator does not establish current support. For app acceptance, verify supported/unsupported pairs, missing assets, download refusal/dismissal, repeated same-pair requests, navigation away while pending, and batch correlation on macOS or a physical iOS device. [Apple sample and device requirements](https://developer.apple.com/documentation/translation/translating-text-within-your-app)

API and lifetime conclusions use the Apple sources above; retaining a session for later reuse is not adopted as the recommended repeat-request pattern.

## Sources

- [SwiftUI: In App Translation. From Simple One-Liner to Beyond!](https://levelup.gitconnected.com/swiftui-in-app-translation-from-simple-one-liner-to-beyond-a5ddb9158396)
