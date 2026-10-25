# Preserve errors when bridging into tasks

Use this for ignored throwing-task warnings, task failure ownership, or storing an async outcome as a value with the Swift 6.4 toolchain. Prefer calling an async function directly when no independent task lifetime is needed.

## Choose who observes a throwing task

[SE-0520](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0520-discardableresult-task-initializers.md) adds a warning when an unstructured task with a non-`Never` failure is unused. Catch inside the task when its launch site owns failure reporting, or retain its handle and consume `try await task.value`. Storing a handle alone does not observe the error. `_ = Task { ... }` silences the diagnostic while intentionally discarding the result; it does not handle a failure or cancel the work. Apply the same decision to detached and immediate task variants.

Complete definitions for an application error and a caller that observes it:

```swift
enum LoadFailure: Error { case missing }

func loadCount() async throws(LoadFailure) -> Int {
    throw .missing
}

func observedCount() async throws -> Int {
    let task: Task<Int, any Error> = Task {
        try await loadCount()
    }
    return try await task.value
}
```

The checked Xcode 27.1 SDK still constrains throwing task initializers to `Failure == any Error`; the blog's `Task<String, URLError>` initializer example was rejected. Do not assume a proposal's typed initializer shape is available in the installed SDK. Inspect that SDK before exposing a specific task failure type. Owning this unstructured task also means deciding its cancellation policy; awaiting `value` does not turn it into a structured child.

## Store an outcome without creating another task

[SE-0530](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0530-async-result-support.md) adds an async `Result.init(catching:)` that evaluates its closure in the current task. Use it when an API needs a success/failure value, rather than creating an unstructured task just to obtain a result:

```swift
func capturedCount() async -> Result<Int, LoadFailure> {
    await Result { () async throws(LoadFailure) -> Int in
        try await loadCount()
    }
}
```

The explicit `throws(LoadFailure)` annotation avoids widening the closure's error type to `any Error` in the checked compiler. The closure retains the caller's isolation. A thrown cancellation error is also a failure value when the closure's error type permits it; callers that must propagate cancellation need to handle that case. This initializer does not automatically check cancellation before evaluating the closure. With an older toolchain, use explicit `do`/`catch` to construct `.success` or `.failure`.

## Evidence and limits

Proposals checked 2026-10-25. Exact definitions typechecked with Swift 5/6 language modes, complete strict concurrency, nonisolated default isolation, no extra feature flags, and warnings as errors using Xcode 27.1 (27A9275), Apple Swift 6.4 on macOS 27.0.1 arm64. Local execution observed the application error through `value`, captured success/failure, and captured cancellation with `Result`. Separate compiler checks diagnosed an unused throwing task and accepted explicit discard and a nonthrowing task. Older toolchains/runtimes, UI lifecycle, network work, and automatic cancellation propagation were not tested.

## Sources

- [Swift 6.4: What's New in Concurrency](https://www.avanderlee.com/concurrency/swift-6-4-whats-new-in-concurrency/) (2026-06-21)
