---
name: swift-practical-patterns
description: Implement or diagnose SwiftUI background app refresh, seeded duplicates in SwiftData CloudKit sync, Core Spotlight search, in-app Translation framework flows, awaited cleanup with Swift 6.4 async defer, task naming and name-inspection availability, synchronous caches with Mutex and independent Atomic counters, custom FormatStyle interpolation in plain Swift strings, inline SF Symbols in localized SwiftUI Text, struct memberwise initializer access and Swift 6.4 private-storage migration, optional opaque/existential type syntax and inference, protocol requirement versus extension method selection, system container margins and UIKit layoutMarginsGuide, or iPhone Duo ArrangementView split sizing, sheet placement, and custom vertical toolbar legibility. Use for lifecycle, identity, cancellation, language assets, formatting, initialization, type syntax, layout constraints, and validation decisions.
license: MIT
metadata:
  version: "0.1.0"
---

# Swift Practical Patterns

Use focused guidance for Swift app integrations, asynchronous cleanup, synchronous shared state, task diagnostics, string formatting, struct initialization, type syntax, protocol dispatch, and adaptive layout decisions.

## Apply the guidance

- Read only the reference that matches the task. Check its platform, version, and configuration requirements before applying the approach.
- Preserve the limits and validation conditions. Distinguish typechecking, simulator display checks, local execution, debug triggering, and real system behavior when reporting evidence.
- Recheck affected official APIs and behavior when the target version falls outside the verified scope.

## Task routing

- SwiftUI background content refresh, BGAppRefreshTaskRequest, scene registration, request replacement, or cancellation: read [background-refresh.md](references/background-refresh.md).
- Seeded duplicates, business and record identity, duplicate diagnosis, or reconciliation decisions in SwiftData automatic CloudKit sync: read [swiftdata-sync.md](references/swiftdata-sync.md).
- Core Spotlight indexing, updates/deletion, deep links, reindexing, CSUserQuery, or stale in-app search results: read [core-spotlight.md](references/core-spotlight.md).
- In-app text translation, translationPresentation, TranslationSession configuration and repeat requests, language assets, or batch correlation: read [translation.md](references/translation.md).
- Awaited resource cleanup, asynchronous test teardown, Swift 6.4 defer bodies with await, or cleanup cancellation/error boundaries: read [async-cleanup.md](references/async-cleanup.md).
- Task diagnostic names, static versus retained-handle name access, SwiftUI .task(id:name:), or naming fallbacks on older runtimes: read [task-names.md](references/task-names.md).
- Synchronous shared caches, indivisible check-and-insert operations, recursive Mutex acquisition, or independent atomic counters and relaxed-ordering limits: read [synchronous-shared-state.md](references/synchronous-shared-state.md).
- Custom FormatStyle interpolation in plain String literals, generic input/output constraints, or choosing between a formatting extension and localized interpolation: read [string-interpolation.md](references/string-interpolation.md).
- Inline SF Symbols inside a sentence, LocalizedStringKey image interpolation, or styling an interpolated symbol while preserving Image/Text types: read [inline-symbols.md](references/inline-symbols.md).
- Synthesized struct memberwise initializers, Swift 6.4 private initialized storage, inaccessible initializers, or extension initializer conflicts: read [memberwise-initializers.md](references/memberwise-initializers.md).
- Protocol extension calls that bypass a concrete implementation, requirement defaults, or dispatch semantics versus optimization: read [protocol-dispatch.md](references/protocol-dispatch.md).
- Optional some/any types, some View?, protocol-composition parentheses, or opaque-result nil inference: read [optional-protocol-types.md](references/optional-protocol-types.md).
- System container margins, ContentMarginGuide, contentMargins(for:), carousel content insets, or UIKit layoutMarginsGuide: read [layout-margins.md](references/layout-margins.md).
- ArrangementView ratio/point/ideal-size allocation, sheet placement and bar direction, or custom vertical toolbar legibility on iPhone Duo: read the matching section of [duo-layout-details.md](references/duo-layout-details.md).
