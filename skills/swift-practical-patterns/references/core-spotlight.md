# Core Spotlight index lifecycle and in-app search

Use Core Spotlight to expose app records in system search and query the same indexed content from your app. This reference covers stable identity, updates/deletion, deep links, index recovery, and user-entered queries. It does not implement an App Intents integration or a complete search UI.

Official documentation verified 2026-10-09. The compilable snippets below were checked with Xcode 27.0, Apple Swift 6.4, Swift 6 language mode, and the iOS/macOS 27 SDKs for iOS 16 and macOS 13 deployment targets. No indexing, query, extension, or device behavior was executed.

## Keep the store authoritative and the index recoverable

Give each `CSSearchableItem` a durable `uniqueIdentifier` that resolves to your app's record. The identifier must be unique within your app; a `domainIdentifier` groups records but does not replace that identity. Set your custom identifier before first indexing and use the same value for updates, deletion, query-result mapping, and deep links. A generated identifier is useful only if the app retains its mapping. [Unique identifiers](https://developer.apple.com/documentation/corespotlight/cssearchableitem/uniqueidentifier)

Index newly available records, index them again when relevant attributes change, and delete their index entries when the source records are deleted. Items also expire; choose an `expirationDate` appropriate to the content's lifetime and refresh it when needed. The index can be lost or need repair, so keep the app's store as the source of truth and make reindexing possible. [Apple indexing lifecycle](https://developer.apple.com/documentation/corespotlight/adding-your-app-s-content-to-spotlight-indexes), [Index recovery](https://developer.apple.com/documentation/corespotlight/regenerating-your-app-s-indexes-on-demand)

For production, Apple directs apps to use a named `CSSearchableIndex`; reserve the default index for testing/prototyping. Named indexes support batching and configurable data protection. Modify an index from only one thread/task at a time. Serialize **complete operations**, including their suspension points: placing an async method on an actor alone does not prevent another call entering while the first awaits. Do not overlap batches. [Apple index requirements](https://developer.apple.com/documentation/corespotlight/cssearchableindex), [Swift actor reentrancy](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md#actor-reentrancy)

### Minimal indexing and deletion

These are compilable helpers, not a complete synchronization/recovery implementation. Supply snapshots from the app's existing store and serialize calls through the app's index-update owner. `SearchRecord.id` must already satisfy the identity rule above. A `false` return means indexing was unavailable; a `true` return means submission completed without a reported error. Apple defines completion as journaling the operation, rather than proof that the item is already searchable; later indexing failure can trigger a reindex request. [Index submission semantics](https://developer.apple.com/documentation/corespotlight/cssearchableindex/indexsearchableitems(_:completionhandler:))

```swift
import CoreSpotlight
import UniformTypeIdentifiers

struct SearchRecord {
    let id: String
    let title: String
    let summary: String
}

func indexRecords(_ records: [SearchRecord]) async throws -> Bool {
    guard CSSearchableIndex.isIndexingAvailable() else { return false }
    let items = records.map { record in
        let attributes = CSSearchableItemAttributeSet(contentType: .content)
        attributes.title = record.title
        attributes.contentDescription = record.summary
        return CSSearchableItem(
            uniqueIdentifier: record.id,
            domainIdentifier: "records",
            attributeSet: attributes
        )
    }
    let index = CSSearchableIndex(name: "com.example.app.content")
    try await index.indexSearchableItems(items)
    return true
}

func deleteIndexEntries(withIdentifiers ids: [String]) async throws {
    let index = CSSearchableIndex(name: "com.example.app.content")
    try await index.deleteSearchableItems(withIdentifiers: ids)
}
```

For initial indexing or rebuilding missing content, leave `isUpdate` false. Setting `item.isUpdate = true` updates an existing entry more efficiently, but **ignores an item absent from the index**; it does not perform an upsert. With update mode, setting an attribute to `nil` removes its value. This is useful for changing metadata such as `lastUsedDate` only when the indexed item is known to exist. [Apple update semantics](https://developer.apple.com/documentation/corespotlight/cssearchableitem/isupdate)

### Client state is a recovery checkpoint

Named-index batch updates can persist up to 250 bytes of client-state `Data` with `endBatch(withClientState:)`; `fetchLastClientState()` helps determine where to resume after interruption. Use a source revision or progress marker that the app can interpret. The mere presence of state, or a timestamp from first launch, does not establish that today's added, edited, deleted, or expired records are represented. [Batch state and size limit](https://developer.apple.com/documentation/corespotlight/cssearchableindex/endbatch(withclientstate:completionhandler:)), [Apple batching guidance](https://developer.apple.com/documentation/corespotlight/adding-your-app-s-content-to-spotlight-indexes)

This reference does not supply an unverified batch-failure algorithm. Before adopting batching, define successful checkpoint advancement, fetch/index/end-batch error handling, and how the next run recovers partial progress. A marker written after failed work must not falsely mark that work complete.

## Resolve Spotlight deep links against current data

Handle `onContinueUserActivity(CSSearchableItemActionType)` in the SwiftUI view that owns routing. Read the string under `activity.userInfo?[CSSearchableItemActivityIdentifier]`; it is the identifier supplied when indexing. Resolve it through the current store, then navigate using the app's existing routing. Treat a malformed identifier, deleted record, or currently unavailable record as a recoverable navigation outcome. The index entry is not proof that the source record is still accessible. [Activity type](https://developer.apple.com/documentation/corespotlight/cssearchableitemactiontype), [Identifier key](https://developer.apple.com/documentation/corespotlight/cssearchableitemactivityidentifier)

Check declared activity types and scene selection against the modifier's configuration requirements. SwiftUI's choice of scene depends on the app structure, active scenes, and external-event configuration; a correct identifier extraction does not prove cold-launch or multi-scene routing. [SwiftUI activity handling](https://developer.apple.com/documentation/swiftui/view/oncontinueuseractivity(_:perform:))

For index repair when the app is not running, Apple's designated path is a **CoreSpotlight Delegate** app extension subclassing `CSIndexExtensionRequestHandler`. Implement both all-items and selected-identifiers reindex handlers, with access to the original indexable data, and acknowledge the completed indexing/recovery work. Do not acknowledge immediately while that work is still pending. Adding model/source files to both targets shares code or bundled resources; it does not give the extension access to the app's mutable runtime store. Configure an appropriate shared container when that store must be read by both processes. A full extension target and its data access were not built or exercised here. [Apple reindexing extension](https://developer.apple.com/documentation/corespotlight/regenerating-your-app-s-indexes-on-demand), [App Group data sharing](https://developer.apple.com/documentation/xcode/configuring-app-groups)

## Query user-entered text with CSUserQuery

`CSUserQuery` and its `responses` sequence require iOS/iPadOS 16 or macOS 13. Lexical querying is available at those versions; Apple's search-interface guide adds semantic search from iOS 18/macOS 15. Do not promise semantic matching on the older targets, or a particular semantic result on an untested target. [CSUserQuery](https://developer.apple.com/documentation/corespotlight/csuserquery), [Search-interface guide](https://developer.apple.com/documentation/corespotlight/building-a-search-interface-for-your-app)

Create a query with `CSUserQuery(userQueryString:userQueryContext:)` for text typed by a person. The inherited `queryString:queryContext:` initializer takes Spotlight predicate syntax; it is not an interchangeable initializer for arbitrary text. The article author reports an invalid-query error when choosing the latter with free text; use the documented user-query API rather than depending on that exact error code. [User-query initializer](https://developer.apple.com/documentation/corespotlight/csuserquery/init(userquerystring:userquerycontext:)), [Predicate queries](https://developer.apple.com/documentation/corespotlight/cssearchquery)

Reading `query.responses` starts the search and delivers `.item` and `.suggestion` values. Do not also call `start()` for that automatic execution path. A query is used once; when the input changes, cancel it and create a new one. Configure `CSUserQueryContext` for the attributes and result limits the UI needs. A query can search the app's indexes; the name used when creating an index is not a query-scoping argument in this initializer. Map returned identifiers to records that belong to the current screen's data set. [Responses](https://developer.apple.com/documentation/corespotlight/csuserquery/responses-swift.property), [Query lifecycle](https://developer.apple.com/documentation/corespotlight/csuserquery), [Query context](https://developer.apple.com/documentation/corespotlight/csuserquerycontext)

The following compilable collector accepts an already-created query. It is only the result-collection part: the caller must retain the query so it can call `cancel()` when superseded or when the search UI disappears.

```swift
import CoreSpotlight

@available(iOS 16.0, macOS 13.0, *)
@MainActor
func collectResults(from query: CSUserQuery) async throws -> [CSUserQuery.Item] {
    var results: [CSUserQuery.Item] = []
    for try await response in query.responses {
        try Task.checkCancellation()
        if case .item(let result) = response {
            results.append(result)
        }
    }
    try Task.checkCancellation()
    return results
}
```

### Cancellation and stale results are separate responsibilities

Reuse the app's existing input-delay mechanism to avoid launching a query for every keystroke. Apple's guide gives 0.3 seconds as an example, not a required interval. Cancellation of a SwiftUI `.task(id:)` is not sufficient evidence that the underlying Spotlight query has promptly stopped. Keep its handle and call `query.cancel()` when replacing it. Also invalidate the old request's right to update UI state. [Apple search and cancellation guidance](https://developer.apple.com/documentation/corespotlight/building-a-search-interface-for-your-app)

Keep the query/task handles, request ID, and result publication in one isolation domain, typically `MainActor` for an owner of UI state. Validate the request and publish its result without an intervening suspension, so a new input cannot invalidate the request between those steps. This isolates search-state bookkeeping; index-update operations still need the complete-operation serialization described above.

Apply this flow in the existing search owner; it is pseudocode rather than an additional architecture requirement:

```text
On input change, including clearing it:
    Invalidate the previous request ID
    Cancel the previous Swift task and its CSUserQuery
    Clear results and errors that belong to the previous input
    If empty, stop
    After the input delay, check that the captured request is still current
    Create a new CSUserQuery and retain its handle
    Await results from that query
    Before publishing results OR an error, check cancellation and request ID
    Resolve identifiers against the current store; skip unavailable records
On search UI disappearance:
    Invalidate the current request and cancel its task and query
```

Use a request ID rather than text equality alone if an input can change from A to B and back to A while the first A request is still pending. Do not let a cancelled query's error path clear a newer query's results. These are implementation safeguards; their app-specific integration and runtime timing remain to be tested.

## Ranking, engagement, and verification

`CSUserQueryContext.enableRankedResults` defaults to `true` in Apple's documentation. Pass an explicit context when configuring search behavior; the author's observation that ranking failed with a `nil` context is not adopted as a general requirement. On iOS 18/macOS 15+, `userEngaged(_:visibleItems:interaction:)` reports actual interaction with a result and the items visible at that time. It helps future ranked queries, but does not guarantee a particular result's position. [Ranked-results option](https://developer.apple.com/documentation/corespotlight/csuserquerycontext/enablerankedresults), [Engagement reporting](https://developer.apple.com/documentation/corespotlight/csuserquery/userengaged(_:visibleitems:interaction:))

`CSUserQuery.prepare()` also requires iOS 18/macOS 15. Apple recommends calling it once per app run shortly before search is needed; preparation consumes resources and increases memory use. Gate it for older targets and do not call it per keystroke. [Apple preparation guidance](https://developer.apple.com/documentation/corespotlight/building-a-search-interface-for-your-app)

The two Swift snippets above were extracted verbatim and typechecked with `swiftc -typecheck -swift-version 6` for arm64 iOS 16 and macOS 13 targets using the 27.0 SDKs; both passed without diagnostics. These checks validate API usage and availability, not index visibility, result ordering, cancellation timing, or extension loading.

In a signed app, verify creation, editing, deletion, expiration/recovery, deep links to existing and unavailable records, rapid input replacement and clearing, view disappearance, and query errors while a newer request is active. Test suggestions, semantic matches, and ranking on the target OS before relying on them; the article's limited observations do not establish their current behavior. Reindex-extension acceptance also requires its actual target and runtime store access.

Identity, lifecycle, availability, and query conclusions use the Apple sources above; batching failure recovery and device search quality remain outside the executed verification.

## Sources

- [Core Spotlight integration for Spotlight and internal app search](https://nilcoalescing.com/blog/CoreSpotlightIntegration/)
