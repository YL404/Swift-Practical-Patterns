# SwiftData failed saves, rollback limits, and context isolation

Use this reference when handling `ModelContext.save()` errors, deciding between `rollback()` and an isolated `ModelContext`, preventing duplicate-insert rollback corruption, or testing save failures in parallel Swift Testing suites. Scope: SwiftData on iOS/iPadOS 17+ and macOS 14+ with the built-in store. For CloudKit sync identity and schema rules, see [swiftdata-sync.md](swiftdata-sync.md); for list fetch sizing and schema splits, see [swiftdata-fetch-shape.md](swiftdata-fetch-shape.md).

Apple documentation and SDK interfaces verified 2026-11-24 with Xcode 27.2 (27B52) / Apple Swift 6.4.2 on macOS 27.2.

## Do not assume failed saves or `transaction(block:)` clean up pending changes

`ModelContext.save()` writes all pending inserts, updates, and deletes in that context to the persistent store, and `ModelContext.transaction(block:)` runs its closure and then saves pending changes. Neither API automatically discards uncommitted changes when an error is thrown. [Apple: save()](https://developer.apple.com/documentation/swiftdata/modelcontext/save()), [Apple: transaction(block:)](https://developer.apple.com/documentation/swiftdata/modelcontext/transaction(block:))

- **Unreverted changes linger in the context:** in local macOS 27.2 checks, both a thrown error inside `transaction(block:)` and a `save()` validation failure (`NSCocoaErrorDomain Code=1580` from `@Relationship(minimumModelCount: 1)`) left `context.hasChanges == true` with the uncommitted changes still registered.
- **Shared contexts propagate failure across screens:** `ModelContainer.mainContext` has `autosaveEnabled == true` by default ([Apple: autosaveEnabled](https://developer.apple.com/documentation/swiftdata/modelcontext/autosaveenabled)). If one screen leaves a failed insert, edit, or delete in a shared context, a later save from an unrelated screen either fails on the stale invalid change or silently commits a previously rejected change (such as a failed delete once a transient store error clears).

## Isolate risky or multi-step writes in a dedicated `ModelContext`

`ModelContext.rollback()` discards **all** pending inserts and deletes, restores changed models to their most recent committed state, and clears the undo stack ([Apple: rollback()](https://developer.apple.com/documentation/swiftdata/modelcontext/rollback())). That makes in-place rollback unsafe in a shared context where another screen may hold unsaved draft edits, and it also has a critical store-level limit observed locally:

- **Per-context `DefaultStore` poisoning after real `save()` validation failures (observed on macOS 27.2):** when `save()` actually threw a `DefaultStore` validation error (`NSCocoaErrorDomain Code=1580`, tested on both in-memory and on-disk SQLite stores), calling `processPendingChanges()` followed by `rollback()` cleared `context.hasChanges` to `false` on the `ModelContext`, **yet every subsequent unrelated `save()` on that same `ModelContext` still threw the same validation error for the rolled-back model**. Creating a separate `ModelContext(container)` saved cleanly on the same container (`notes` persisted, invalid `Team` absent).
- **Use a short-lived context for failure-prone writes:** perform multi-step mutations, imports, or writes that can violate store constraints inside a fresh `ModelContext(container)` with `autosaveEnabled = false` (for example inside a `@ModelActor` or a helper that fetches by `PersistentIdentifier`). If `save()` throws, discard that context instead of trying to repair and reuse it.

Complete SwiftData helper definition showing an isolated write context on a `ModelContainer`. The caller supplies an existing container and handles thrown errors; UI refresh relies on the store update observed by other contexts rather than mutating a shared `mainContext` directly.

```swift
import SwiftData

@Model
final class JournalNote {
    var title: String

    init(title: String) {
        self.title = title
    }
}

func persistNote(title: String, in container: ModelContainer) throws {
    let writeContext = ModelContext(container)
    writeContext.autosaveEnabled = false
    writeContext.insert(JournalNote(title: title))
    try writeContext.save()
}
```

## Guard `rollback()` when reusing a single-write context

When an app intentionally commits immediately after every write on one context and rolls back **pre-save** failures (such as application validation errors thrown before `DefaultStore.save(_:)` executes, or an explicit cancel action on an isolated draft context), follow two guards:

1. **Coalesce before rolling back:** call `context.processPendingChanges()` immediately before `context.rollback()`. Apple documents `processPendingChanges()` to coalesce pending changes and record them for the undo manager ([Apple: processPendingChanges()](https://developer.apple.com/documentation/swiftdata/modelcontext/processpendingchanges())). The source article observed on its iOS test host that rolling back an unprocessed delete left the restored model crashing on the next property read unless `processPendingChanges()` ran first. That crash did **not** reproduce in local macOS 27.2 probes (both with and without `processPendingChanges()`, across in-memory and SQLite stores, with and without cascade child relationships and `UndoManager`), and Apple Forums separately report framework crashes inside `rollback()` after deleting parent models with child relationships where the documented workaround is deleting children first or using a separate context. Treat `processPendingChanges()` before `rollback()` as a low-cost defensive step, not as a substitute for context isolation on complex graph deletes or real `DefaultStore` save errors.
2. **Never re-insert an already-managed model (`model.modelContext != nil`):** `ModelContext.insert(_:)` is **not** a harmless no-op on a model that already belongs to the context ([Apple: insert(_:)](https://developer.apple.com/documentation/swiftdata/modelcontext/insert(_:))). In local macOS 27.2 probes (both in-memory and on-disk SQLite, including freshly fetched models), modifying an existing persisted model, calling `context.insert(model)` again, and then calling `rollback()` (with or without `processPendingChanges()`) **detached that in-memory instance** (`model.modelContext == nil`) while leaving its mutated property dirty in memory (`useCount == 2`) even though the store and a fresh fetch kept the committed value (`useCount == 1`, returning a different instance). Subsequent edits on the held reference were silently dropped on later `save()` calls. Only call `insert(_:)` when `model.modelContext == nil`.

Complete helper extension showing both guards for a single-write context where callers must not leave unrelated unsaved edits pending. `beforeSave` is a caller-supplied placeholder for synchronous checks that can throw before `save()`.

```swift
import SwiftData

extension ModelContext {
    func insertIfUnregistered<Model: PersistentModel>(_ models: [Model]) {
        for model in models where model.modelContext == nil {
            insert(model)
        }
    }

    func saveOrRollback(
        beforeSave: () throws -> Void = {}
    ) throws {
        do {
            try beforeSave()
            try save()
        } catch {
            processPendingChanges()
            rollback()
            throw error
        }
    }
}
```

Do not treat `saveOrRollback` as recovery from a `DefaultStore` validation error on a long-lived `mainContext`: if `try save()` itself fails inside the store, verify on the target OS whether subsequent saves on that context remain poisoned, and prefer an isolated `ModelContext(container)` when writes can fail at the store layer.

## Test both pre-save recovery and real store failures in parallel tests

When testing save-failure recovery under Swift Testing (which runs tests in parallel by default), do not toggle a shared `static var` failure flag across tests. Bind a `#if DEBUG` `@TaskLocal` flag with `withValue(true)` around a single test call so injected failures stay scoped to that task and its child tasks.

Keep the testing boundary clear: a `#if DEBUG` flag checked *before* `try save()` verifies application-side recovery, UI state preservation, and retry idempotency, **not** `DefaultStore` error handling. Add at least one test that triggers a real `save()` error (such as a relationship count constraint violation) against a fresh `ModelContainer` and verifies that a subsequent unrelated write still succeeds under your chosen context strategy.

## Validation scope

Verified 2026-11-24 with Xcode 27.2 (27B52), Apple Swift 6.4.2 (swiftlang-6.4.2.1.9), on macOS 27.2 arm64:

- Both Swift code blocks were extracted verbatim and passed `swiftc -typecheck` with `-warnings-as-errors` for iOS 17.0 (iPhoneOS 27.2 SDK) and macOS 14.0 (macOS 27.2 SDK) across Swift 6 (`nonisolated` and `MainActor` default isolation) and Swift 5 (`-strict-concurrency=complete`, `nonisolated` default isolation).
- Local macOS 27.2 execution probes (`cloudKitDatabase: .none`, both in-memory and on-disk SQLite stores) verified:
  - Unreverted pending deletes were silently committed by a subsequent unrelated `save()`.
  - `ModelContext.transaction(block:)` left `hasChanges == true` both when the closure threw and when its implicit `save()` failed with `NSCocoaErrorDomain Code=1580`.
  - After a real `DefaultStore` `save()` validation failure (`@Relationship(minimumModelCount: 1)`), `processPendingChanges()` + `rollback()` reset `hasChanges` to `false`, yet a subsequent unrelated `save()` on the same `ModelContext` still threw `Code=1580`; a separate `ModelContext(container)` on the same on-disk SQLite container saved cleanly.
  - Calling `context.insert(existingModel)` on an already-managed model (`modelContext != nil`) before `rollback()` detached the reference (`modelContext == nil`), left the mutated property dirty in memory while refetches returned a separate instance with committed values, and dropped later saves made through the detached reference; guarding with `model.modelContext == nil` prevented detachment and restored the committed value.
  - Unprocessed-delete `rollback()` followed by `save()` and property reads did **not** crash in local macOS 27.2 probes (simple models, cascade parent-child models, in-memory and SQLite stores, with and without `UndoManager`).

iOS runtimes, SwiftUI `@Query` live view invalidation after rollback, CloudKit-enabled stores, custom `DataStore` implementations, and multi-process contention were not executed locally.

## Sources

- https://roland.leth.ro/blog/tech/swiftdata-rollback-after-a-failed-save
- https://fatbobman.com/en/weekly/issue-161/
