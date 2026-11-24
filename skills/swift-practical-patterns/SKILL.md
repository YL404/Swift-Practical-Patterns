---
name: swift-practical-patterns
description: "Practical Swift, SwiftUI, Apple-framework, and Xcode guidance for non-obvious API boundaries, OS/SDK version limits, and edge cases across language and concurrency, UI/layout/rendering, system integrations (e.g., SwiftData, widgets, Live Activities, App Intents, Foundation Models), and Xcode workflows. Use alongside broader Swift/SwiftUI skills."
license: MIT
metadata:
  version: "0.4.0"
---

# Swift Practical Patterns

Companion guidance to broader Swift and SwiftUI skills, focused on concrete API boundaries, version constraints, and edge-case implementation decisions. Match the task below and read only the needed reference.

## Apply the guidance

- Read only the reference that matches the task. Check its platform, version, and configuration requirements before applying the approach.
- Preserve the limits and validation conditions. Distinguish typechecking, simulator display checks, local execution, debug triggering, and real system behavior when reporting evidence.
- Recheck affected official APIs and behavior when the target version falls outside the verified scope.

## Task routing

### App and system integrations

- Assistive Access scenes, streamlined versus full-screen configuration, runtime detection or preview limits on iPhone/iPad: [assistive-access.md](references/assistive-access.md)
- App Attest key registration/reuse, challenge-bound requests, server verification boundaries, environment separation, or retry/counter coordination: [app-attest.md](references/app-attest.md)
- SwiftUI background content refresh, BGAppRefreshTaskRequest, scene registration, request replacement, or cancellation: [background-refresh.md](references/background-refresh.md)
- Local Live Activity configuration, ActivityContent updates, stale dates, final content/dismissal, relaunch reconciliation, or iOS 17.2+ push-to-start with background asset hydration and MainActor-default-isolation ActivityAttributes: [live-activities.md](references/live-activities.md)
- Audio downloaded by notification service extensions, App Group sound lookup, expiry completion, or choosing UserNotifications versus AlarmKit: [notification-service-audio.md](references/notification-service-audio.md)
- Seeded duplicates, business and record identity, duplicate diagnosis, or reconciliation decisions in SwiftData automatic CloudKit sync: [swiftdata-sync.md](references/swiftdata-sync.md)
- SwiftData list fetch cost, large detail fields, propertiesToFetch, relationship prefetching, or deciding whether to split a model: [swiftdata-fetch-shape.md](references/swiftdata-fetch-shape.md)
- SwiftData save() failure recovery, transaction(block:) non-rollback, DefaultStore context poisoning, duplicate-insert detach on rollback, or parallel failure-injection tests: [swiftdata-save-recovery.md](references/swiftdata-save-recovery.md)
- Core Spotlight indexing, updates/deletion, deep links, reindexing, CSUserQuery, or stale in-app search results: [core-spotlight.md](references/core-spotlight.md)
- In-app text translation, translationPresentation, TranslationSession configuration and repeat requests, language assets, or batch correlation: [translation.md](references/translation.md)
- Reusable Foundation Models profile modifiers, model/default precedence, request-local history transforms, response-token caps, local token-budget screening, or cloud-routing boundaries: [dynamic-model-profiles.md](references/dynamic-model-profiles.md)
- Testing registered App Intents and entity references out of process, or selecting a UI test target without importing app internals: [app-intents-testing.md](references/app-intents-testing.md)
- Widget gallery location preferences, carPlay availability, or confusing disfavoredLocations with hiding or privacy enforcement: [widget-placement.md](references/widget-placement.md)
- Checking whether an OSLog level is enabled before expensive diagnostic work on pre-26.4 OS targets, or avoiding strong-linked Logger.isEnabled(type:) launch crashes: [oslog-availability.md](references/oslog-availability.md)
- Replacing deprecated URL.path, path(percentEncoded:) defaults and relative-URL OS differences, directory trailing slashes, FilePath(url), or POSIX file-system paths: [file-url-paths.md](references/file-url-paths.md)

### Swift language and concurrency

- Awaited resource cleanup, asynchronous test teardown, Swift 6.4 defer bodies with await, OS 27 cancellation shields, or cleanup cancellation/error boundaries: [async-cleanup.md](references/async-cleanup.md)
- Unused throwing Task warnings, task failure ownership, SDK limits on typed task initializers, or async Result capture: [task-outcomes.md](references/task-outcomes.md)
- Task diagnostic names, static versus retained-handle name access, SwiftUI .task(id:name:), or naming fallbacks on older runtimes: [task-names.md](references/task-names.md)
- Synchronous shared caches, indivisible check-and-insert operations, recursive Mutex acquisition, or independent atomic counters and relaxed-ordering limits: [synchronous-shared-state.md](references/synchronous-shared-state.md)
- Duplicate concurrent loads, sharing an in-flight Task, retry after failure, or cancellation ownership among independent waiters: [shared-async-work.md](references/shared-async-work.md)
- Fixed weak delegates in Sendable classes, weak let, or intentionally suppressing inferred conformance with ~Sendable: [sendability-declarations.md](references/sendability-declarations.md)
- Async or throwing computed properties, typed getter errors, read-only/key-path restrictions, or protocol witnesses with fewer effects: [effectful-properties.md](references/effectful-properties.md)
- Equal snapshots that still notify, no-op in-place collection edits, or Observation notification versus didSet behavior: [observation-mutations.md](references/observation-mutations.md)
- Custom FormatStyle interpolation in plain String literals, generic input/output constraints, or choosing between a formatting extension and localized interpolation: [string-interpolation.md](references/string-interpolation.md)
- Synthesized struct memberwise initializers, Swift 6.4 private initialized storage, inaccessible initializers, or extension initializer conflicts: [memberwise-initializers.md](references/memberwise-initializers.md)
- Protocol extension calls that bypass a concrete implementation, requirement defaults, or dispatch semantics versus optimization: [protocol-dispatch.md](references/protocol-dispatch.md)
- Macro constraint diagnostics selected through overloads, inferred field signatures, associated-type witness limits, or compiler tests beyond expansion snapshots: [macro-type-information.md](references/macro-type-information.md)
- Non-View ContentBuilder DSLs, TupleContent conformances, app-local ordered rules, multi-value back-deployment, or deciding when to own the builder: [content-builder-dsls.md](references/content-builder-dsls.md)
- Optional some/any types, some View?, protocol-composition parentheses, or opaque-result nil inference: [optional-protocol-types.md](references/optional-protocol-types.md)

### SwiftUI state, layout, navigation, and platform UI

- Parent-configured observable state, repeated constructor work, guarded deferred seeds, or choosing model replacement versus view/task identity: [state-initialization.md](references/state-initialization.md)
- Migrating onChange capture lists, preserving old/new values, initial delivery versus app initialization, or View/Scene closure semantics: [on-change.md](references/on-change.md)
- Choosing a sheet transition at runtime, AnyNavigationTransition, crossFade, or platform/availability limits: [navigation-transitions.md](references/navigation-transitions.md)
- Unsaved sheet edits, interactive versus programmatic dismissal, or detent changes mistaken for close attempts: [sheet-dismissal.md](references/sheet-dismissal.md)
- Adaptive .inspector placement, iOS 26+ inspector toolbars/columns, inspectorColumnWidth preferences, or compact inspector sheet-detent limits: [inspector-presentation.md](references/inspector-presentation.md)
- Selection-driven multi-item drag payloads, matching identifier/namespace scope, or selection order in reorder differences: [collection-drag-selection.md](references/collection-drag-selection.md)
- Custom row colors over selection, backgroundProminence scope, or system selection backgrounds versus manual focus tracking: [selection-appearance.md](references/selection-appearance.md)
- Default scroll anchor roles, ScrollPosition readback versus live offsets, or target identity and anchor selection: [scroll-position.md](references/scroll-position.md)
- Choosing among scrollTransition, visualEffect, onScrollVisibilityChange, onScrollTargetVisibilityChange (idType: label), and clamped onScrollGeometryChange offsets: [scroll-driven-effects.md](references/scroll-driven-effects.md)
- Sizing cards with containerRelativeFrame, count/span spacing, onGeometryChange breakpoints, or feedback between geometry and state: [container-geometry.md](references/container-geometry.md)
- Replacing size proposals with fixedSize(), flexible shape ideal-size order, container-level equal-height matching, or unclipped overflow and sibling starvation: [fixed-size.md](references/fixed-size.md)
- Container layoutPriority(_:) space allocation in HStack/VStack, ZStack and multi-view overlay/background sizing, custom LayoutSubview.priority, or containers that ignore priority: [layout-priority.md](references/layout-priority.md)
- Aligning a sibling with a nested title, custom AlignmentID fallbacks, active guide selection, or projected alignment values: [alignment-guides.md](references/alignment-guides.md)
- System container margins, ContentMarginGuide, contentMargins(for:), carousel content insets, or UIKit layoutMarginsGuide and iOS 27.1+ zero UIView default margins: [layout-margins.md](references/layout-margins.md)
- Circular glass buttons, buttonBorderShape versus clipping, containerShape, concentric corner minima/uniformity, or invalid RoundedCornerStyle.concentric syntax: [control-shapes.md](references/control-shapes.md)
- Grouping custom glassEffect views, GlassEffectContainer spacing versus glassEffectUnion, glassEffectID, or choosing glassEffectTransition (.matchedGeometry versus .materialize): [glass-containers.md](references/glass-containers.md)
- Navigation-bar minimization, safe-area reflow versus stable content, or edge-only restoration and its supported combinations: [toolbar-minimization.md](references/toolbar-minimization.md)
- macOS title/subtitle removal, titlebar backing versus item glass, toolbar background/color-scheme preferences, or full-screen hover reveal: [macos-toolbar-appearance.md](references/macos-toolbar-appearance.md)
- Existing NSViewRepresentable/NSHostingView nesting, explicit teardown, or shared AppKit view ownership after host replacement: [appkit-host-lifecycle.md](references/appkit-host-lifecycle.md)
- ArrangementView ratio/point/ideal-size allocation, sheet placement and bar direction, or custom vertical toolbar legibility on iPhone Duo: [duo-layout-details.md](references/duo-layout-details.md) (matching section)

### SwiftUI visuals, text, charts, and maps

- Bundled custom-font registration or silent fallback, semantic measurement-unit fonts, custom Markdown attribute decoding and rendering mappings, attributed-text link policies and in-app browser preferences, or paragraph line height versus line spacing and Dynamic Type: [attributed-text.md](references/attributed-text.md)
- Inline SF Symbols inside a sentence, LocalizedStringKey image interpolation, or styling an interpolated symbol while preserving Image/Text types: [inline-symbols.md](references/inline-symbols.md)
- SF Symbols CLI name/rename and deployment checks, standalone palette/multicolor rules, variable values, or choosing state, event, replacement, and insertion/removal effects: [symbol-images.md](references/symbol-images.md)
- PDF image assets that blur when enlarged, Single Scale versus Preserve Vector Data, or bitmap conversion before scaling: [vector-assets.md](references/vector-assets.md)
- CustomAnimation completion/merging/thread safety (macOS vs iOS), spring duration versus completion criteria, phase reset and event feedback, or TimelineView schedules/cadence: [animation-control.md](references/animation-control.md)
- Descendants separating during ancestor animations, combined opacity, destinationOut cutout scope, layered gradient-stroke halos and clipping order, or drawingGroup support/alpha limits: [rendering-groups.md](references/rendering-groups.md)
- SwiftUI Metal color/distortion/layer contracts, shader function lookup, uniform order, point coordinates, or sampling bounds: [shader-effects.md](references/shader-effects.md)
- Masking an elapsed-time overlay to an area/line chart, plot-relative gradients, or cutoff versus nearest-sample semantics: [chart-dimming.md](references/chart-dimming.md)
- Sunburst parent-child angular alignment, positive-weight partitions, or inner versus outer radius ratios: [sunburst-charts.md](references/sunburst-charts.md)
- Calendar heatmap weekday projection, real-date identity, missing-day slots, or native versus synthetic chart accessibility: [calendar-charts.md](references/calendar-charts.md)
- Map initial versus bound camera position, center/zoom bounds, or geographic polygon, polyline, and radius overlays: [map-camera-overlays.md](references/map-camera-overlays.md)

### Previews and Xcode workflows

- Xcode 27 preview argument groups and their deployment/runtime requirements, or offline Foundation Models PartiallyGenerated fixtures: [preview-state-fixtures.md](references/preview-state-fixtures.md)
- Xcode clean/no-change/edit build comparisons, repeated scripts, compilation-cache evaluation or prefix-mapped grey LLDB breakpoints, or choosing between Xcode MCP (mcpbridge/mcp-server) and xcodebuild: [xcode-build-performance.md](references/xcode-build-performance.md)
- Disk-pressure decisions about build artifacts, distributed archive retention, or matching binaries and dSYMs by UUID: [build-artifact-retention.md](references/build-artifact-retention.md)
