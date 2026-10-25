# Multi-item drag selection and reorder order

Use this reference when a drag should include the current selection, when adjacent drag containers accidentally share selection, or when applying a multi-item reorder difference. Basic `reorderable`/`reorderContainer` wiring does not need a custom drag coordinator.

Verified against Apple documentation on 2026-10-25. The drag-selection example requires iOS/iPadOS 27 or macOS 26. Reordering requires iOS/iPadOS/macOS 27. Preserve older deployment targets and provide their existing interaction rather than assuming that all these APIs share the same availability.

## Pass selection identifiers to the matching container

Mark each source with `draggable(containerItemID:)`, provide transferable values through `dragContainer`, and place [dragContainerSelection(_:containerNamespace:)](https://developer.apple.com/documentation/swiftui/view/dragcontainerselection(_:containernamespace:)) around that container. SwiftUI finds the nearest enclosing selection with the same identifier type and, when specified, the same namespace.

If the lifted item belongs to the selection, the payload callback receives the selected identifiers. Lifting an unselected item includes only that item; it must not export the unrelated selection. Without selection information, the callback receives the lifted identifier. Resolve the callback's identifiers to the current model when the callback runs, and return an empty collection when the app intends to disable that drag. [Payload contract](https://developer.apple.com/documentation/swiftui/view/dragcontainer(for:in:_:))

This complete source view uses unique, stable string tokens as both identifiers and transferable demo values. The parent owns selection and its selection UI. For editable production models, keep identity separate from mutable text and define an appropriate `Transferable` representation.

```swift
import SwiftUI

@available(iOS 27.0, macOS 26.0, *)
struct SelectedTokenDragSource: View {
    let tokens: [String]
    let selectedTokenIDs: [String]

    var body: some View {
        VStack {
            ForEach(tokens, id: \.self) { token in
                Text(token)
                    .draggable(containerItemID: token)
            }
        }
        .dragContainer(for: String.self, itemID: \.self) { ids in
            ids.filter { tokens.contains($0) }
        }
        .dragContainerSelection(selectedTokenIDs)
    }
}
```

`String` is not `Identifiable`, so this example uses the [itemID: overload](https://developer.apple.com/documentation/swiftui/view/dragcontainer(for:itemid:in:_:)). Keep the same identifier type throughout the row, payload, and selection APIs. For independently selected nested or adjacent groups, scope the selection around each intended group or pass a matching namespace to `dragContainer(in:)`, `draggable(containerNamespace:)`, and `dragContainerSelection(containerNamespace:)`. An ancestor selection of the same identifier type can otherwise be found by a descendant container.

Selection-driven transfer does not itself delete the source or implement a move. Decide copy versus move separately and update the source only under the chosen successful transfer policy.

## Preserve the order supplied by a reorder difference

For a `reorderContainer` move, Apple's [contract](https://developer.apple.com/documentation/swiftui/view/reordercontainer(for:isenabled:move:)) says `difference.sources` contains identifiers in the order the person selected them. Capture moved values in that sequence before removing them, then resolve the destination in the remaining collection and insert the captured values in the same sequence. Rebuilding the moved sequence by scanning the original array or iterating a `Set` can change the requested order. Apply all source identifiers, not just `sources[0]`.

The `swiftui-expert-skill` list-patterns reference uses “source order” without explicitly distinguishing collection order from selection order. For this API, preserve the order of `difference.sources`, as Apple’s contract specifies; do not reconstruct moved values by scanning the original collection. `swiftui-pro` 2.0.0 explicitly follows that selected-identifier sequence. This clarification does not replace its helper or establish an ordering guarantee for unrelated drag payload APIs.

As an implementation recommendation, validate source/destination identifiers before publishing the mutation, or prepare a copy and assign it only after success. The article's single-item helper removes the item before checking a `.before` destination; a missing destination would then leave the item removed. Define a stale-data policy for model changes during a drag rather than silently losing an item. For multiple collections, also resolve `destination.collectionID` and preserve the supplied source sequence across collections.

## Validation scope

The exact source view passed typechecking with Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), the iPhoneOS 27.1 SDK targeting arm64 iOS 27, and the macOS 27.0 SDK targeting arm64 macOS 26. Checks used Swift 6, complete strict concurrency, warnings as errors, and both `nonisolated` and `MainActor` default isolation. No drag, drop, selection control, namespace interaction, reorder mutation, accessibility interaction, or cross-app transfer was run. Ordering guidance follows the official contract and source-code review, not a reproduced gesture.

## Sources

- https://nilcoalescing.com/blog/NewSwiftUIAPIsForReorderingAndDragAndDropOniOS27/
