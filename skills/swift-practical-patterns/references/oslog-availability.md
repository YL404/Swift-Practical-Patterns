# Checking OSLog levels safely across OS deployment targets

Use this reference when an app deploys to macOS, iOS, or other Apple OS versions earlier than 26.4 and needs to test whether a log level is enabled before performing expensive formatting or diagnostic work. Ordinary `Logger.debug(...)` / `Logger.info(...)` calls with lightweight interpolations already defer message construction through `OSLogMessage` and do not need a manual check.

Apple documentation and Xcode 27.2 SDK interfaces verified 2026-11-24.

## Do not call `Logger.isEnabled(type:)` when targeting pre-26.4 OS releases

Apple's documentation declares [`Logger.isEnabled(type:)`](https://developer.apple.com/documentation/os/logger/isenabled(type:)) as introduced in **iOS 26.4, iPadOS 26.4, Mac Catalyst 26.4, macOS 26.4, tvOS 26.4, visionOS 26.4, and watchOS 26.4**. Apple Developer Technical Support also confirmed on the [Apple Developer Forums](https://developer.apple.com/forums/thread/855301?answerId=914801022#914801022) that the symbol is absent on macOS 26.2 and present by 26.7.1 (introduced in 26.4), and filed a bug for the availability mismatch.

In Xcode 27.2 (`MacOSX27.2.sdk` and `iPhoneOS27.2.sdk`), however, `os.swiftmodule` declares `Logger.isEnabled(type:)` inside `@available(macOS 11.0, iOS 14.0, watchOS 7.0, tvOS 14.0, *) public struct Logger` without its own 26.4 availability attribute. This creates a silent backward-compatibility trap:

- **No compiler availability error:** compiling with a lower deployment target (such as macOS 15.0 or iOS 18.0) succeeds without diagnostics.
- **`if #available` does not make the symbol weak-linked:** because the SDK claims the symbol existed since macOS 11 / iOS 14, wrapping the call in `if #available(macOS 26.4, iOS 26.4, *)` still emits a **strong (non-weak) external import** of `_$s2os6LoggerV9isEnabled4typeSbSo0a5_log_E2_ta_tF` from `/usr/lib/swift/libswiftos.dylib`.
- **Launch-time crash on older OS versions:** when the app launches on an OS whose `libswiftos.dylib` lacks that symbol (for example macOS 15.6.1 or macOS 26.2), `dyld` terminates the process immediately with `Symbol not found` before any app code or `#available` check runs.

Until a future SDK fixes the availability annotation and your build uses it—or your deployment target is 26.4+ across every supported platform—do not reference `Logger.isEnabled(type:)` in shipped code.

## Retain an `OSLog` handle and construct `Logger` from it

[`OSLog.isEnabled(type:)`](https://developer.apple.com/documentation/os/oslog/isenabled(type:)) maps directly to the C function `os_log_type_enabled` in `libSystem`, which has been available since **macOS 10.12, iOS 10.0, tvOS 10.0, watchOS 3.0, and visionOS 1.0** (`Logger`'s own inlinable logging methods call `logObject.isEnabled(type:)` internally). Keep the `OSLog` handle alongside the `Logger` built from it, and query the handle before doing expensive diagnostic work.

Complete reusable wrapper for macOS 11+ / iOS 14+ deployment targets. Pass a reverse-DNS subsystem and feature category; guard only genuinely expensive work (such as large summaries or serialization) that should not run when debug logging is off.

```swift
import OSLog

@available(macOS 11.0, iOS 14.0, tvOS 14.0, watchOS 7.0, *)
struct ScopedLog: Sendable {
    let handle: OSLog
    let logger: Logger

    init(subsystem: String, category: String) {
        let handle = OSLog(subsystem: subsystem, category: category)
        self.handle = handle
        self.logger = Logger(handle)
    }

    func logExpensiveDebug(_ buildSummary: () -> String) {
        guard handle.isEnabled(type: .debug) else { return }
        let summary = buildSummary()
        logger.debug("\(summary)")
    }
}
```

Keep the default private interpolation for user or sensitive strings; add `privacy: .public` only for explicitly non-sensitive diagnostics.

## Audit linked imports when targeting older OS releases

Compiler availability checks only enforce what the SDK's `.swiftinterface` and headers declare. When shipping binaries built with a newer Xcode to older OS deployment targets, inspect non-weak external imports against the minimum supported OS (for example with `nm -m <binary>` or `dyld_info -imports <binary>`) and verify launch on the oldest supported OS release.

## Verification and remaining boundaries

Inspected `MacOSX27.2.sdk` and `iPhoneOS27.2.sdk` in Xcode 27.2 (27B52), Apple Swift 6.4.2 (swiftlang-6.4.2.1.9) on 2026-11-24:

- `os.swiftmodule` in both SDKs declares `public func isEnabled(type: os::OSLogType) -> Swift::Bool` directly on `@available(macOS 11.0, iOS 14.0, watchOS 7.0, tvOS 14.0, *) public struct Logger` without a 26.4 availability attribute, and `libswiftos.tbd` exports `_$s2os6LoggerV9isEnabled4typeSbSo0a5_log_E2_ta_tF`.
- Linking test dylibs for `arm64-apple-macos15.0` and `arm64-apple-ios18.0` with `if #available(macOS 26.4, iOS 26.4, *) { logger.isEnabled(type: .debug) }` produced `(undefined) external _$s2os6LoggerV9isEnabled4typeSbSo0a5_log_E2_ta_tF (from libswiftos)` (strong import), whereas a control call to `@available(iOS 26.0, macOS 26.0, *) ScrollPosition.y` inside `if #available` in the same binary produced `(undefined) weak external` as expected.
- The `ScopedLog` code block above was extracted verbatim and passed `swiftc -typecheck` targeting arm64 macOS 11.0 / 15.0 and arm64 iOS 14.0 / 18.0 under Swift 6 (`nonisolated` and `MainActor` default isolation) and Swift 5 (`nonisolated` default isolation), complete strict concurrency, and warnings as errors. Linking it into macOS 15.0 and iOS 18.0 dylibs imported `_os_log_type_enabled (from libSystem)` with no reference to `_$s2os6LoggerV9isEnabled4typeSbSo0a5_log_E2_ta_tF`.
- A local macOS 27.2 executable instantiated `ScopedLog`, confirmed `logExpensiveDebug` skipped the closure when `.debug` logging was disabled (`isEnabled(type: .debug) == false`), and confirmed `handle.isEnabled(type: .error) == true` on the host OS.

The launch-time `dyld` crash on macOS 15.6.1, macOS 26.2, or pre-26.4 iOS devices was not reproduced locally because the host runs macOS 27.2; older-OS crash behavior relies on the verified strong-import linkage plus the linked Apple DTS and developer reports.

## Sources

- [A Logger symbol missing on macOS 15 kept my Mac app from launching](https://yardlet.app/blog/logger-isenabled-macos-15.html) (2026-11-01; updated 2026-11-05)
- [Missing Logger Symbol on macOS 15](https://mjtsai.com/blog/2026/11/02/missing-logger-symbol-on-macos-15/) (2026-11-02; updated 2026-11-09)
- [iOS Code Review Issue #94](https://ioscodereview.com/issues/issue-94-cocoapods-closes-next-week-siri-can-call-your-intents-and-prices-move-december-3/) (2026-11-23)
