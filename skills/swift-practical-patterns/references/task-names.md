# Diagnose concurrent work with task names

Use this when several tasks originate at the same call site and you need to identify their work in diagnostics. Scope: Swift 6.2+ task creation, with the iOS/macOS runtime distinctions below. Names are optional, fixed at creation, and do not establish task identity or change isolation, priority, cancellation, or ownership. Keep application state in explicit values rather than interpreting a diagnostic label. [SE-0469](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0469-task-names.md)

## Separate compiler support from runtime support

| Operation | Requirement and fallback |
|---|---|
| Pass `name:` to `Task`, `Task.detached`, or task-group child creation APIs | Introduced in Swift 6.2. The creation APIs support earlier deployment targets than name inspection; older runtimes may ignore the label. |
| Read the current task with static `Task.name` | iOS 26 / macOS 26 or later. Guard the read when deploying earlier. |
| Read a retained task handle with instance `task.name` | Verified with Swift 6.4 and the 27 SDKs; requires iOS 27 / macOS 27 or later. This is a separate availability requirement from static `Task.name`; check older compilers separately. |
| Name a SwiftUI view task with `.task(name:)` or `.task(id:name:)` | Use an SDK containing these overloads. In Xcode 27.1 they support iOS 15 / macOS 12 deployment, but `name`, `file`, and `line` have no effect before iOS/macOS 26.4. |

Apple documents the runtime-dependent effect of task creation names alongside [static Task.name](https://developer.apple.com/documentation/swift/task/name-swift.type.property), the later availability of [instance Task.name](https://developer.apple.com/documentation/swift/task/name-swift.property), and the naming fallback in [View.task](https://developer.apple.com/documentation/swiftui/view/task(name:priority:file:line:_:)). Compiling against a newer SDK does not make an older runtime expose task names. Other Apple platforms need their corresponding availability checks.

## Name work at its existing creation boundary

This is a complete diagnostic helper for iOS/macOS 26+. It names structured children without changing them into independent tasks. Result order follows child completion, so callers must not rely on the returned order.

```swift
@available(iOS 26, macOS 26, *)
func collectNamedChildren() async -> [String] {
    await withTaskGroup(of: String.self) { group in
        for label in ["Load items", "Load categories"] {
            group.addTask(name: label) {
                Task.name ?? "unnamed"
            }
        }
        var names: [String] = []
        for await name in group {
            names.append(name)
        }
        return names
    }
}
```

For existing unstructured work, add `name:` to the existing `Task` initializer. Keep the handle and existing cancellation policy. Choose `Task.detached` only when its independent context is appropriate; naming is not a reason to detach. Names are not unique IDs. Prefer short operation labels and avoid sensitive inputs that would enter debugger output, logs, or profiling artifacts.

There is no explicit naming syntax for `async let` in SE-0469. Do not wrap its expression in an unstructured `Task` to obtain a label: that creates another task with different ownership. Use named task-group children when individual labels are needed and a group fits the workload. [SE-0469 async-let scope](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0469-task-names.md#future-directions)

## Preserve SwiftUI lifecycle and restart identity

This complete wrapper view accepts an application operation; it is not a data-loading implementation.

```swift
import SwiftUI

struct NamedLoadView: View {
    let itemID: Int
    let load: @Sendable (Int) async -> Void

    var body: some View {
        Text("Item \(itemID)")
            .task(id: itemID, name: "Load item") {
                await load(itemID)
            }
    }
}
```

The `id` value controls cancellation and recreation when it changes. The `name` is a diagnostic label, not the restart key. Do not add `Task { ... }` inside the action merely to name it, because that introduces unstructured work outside the view task's cancellation lifecycle. Cancellation remains cooperative: the supplied operation must respond appropriately. Apple documents the lifecycle and ID comparison in [task(id:name:priority:file:line:_:)](https://developer.apple.com/documentation/swiftui/view/task(id:name:priority:file:line:_:)).

On iOS/macOS 26.4+, leaving `name` as `nil` asks SwiftUI to generate a call-site label using `file` and `line`. Before 26.4, these naming arguments are ignored while the view task retains its supported lifecycle behavior. Keep the deployment fallback rather than duplicating the loading operation solely for naming.

## Verification scope

Verified 2026-10-22 against SE-0469, Apple documentation, and the installed SDK interfaces. Apple Swift 6.4 (swiftlang-6.4.0.34.1), Xcode 27.1 (27A9275), macOS 27.0.1 arm64; Swift 5 and Swift 6 language modes, complete strict concurrency, explicit nonisolated default isolation, no additional feature flags. The helper and named parent/child probes compiled and ran locally, including reading a retained handle after completion. With the current iPhoneOS 27.1 / macOS 27.0 SDKs, typechecks covered named creation at iOS 13 / macOS 10.15 and the wrapper at iOS 15 / macOS 12. Unguarded static name reads were rejected at iOS 18 / macOS 15 and accepted at 26; unguarded instance reads were rejected at 26 and accepted at 27. These checks do not test old compilers or older OS execution. Runtime display, LLDB task enumeration, and Instruments traces need separate validation; a task missing from a diagnostic view is not proof it did not exist or was cancelled.

## Sources

- [Task Names in Swift Concurrency](https://artemnovichkov.com/blog/task-names-in-swift-concurrency)
