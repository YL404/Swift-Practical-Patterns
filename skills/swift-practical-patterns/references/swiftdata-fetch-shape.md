# SwiftData list fetches and detail payloads

Use when entering a list is slow or memory grows before detail content appears. Scope: SwiftData on iOS/iPadOS 17+ and macOS 14+, with the built-in store. Documentation checked 2026-10-23. Model splitting is a candidate to measure, not a default requirement for every field or a promised speedup.

## Check fetch work before changing the schema

Separate fetching model data from constructing and updating rows. A lazy view container does not by itself limit the data fetched by a query. Measure a representative persisted dataset, cold and warm contexts, fetch completion, list entry, memory, and scrolling on the supported runtimes. Keep row data access the same between comparisons; a row, predicate, sort, or prefetch that reads detail data can defeat the intended separation.

Apple documents `FetchDescriptor.propertiesToFetch` for fetching selected attributes, with an additional storage read when an omitted attribute is later accessed. Its empty default fetches all attributes. Try a descriptor matching the actual consumer, and measure it before deciding a schema change is necessary. [Apple: propertiesToFetch](https://developer.apple.com/documentation/swiftdata/fetchdescriptor/propertiestofetch)

The source article reports that this optimization did not reduce fetched columns in its tests. That observation is not a general replacement for Apple's contract: the published comparison lacks a complete reproducer and exact toolchain/device conditions, and it was not reproduced here. Do not teach that SwiftData always materializes every attribute or that `propertiesToFetch` never works. Investigate any discrepancy on the affected SDK, runtime, store, and query path.

Related models can require subsequent storage reads. `relationshipKeyPathsForPrefetching` requests related models in the original fetch; its default is empty. Prefetch a relationship when rows actually need it and measurement supports that choice. Leave detail-only payloads out of list prefetches; defer traversal until the detail consumer needs them. This does not guarantee a particular SQL sequence or memory footprint. [Apple: relationship prefetching](https://developer.apple.com/documentation/swiftdata/fetchdescriptor/relationshipkeypathsforprefetching)

`@Attribute(.externalStorage)` describes adjacent binary storage. It does not, by itself, specify an on-demand loading policy or a memory bound. Measure it separately from a relationship split. [Apple: externalStorage](https://developer.apple.com/documentation/swiftdata/schema/attribute/option/externalstorage)

## Split by access pattern when the measured cost justifies it

Keep list fields on a small model and move substantial detail-only content to a related model. Start with the actual expensive payload, not one entity per scalar. The tradeoff is additional relationships, reads, deletion rules, and migration work. A row that always needs the related content may benefit more from prefetching than from deferring access.

These complete model declarations and insertion function illustrate a privately owned note body. Register both types in the container schema. The caller supplies a context owned by the main actor and handles thrown errors; this is not a complete editor or migration.

```swift
import SwiftData

@Model
final class Note {
    var title: String = ""
    @Relationship(deleteRule: .cascade, inverse: \NoteBody.note)
    var detail: NoteBody?

    init(title: String) {
        self.title = title
    }
}

@Model
final class NoteBody {
    var text: String = ""
    var note: Note?

    init(text: String) {
        self.text = text
    }
}

@MainActor
func insertNote(title: String, text: String, in context: ModelContext) throws -> Note {
    let note = Note(title: title)
    let detail = NoteBody(text: text)
    context.insert(note)
    context.insert(detail)
    note.detail = detail
    detail.note = note
    try context.save()
    return note
}
```

`save()` commits all pending changes in that context, not just this note. The helper propagates errors without implementing retry or rollback; handle recovery in the caller's context policy rather than rolling back an unrelated shared editing session. [Apple: save](https://developer.apple.com/documentation/swiftdata/modelcontext/save())

The explicit inverse describes the two directions. The cascade is appropriate only because this example gives the body one owner: deleting the note deletes its body. Shared attachments need different ownership and deletion rules. Removing or replacing a relationship is not the same operation as deleting its owning model; explicitly decide how detached payloads are cleaned up. [Apple: Relationship](https://developer.apple.com/documentation/swiftdata/relationship(_:deleterule:minimummodelcount:maximummodelcount:originalname:inverse:hashmodifier:)), [cascade](https://developer.apple.com/documentation/swiftdata/schema/relationship/deleterule-swift.enum/cascade)

If the list needs a preview, either read an appropriately small related value or deliberately store a summary on the list model. A duplicated summary needs one update policy for insertion, edits, imports, and reconciliation; do not cache it without invalidation. Detail views must handle a nil relationship without silently overwriting missing or not-yet-synchronized data.

For shipped stores, prove migration preserves the original content and relationships before removing old fields. For CloudKit, relationships must be optional, changes are not atomic across the server, and a promoted schema is additive only. A locally passing split does not validate production migration or sync; consult [sync constraints](swiftdata-sync.md) and [Apple's schema guidance](https://developer.apple.com/documentation/swiftdata/syncing-model-data-across-a-persons-devices).

## Validation scope

Xcode 27.1 / Apple Swift 6.4, Swift 6, complete strict concurrency and warnings as errors: the code was typechecked for iOS 17 and macOS 14 deployment with nonisolated and MainActor default isolation. A local macOS 27.0.1 SQLite store with CloudKit disabled was used to check insertion, refetch in a fresh context, optional bodies, inverse links, edits, and owner deletion with cascade. No iOS execution, save-failure recovery, migration, CloudKit, SQL projection comparison, memory benchmark, or list UI was tested. No reproduction of the article's timings or memory ratios is claimed.

## Sources

- https://fatbobman.com/en/posts/swiftdata-optimization-starts-with-modeling/
