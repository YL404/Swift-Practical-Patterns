# Reusable Foundation Models profile policies

Use this for request-by-request model configuration and reusable profile modifiers on iOS/iPadOS 27+ or macOS 27+, with the Xcode 27 SDK or later. Check model availability separately; a profile does not grant PCC entitlements, provider credentials, or network access.

## Put required overrides inside defaults

[`DynamicProfileModifier`](https://developer.apple.com/documentation/foundationmodels/languagemodelsession/dynamicprofilemodifier) wraps existing profile content. For conflicting values, the modifier nearest the `Profile` wins over outer defaults. Call-site `GenerationOptions` win over matching generation settings; they do not replace the model. Lifecycle callbacks accumulate rather than replacing each other. [Composition and precedence](https://developer.apple.com/documentation/foundationmodels/composing-dynamic-sessions-with-instructions-and-profiles)

Complete typecheckable definitions; the shared state below is for serial use, with no tools or concurrent state mutation. The response budget is a positive application-selected value, not a recommended universal limit:

```swift
import FoundationModels
import Observation

@Observable
final class AssistantPolicy {
    var usesLocalModel = false
}

struct LocalBudget: LanguageModelSession.DynamicProfileModifier {
    let enabled: Bool
    let responseTokens: Int

    init(enabled: Bool, responseTokens: Int) {
        precondition(responseTokens > 0)
        self.enabled = enabled
        self.responseTokens = responseTokens
    }

    func body(content: Content) -> some LanguageModelSession.DynamicProfile {
        if enabled {
            content
                .model(SystemLanguageModel.default)
                .temperature(0.3)
                .maximumResponseTokens(responseTokens)
        } else {
            content
        }
    }
}

struct AssistantProfile<Remote: LanguageModel>:
    LanguageModelSession.DynamicProfile {
    let policy: AssistantPolicy
    let remote: Remote
    let responseTokens: Int

    var body: some LanguageModelSession.DynamicProfile {
        Profile {
            Instructions("Answer briefly using plain language.")
        }
        .modifier(LocalBudget(
            enabled: policy.usesLocalModel,
            responseTokens: responseTokens
        ))
        .model(remote)
        .temperature(0.7)
    }
}
```

Create the policy once and pass the same object into the profile used by `LanguageModelSession(profile:)`. Change it between requests. Profile evaluation takes the current state for the next model request; a copied Boolean at initialization would not observe later changes to an unrelated caller variable. Moving `LocalBudget` outside `.model(remote)` makes the remote model win while the uncontested token cap still applies. Keep access serialized; `@Observable` supplies observation, not synchronization.

[`maximumResponseTokens`](https://developer.apple.com/documentation/foundationmodels/generationoptions/maximumresponsetokens) can stop generation early without an error. A successful call therefore does not prove the answer is complete. Validate required output and evaluate the chosen budget; do not use a small cap as a general structured-output strategy. Model availability and tool network activity still need their own handling.

## Transform a request's history without deleting the transcript

[`historyTransform(_:)`](https://developer.apple.com/documentation/foundationmodels/languagemodelsession/dynamicprofile/historytransform(_:)) changes the entries sent for that request, preserving the session's global transcript. Use this distinction when switching to a smaller-context model. An entry count is neither a token budget nor a count of conversation turns. Preserve tool-call/output relationships and required context when selecting history; a blind suffix is not a general tool-history implementation. Validate the transformed conversation against each provider. This modifier does not redact saved history.

## Evidence and limits

Official pages and SDK declarations checked 2026-10-25. The definitions passed Swift 5/6 typechecking with warnings as errors, complete strict concurrency, nonisolated default isolation, and no extra feature flags using Xcode 27.1 (27A9275), Apple Swift 6.4, iPhoneOS 27.1 and macOS 27.0 SDKs, targeting iOS/macOS 27. No model request, state-switch behavior, transcript transformation, entitlement, quota, output-quality, cache, or latency test was run. No particular retry or billing-saving guarantee is established.

## Sources

- [Building a custom DynamicProfileModifier in Foundation Models](https://artemnovichkov.com/blog/building-a-custom-dynamic-profile-modifier-in-foundation-models) (2026-07-17)
