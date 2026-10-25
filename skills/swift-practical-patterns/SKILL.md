---
name: swift-practical-patterns
description: "Swift/SwiftUI patterns: background refresh; local Live Activity lifecycle; notification audio; SwiftData/CloudKit duplicates/fetch shape; sheet dismissal; Core Spotlight; in-app Translation; async defer cleanup; task naming; Mutex/Atomic state; shared in-flight loads; async and throwing getters; Observable initialization/writes/onChange captures; FormatStyle, inline symbols, symbol effects, and attributed-text layout; map cameras/overlays; memberwise initialization; optional some/any syntax; protocol dispatch; macro type diagnostics/inference; non-View ContentBuilder DSLs; animation timing, phase feedback, and TimelineView cadence; container sizing, geometry breakpoints, and content margins; scroll anchors and position readback; chart dimming/sunburst/calendar coordinates; shaders; PDF assets; button shapes and concentric corners; alignment guides, compositing, and cutouts; navigation-bar minimization policies; decorative glows; macOS toolbars; iPhone Duo layouts; preview arguments and partial generation."
license: MIT
metadata:
  version: "0.2.0"
---

# Swift Practical Patterns

Use this guidance alongside broader Swift/SwiftUI skills for preview input catalogs, offline partial-generation fixtures, Swift app integrations, local Live Activity lifecycle, downloaded notification audio and alarm boundaries, sheet dismissal policies, persistence fetch shape, asynchronous cleanup, synchronous shared state, shared async loads, effectful getters, Observable construction and mutation boundaries, onChange capture semantics, task diagnostics, string and attributed-text formatting, paragraph line height, map cameras and geographic overlays, struct initialization, type syntax, protocol dispatch, symbol effects, chart dimming, hierarchical ring alignment, calendar chart coordinates, Metal shader binding, PDF asset scaling, macro expansion, non-View composition, animation control, scroll-position decisions, nested alignment, rendering boundaries, static decorative glows, navigation-bar minimization policies, system button shapes, concentric corners, macOS toolbar appearance, and adaptive layout decisions.

## Apply the guidance

- Read only the reference that matches the task. Check its platform, version, and configuration requirements before applying the approach.
- Preserve the limits and validation conditions. Distinguish typechecking, simulator display checks, local execution, debug triggering, and real system behavior when reporting evidence.
- Recheck affected official APIs and behavior when the target version falls outside the verified scope.

## Task routing

- Xcode 27 preview argument groups and their deployment/runtime requirements, or offline Foundation Models PartiallyGenerated fixtures: read [preview-state-fixtures.md](references/preview-state-fixtures.md).
- SwiftUI background content refresh, BGAppRefreshTaskRequest, scene registration, request replacement, or cancellation: read [background-refresh.md](references/background-refresh.md).
- Local Live Activity configuration, ActivityContent updates, stale dates, final content/dismissal, or relaunch reconciliation: read [live-activities.md](references/live-activities.md).
- Audio downloaded by notification service extensions, App Group sound lookup, expiry completion, or choosing UserNotifications versus AlarmKit: read [notification-service-audio.md](references/notification-service-audio.md).
- Seeded duplicates, business and record identity, duplicate diagnosis, or reconciliation decisions in SwiftData automatic CloudKit sync: read [swiftdata-sync.md](references/swiftdata-sync.md).
- SwiftData list fetch cost, large detail fields, propertiesToFetch, relationship prefetching, or deciding whether to split a model: read [swiftdata-fetch-shape.md](references/swiftdata-fetch-shape.md).
- Unsaved sheet edits, interactive versus programmatic dismissal, or detent changes mistaken for close attempts: read [sheet-dismissal.md](references/sheet-dismissal.md).
- Core Spotlight indexing, updates/deletion, deep links, reindexing, CSUserQuery, or stale in-app search results: read [core-spotlight.md](references/core-spotlight.md).
- In-app text translation, translationPresentation, TranslationSession configuration and repeat requests, language assets, or batch correlation: read [translation.md](references/translation.md).
- Awaited resource cleanup, asynchronous test teardown, Swift 6.4 defer bodies with await, or cleanup cancellation/error boundaries: read [async-cleanup.md](references/async-cleanup.md).
- Task diagnostic names, static versus retained-handle name access, SwiftUI .task(id:name:), or naming fallbacks on older runtimes: read [task-names.md](references/task-names.md).
- Synchronous shared caches, indivisible check-and-insert operations, recursive Mutex acquisition, or independent atomic counters and relaxed-ordering limits: read [synchronous-shared-state.md](references/synchronous-shared-state.md).
- Duplicate concurrent loads, sharing an in-flight Task, retry after failure, or cancellation ownership among independent waiters: read [shared-async-work.md](references/shared-async-work.md).
- Async or throwing computed properties, typed getter errors, read-only/key-path restrictions, or protocol witnesses with fewer effects: read [effectful-properties.md](references/effectful-properties.md).
- Parent-configured observable state, repeated constructor work, guarded deferred seeds, or choosing model replacement versus view/task identity: read [state-initialization.md](references/state-initialization.md).
- Equal snapshots that still notify, no-op in-place collection edits, or Observation notification versus didSet behavior: read [observation-mutations.md](references/observation-mutations.md).
- Migrating onChange capture lists, preserving old/new values, initial delivery versus app initialization, or View/Scene closure semantics: read [on-change.md](references/on-change.md).
- Custom FormatStyle interpolation in plain String literals, generic input/output constraints, or choosing between a formatting extension and localized interpolation: read [string-interpolation.md](references/string-interpolation.md).
- Inline SF Symbols inside a sentence, LocalizedStringKey image interpolation, or styling an interpolated symbol while preserving Image/Text types: read [inline-symbols.md](references/inline-symbols.md).
- Standalone SF Symbol palette/multicolor rules, variable values, or choosing state, event, replacement, and insertion/removal effects: read [symbol-images.md](references/symbol-images.md).
- Map initial versus bound camera position, center/zoom bounds, or geographic polygon, polyline, and radius overlays: read [map-camera-overlays.md](references/map-camera-overlays.md).
- SwiftUI Metal color/distortion/layer contracts, shader function lookup, uniform order, point coordinates, or sampling bounds: read [shader-effects.md](references/shader-effects.md).
- Circular glass buttons, buttonBorderShape versus clipping, containerShape, concentric corner minima/uniformity, or invalid RoundedCornerStyle.concentric syntax: read [control-shapes.md](references/control-shapes.md).
- PDF image assets that blur when enlarged, Single Scale versus Preserve Vector Data, or bitmap conversion before scaling: read [vector-assets.md](references/vector-assets.md).
- Masking an elapsed-time overlay to an area/line chart, plot-relative gradients, or cutoff versus nearest-sample semantics: read [chart-dimming.md](references/chart-dimming.md).
- Sunburst parent-child angular alignment, positive-weight partitions, or inner versus outer radius ratios: read [sunburst-charts.md](references/sunburst-charts.md).
- Calendar heatmap weekday projection, real-date identity, missing-day slots, or native versus synthetic chart accessibility: read [calendar-charts.md](references/calendar-charts.md).
- Semantic measurement-unit fonts, custom Markdown attribute decoding and rendering mappings, attributed-text link policies and in-app browser preferences, or paragraph line height versus line spacing and Dynamic Type: read [attributed-text.md](references/attributed-text.md).
- Synthesized struct memberwise initializers, Swift 6.4 private initialized storage, inaccessible initializers, or extension initializer conflicts: read [memberwise-initializers.md](references/memberwise-initializers.md).
- Protocol extension calls that bypass a concrete implementation, requirement defaults, or dispatch semantics versus optimization: read [protocol-dispatch.md](references/protocol-dispatch.md).
- Macro constraint diagnostics selected through overloads, omitted field annotations in generated signatures, or associated-type witness limits: read [macro-type-information.md](references/macro-type-information.md).
- Non-View ContentBuilder DSLs, TupleContent conformances, app-local ordered rules, multi-value back-deployment, or deciding when to own the builder: read [content-builder-dsls.md](references/content-builder-dsls.md).
- Optional some/any types, some View?, protocol-composition parentheses, or opaque-result nil inference: read [optional-protocol-types.md](references/optional-protocol-types.md).
- CustomAnimation completion/merging, spring duration versus completion criteria, phase reset and event feedback, or TimelineView schedules/cadence: read [animation-control.md](references/animation-control.md).
- Sizing cards with containerRelativeFrame, count/span spacing, onGeometryChange breakpoints, or feedback between geometry and state: read [container-geometry.md](references/container-geometry.md).
- Default scroll anchor roles, ScrollPosition readback versus live offsets, or target identity and anchor selection: read [scroll-position.md](references/scroll-position.md).
- Aligning a sibling with a nested title, custom AlignmentID fallbacks, active guide selection, or projected alignment values: read [alignment-guides.md](references/alignment-guides.md).
- Descendants separating during ancestor animations, combined opacity, destinationOut cutout scope, layered gradient-stroke halos and clipping order, or drawingGroup support/alpha limits: read [rendering-groups.md](references/rendering-groups.md).
- System container margins, ContentMarginGuide, contentMargins(for:), carousel content insets, or UIKit layoutMarginsGuide: read [layout-margins.md](references/layout-margins.md).
- Navigation-bar minimization, safe-area reflow versus stable content, or edge-only restoration and its supported combinations: read [toolbar-minimization.md](references/toolbar-minimization.md).
- macOS title/subtitle removal, titlebar backing versus item glass, toolbar background/color-scheme preferences, or full-screen hover reveal: read [macos-toolbar-appearance.md](references/macos-toolbar-appearance.md).
- ArrangementView ratio/point/ideal-size allocation, sheet placement and bar direction, or custom vertical toolbar legibility on iPhone Duo: read the matching section of [duo-layout-details.md](references/duo-layout-details.md).
