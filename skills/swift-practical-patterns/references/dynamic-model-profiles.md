# Reusable Foundation Models profile policies

Use this for request-by-request model configuration and reusable profile modifiers on iOS/iPadOS 27+ or macOS 27+, with the Xcode 27 SDK or later. The standalone token-budget screening helper requires iOS/iPadOS 26.4+ or macOS 26.4+. Check model availability separately; a profile does not grant PCC entitlements, provider credentials, or network access.

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

## Screen the local token budget before selecting a model

Use [`SystemLanguageModel.tokenCount(for:)`](https://developer.apple.com/documentation/foundationmodels/systemlanguagemodel/tokencount(for:)) on iOS/iPadOS 26.4+ or macOS 26.4+ to count the actual prompt and instructions. Read [`contextSize`](https://developer.apple.com/documentation/foundationmodels/systemlanguagemodel/contextsize) from the selected system model instead of hardcoding a context window. Check availability first. A character-count heuristic is not a safe replacement when counting fails, especially across languages. [Context accounting](https://developer.apple.com/documentation/foundationmodels/managing-the-context-window)

Complete screening helper for a **fresh, single-turn, text-only session without tools or a generated schema**. The caller supplies a positive response reserve and a nonnegative framing margin chosen for its workload. Counting the two components separately is not an exact encoded-request fit guarantee; no universal margin is established here:

```swift
@available(iOS 26.4, macOS 26.4, *)
func hasLocalSummaryBudget(
    for prompt: Prompt,
    instructions: Instructions,
    responseReserve: Int,
    framingReserve: Int,
    model: SystemLanguageModel = .default
) async throws -> Bool {
    precondition(responseReserve > 0 && framingReserve >= 0)
    try Task.checkCancellation()
    guard model.isAvailable else { return false }

    let promptTokens = try await model.tokenCount(for: prompt)
    let instructionTokens = try await model.tokenCount(for: instructions)
    try Task.checkCancellation()

    let capacity = model.contextSize
    guard responseReserve <= capacity else { return false }
    let afterResponse = capacity - responseReserve
    guard framingReserve <= afterResponse else { return false }
    let inputBudget = afterResponse - framingReserve
    return instructionTokens <= inputBudget
        && promptTokens <= inputBudget - instructionTokens
}
```

Include `import FoundationModels`. When the helper returns `true`, use the same model, prompt, and instructions in the new local session. Set the request's `maximumResponseTokens` no higher than the response reserve, retaining the incomplete-output limitation above. A `false` result means unavailable or over this screening budget; it does not authorize an upload. Counting errors and cancellation propagate rather than silently selecting a cloud provider. Still handle context-limit and generation errors at the request boundary. A successful preflight is neither a reservation nor a guarantee of model readiness or output quality. [Availability and single-turn sessions](https://developer.apple.com/documentation/foundationmodels/generating-content-and-performing-tasks-with-foundation-models)

Do not reuse this initial-input Boolean as a multiturn routing decision. Prior prompts and responses, current instructions, tool definitions/calls/outputs, and generated schemas consume context too. Recalculate for the next request's actual profile and effective history; use the [transcript-entry counting overload](https://developer.apple.com/documentation/foundationmodels/systemlanguagemodel/tokencount(for:)-8ywjk) where applicable and account for active instructions/tools/schema without double counting. `history` in a dynamic profile excludes its dynamic instructions. A synchronous profile `body` can consume a prepared decision, but cannot perform the asynchronous counts itself. [Profile evaluation and history](https://developer.apple.com/documentation/foundationmodels/composing-dynamic-sessions-with-instructions-and-profiles)

Treat remote routing as a separate application decision: verify data-sharing policy, provider configuration, model capabilities, and the remote context budget before sending anything. Do not use a catch-all local-error handler as permission to upload. If remote processing is disallowed, offer an unavailable/too-large result or split the task into local sessions; preserve any required state explicitly. This is implementation policy, not an automatic feature of `DynamicProfile`. No remote adapter or fallback implementation is provided here.

## Evidence and limits

Official pages and SDK declarations checked 2026-10-25. The screening helper additionally passed Swift 6 typechecking with nonisolated and MainActor default isolation, complete strict concurrency, and warnings as errors for iOS/macOS 26.4 using the SDKs below; unguarded use was rejected for iOS/macOS 26.0. This verifies signatures and availability, not token counts or fit. No token-count call or preflight behavior test was run.

The complete definitions passed Swift 5/6 typechecking with warnings as errors, complete strict concurrency, nonisolated default isolation, and no extra feature flags using Xcode 27.1 (27A9275), Apple Swift 6.4, iPhoneOS 27.1 and macOS 27.0 SDKs, targeting iOS/macOS 27.

No model request, state-switch behavior, transcript transformation, entitlement, quota, output-quality, cache, or latency test was run. No particular retry or billing-saving guarantee is established.

## Sources

- [Building a custom DynamicProfileModifier in Foundation Models](https://artemnovichkov.com/blog/building-a-custom-dynamic-profile-modifier-in-foundation-models) (2026-07-17)
- [Apple Foundation Models: Hybrid AI with Dynamic Profiles](https://peterfriese.dev/blog/2026/hybrid-ai-apple-foundation-models-gemini) (2026-08-18)
