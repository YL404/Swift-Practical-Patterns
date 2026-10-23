---
name: swift-practical-patterns
description: "Implement or diagnose focused Swift and SwiftUI patterns: background refresh; SwiftData/CloudKit seeded duplicates; Core Spotlight; in-app Translation; async defer cleanup; task naming; Mutex and Atomic state; shared in-flight loads; Observation assignment and mutation; FormatStyle interpolation, inline symbols, and semantic attributed text; memberwise initialization; optional opaque/existential syntax; protocol dispatch; macro type diagnostics and inferred fields; non-View ContentBuilder DSLs; custom animation timing, phase feedback, and TimelineView cadence; container-relative sizing, geometry breakpoints, and system content margins; scroll anchors and position readback; nested alignment guides, geometry/compositing boundaries, and blend cutouts; and iPhone Duo arrangement, sheet, and vertical toolbar details. Use alongside broader Swift/SwiftUI skills when these scenarios need specific API, availability, lifecycle, identity, cancellation, or validation decisions."
license: MIT
metadata:
  version: "0.1.0"
---

# Swift Practical Patterns

Use focused guidance for Swift app integrations, asynchronous cleanup, synchronous shared state, shared async loads, Observation mutation boundaries, task diagnostics, string and attributed-text formatting, struct initialization, type syntax, protocol dispatch, macro expansion, non-View composition, animation control, scroll-position decisions, nested alignment, rendering boundaries, and adaptive layout decisions.

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
- Duplicate concurrent loads, sharing an in-flight Task, retry after failure, or cancellation ownership among independent waiters: read [shared-async-work.md](references/shared-async-work.md).
- Equal snapshots that still notify, no-op in-place collection edits, or Observation notification versus didSet behavior: read [observation-mutations.md](references/observation-mutations.md).
- Custom FormatStyle interpolation in plain String literals, generic input/output constraints, or choosing between a formatting extension and localized interpolation: read [string-interpolation.md](references/string-interpolation.md).
- Inline SF Symbols inside a sentence, LocalizedStringKey image interpolation, or styling an interpolated symbol while preserving Image/Text types: read [inline-symbols.md](references/inline-symbols.md).
- Semantic measurement-unit fonts, custom Markdown attribute decoding and rendering mappings, or attributed-text link policies and in-app browser preferences: read [attributed-text.md](references/attributed-text.md).
- Synthesized struct memberwise initializers, Swift 6.4 private initialized storage, inaccessible initializers, or extension initializer conflicts: read [memberwise-initializers.md](references/memberwise-initializers.md).
- Protocol extension calls that bypass a concrete implementation, requirement defaults, or dispatch semantics versus optimization: read [protocol-dispatch.md](references/protocol-dispatch.md).
- Macro constraint diagnostics selected through overloads, omitted field annotations in generated signatures, or associated-type witness limits: read [macro-type-information.md](references/macro-type-information.md).
- Non-View ContentBuilder DSLs, TupleContent conformances, app-local ordered rules, multi-value back-deployment, or deciding when to own the builder: read [content-builder-dsls.md](references/content-builder-dsls.md).
- Optional some/any types, some View?, protocol-composition parentheses, or opaque-result nil inference: read [optional-protocol-types.md](references/optional-protocol-types.md).
- CustomAnimation completion/merging, spring duration versus completion criteria, phase reset and event feedback, or TimelineView schedules/cadence: read [animation-control.md](references/animation-control.md).
- Sizing cards with containerRelativeFrame, count/span spacing, onGeometryChange breakpoints, or feedback between geometry and state: read [container-geometry.md](references/container-geometry.md).
- Default scroll anchor roles, ScrollPosition readback versus live offsets, or target identity and anchor selection: read [scroll-position.md](references/scroll-position.md).
- Aligning a sibling with a nested title, custom AlignmentID fallbacks, active guide selection, or projected alignment values: read [alignment-guides.md](references/alignment-guides.md).
- Descendants separating during ancestor animations, combined opacity, destinationOut cutout scope, or drawingGroup support/alpha limits: read [rendering-groups.md](references/rendering-groups.md).
- System container margins, ContentMarginGuide, contentMargins(for:), carousel content insets, or UIKit layoutMarginsGuide: read [layout-margins.md](references/layout-margins.md).
- ArrangementView ratio/point/ideal-size allocation, sheet placement and bar direction, or custom vertical toolbar legibility on iPhone Duo: read the matching section of [duo-layout-details.md](references/duo-layout-details.md).
