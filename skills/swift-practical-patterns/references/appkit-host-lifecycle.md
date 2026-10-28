# Nested AppKit hosting teardown

Use this when an existing `NSViewRepresentable` owns an `NSHostingView`, or a long-lived AppKit view moves between SwiftUI-backed hosts. Do not introduce a bridge solely to apply this guidance. These APIs require macOS 10.15+; the reported reconstruction issue below is version-specific evidence, not an API guarantee.

## Tear down the host you own

Apple defines [`dismantleNSView(_:coordinator:)`](https://developer.apple.com/documentation/swiftui/nsviewrepresentable/dismantlensview(_:coordinator:)) as cleanup **in anticipation of removal**. Use that callback to detach an owned inner hosting view and release your reference. [`removeFromSuperview()`](https://developer.apple.com/documentation/appkit/nsview/removefromsuperview()) unlinks the view from its superview and window and removes affected constraints; it must not be called during display. Do not wait for deinitialization to begin this cleanup.

This complete pair of view types demonstrates ownership and teardown. It updates an existing inner host instead of replacing it on each update. Place it in a sized region, for example `NestedContentHost(content: Text("Notes")).frame(width: 240, height: 160)`. It does not forward intrinsic size or implement a shared-view transfer protocol or application-specific delegate/observer cleanup.

```swift
import AppKit
import SwiftUI

@available(macOS 10.15, *)
@MainActor
struct NestedContentHost<Content: View>: NSViewRepresentable {
    var content: Content

    func makeNSView(context: Context) -> OwnedHostingContainer<Content> {
        OwnedHostingContainer(frame: .zero)
    }

    func updateNSView(_ container: OwnedHostingContainer<Content>, context: Context) {
        container.update(content)
    }

    static func dismantleNSView(
        _ container: OwnedHostingContainer<Content>, coordinator: ()
    ) {
        container.detach()
    }
}

@available(macOS 10.15, *)
@MainActor
final class OwnedHostingContainer<Content: View>: NSView {
    private var host: NSHostingView<Content>?

    func update(_ content: Content) {
        if let host {
            host.rootView = content
            return
        }
        let host = NSHostingView(rootView: content)
        host.translatesAutoresizingMaskIntoConstraints = false
        addSubview(host)
        NSLayoutConstraint.activate([
            host.leadingAnchor.constraint(equalTo: leadingAnchor),
            host.trailingAnchor.constraint(equalTo: trailingAnchor),
            host.topAnchor.constraint(equalTo: topAnchor),
            host.bottomAnchor.constraint(equalTo: bottomAnchor)
        ])
        self.host = host
    }

    func detach() {
        host?.removeFromSuperview()
        host = nil
    }
}
```

The callback cleans up this container's host. It is not a global barrier against queued callbacks, work retained elsewhere, or new representable instances. Stop owned observations/tasks separately as required by the actual integration, and do not tear down an external reusable resource that has already moved to another owner.

## Separate construction from shared-resource ownership

For a terminal surface, player view, or other shared `NSView`, avoid making every wrapper initializer unconditionally take it. Define an explicit owner and transfer point. An old wrapper's cleanup should release the resource only if that wrapper still owns it. Keep terminal/session lifetime separate from the disposable host's lifetime.

When diagnosing a stranded view, log host identity, the resource's superview/window, make/update/dismantle, and `viewDidMoveToWindow`. [`NSView.window`](https://developer.apple.com/documentation/appkit/nsview/window) establishes window membership, not visibility or user intent. A non-`nil` window alone does not select the rightful owner when several windows or hosts exist. Do not adopt a universal “newest host wins” or “any window host wins” rule.

Onevcat's [reproducer at revision 8c66eb25d5d659ec1eb5305fa058a5e6b3631efd](https://github.com/onevcat/NestedHostingViewRepro/tree/8c66eb25d5d659ec1eb5305fa058a5e6b3631efd) reports that an inner host can construct a new representable leaf after the outer dismantle callback, while AppKit still has the inner host in its window. Its wrapper initializer then moves a shared view away from the intended destination. The author reports this on macOS 27.0.1 (26A434), Xcode 27.0 (27A266a), Swift 6.4: eight nested configurations fail; direct embedding and explicit detach controls pass. The empty `.task` exposed the issue in the application but is not required by the small reproducer. These are author observations, not promises about every nested host or SwiftUI release.

Reproduce the actual entry path before applying a workaround. Compare nesting versus direct content and detach versus retained inner host; keep shared-resource ownership assertions separate from callback counts. Test repeated transitions, overlapping hosts, closing, and multiwindow transfers under the affected runtime. Preserve accessibility, focus, and responder behavior when changing an existing bridge.

## Verification limits

Official cleanup/window documentation and the pinned reproducer source were read on 2026-10-28. The complete example above passed Swift 6 typechecking with complete strict concurrency, warnings as errors, and explicit nonisolated/MainActor default isolation, targeting arm64 macOS 10.15 with the macOS 27.0 SDK and Xcode 27.1 (27A9275), Apple Swift 6.4, on macOS 27.0.1 arm64. No window was launched and no reproduction matrix, resource transfer, visual behavior, focus, or accessibility test was run locally. Typechecking does not verify that detach fixes a particular application or prevents all future inner-graph work.

## Sources

- [When Apps Are No Longer Tied to a Specific Operating System — Fatbobman's Swift Weekly #159](https://fatbobman.com/en/weekly/issue-159/)
- [A dismantled NSHostingView can still create its view again](https://onevcat.com/2026/10/nested-hosting-view/)
