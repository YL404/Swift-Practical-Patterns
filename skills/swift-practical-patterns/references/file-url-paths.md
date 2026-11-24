# Extracting file paths from Swift `URL`s

Use this reference when replacing deprecated `URL.path` or converting a local `file://` `URL` into a filesystem path string or C string for APIs that cannot accept `URL` directly (`Process` arguments, POSIX/C functions, SQLite, third-party libraries, or display/logging). Stay in `URL` whenever Apple APIs accept `URL` directly (`FileManager`, `Data(contentsOf:)`, `FileHandle(forReadingFrom:)`).

Apple documentation, `swift-foundation` issue records, and Xcode 27.2 SDK interfaces verified 2026-11-24.

## Do not replace `URL.path` mechanically with `URL.path()` or bare `URL.path(percentEncoded: false)`

In Xcode 27.2 SDKs, [`URL.path`](https://developer.apple.com/documentation/foundation/url/path) is marked for future deprecation (`Use path(percentEncoded:) instead`). Replacing `.path` with `.path()` or bare `.path(percentEncoded: false)` changes behavior in three concrete ways:

1. **Flipped percent-encoding default:** [`URL.path(percentEncoded:)`](https://developer.apple.com/documentation/foundation/url/path(percentencoded:)) (macOS 13 / iOS 16 / tvOS 16 / watchOS 9 / visionOS 1+) defaults to `percentEncoded: true`, whereas deprecated `URL.path` returned an unencoded path. Calling bare `url.path()` turns spaces, `#`, `%`, and non-ASCII characters into percent escapes (such as `%20`, `%23`, `%25`), silently breaking filesystem calls once a path contains spaces or punctuation.
2. **Undocumented OS-version behavior change on relative URLs (`baseURL != nil`):** APIs such as `Bundle.url(forAuxiliaryExecutable:)`, `URL(filePath:relativeTo:)`, and `URL(string:relativeTo:)` return relative file URLs carrying a `baseURL`. Deprecated `URL.path` explicitly resolves against `baseURL`. On earlier Apple OS releases before the Swift Foundation `URL` rewrite rolled out (documented in [swift-foundation #1011](https://github.com/swiftlang/swift-foundation/issues/1011) and [Swift Forums #75358](https://forums.swift.org/t/url-path-vs-path-percentencoded-for-file-urls/75358), including macOS 15), `url.path(percentEncoded: false)` did **not** resolve against `baseURL` and returned only the relative component (for example `"tool"` instead of `"/Applications/MyApp.app/Contents/tool"`). Newer OS releases (verified on macOS 27.2; reported from macOS 26+) resolve against `baseURL`, but Apple's `path(percentEncoded:)` documentation still omits which OS version changed behavior. When supporting mixed OS deployment targets, resolve [`absoluteURL`](https://developer.apple.com/documentation/foundation/url/absoluteurl) explicitly instead of relying on runtime OS differences.
3. **Preserved trailing slash on directory URLs:** Deprecated `URL.path`, `FilePath(url)?.string`, and `withUnsafeFileSystemRepresentation` strip the trailing `/` from non-root directory URLs (returning `"/Users/Jane Doe/My Docs"`), whereas `path(percentEncoded: false)` preserves it (`"/Users/Jane Doe/My Docs/"`; root `"/"` stays `"/"`). Audit string equality, dictionary keys, prefix checks, and CLI flags when migrating existing `.path` call sites.

## Choose the conversion for the API boundary

Check `url.isFileURL` before extracting a filesystem path: `path(percentEncoded:)`—and even `withUnsafeFileSystemRepresentation` on macOS 27.2—still return a path string for non-file URLs such as `https://example.com/a%20b/c`.

| Need | Prefer | Minimum OS | Trailing `/` on dirs | Rejects non-file URLs |
|---|---|---|---|---|
| Cross-OS Swift `String` path | `url.absoluteURL.path(percentEncoded: false)` (after `isFileURL` check) | iOS 16 / macOS 13 | Preserved | No (check `isFileURL`) |
| Clean `.` / `..` segments too | `url.absoluteURL.standardizedFileURL.path(percentEncoded: false)` | iOS 16 / macOS 13 | Preserved | No (`standardizedFileURL` returns `self`) |
| Typed path or `System` APIs | [`FilePath`](https://developer.apple.com/documentation/system/filepath)`(url.absoluteURL)` -> `.string` | iOS 14 / macOS 11 | Stripped | Yes (returns `nil`) |
| Direct POSIX / C API call | [`url.withUnsafeFileSystemRepresentation`](https://developer.apple.com/documentation/foundation/url/withunsafefilesystemrepresentation(_:)) (after `isFileURL` check) | iOS 8 / macOS 10.10 | Stripped | No (check `isFileURL`) |

Keep these edge-case boundaries explicit:

- **`standardizedFileURL` vs. `resolvingSymlinksInPath()`:** [`standardizedFileURL`](https://developer.apple.com/documentation/foundation/url/standardizedfileurl) removes `.` and `..` segments and removes `/private` for system aliases such as `/private/tmp` -> `/tmp`, without generally resolving symlinks. Call [`resolvingSymlinksInPath()`](https://developer.apple.com/documentation/foundation/url/resolvingsymlinksinpath()) only when you intentionally want filesystem I/O to resolve symlinks to canonical targets.
- **Invalid percent-encoded UTF-8:** On macOS 27.2, a malformed file URL containing invalid UTF-8 percent bytes (such as `file:///tmp/bad%FFname`) causes `path(percentEncoded: false)`, `FilePath(url)?.string`, and `withUnsafeFileSystemRepresentation` to return an empty string `""` rather than `nil`. Reject empty paths when handling untrusted URLs.
- **C string lifetime:** The pointer passed to `withUnsafeFileSystemRepresentation` is a null-terminated canonical UTF-8 C string valid only inside the closure. Do not let the pointer escape.
- **`FilePath.init?(_:)` and relative URLs:** [`FilePath(_:)`](https://developer.apple.com/documentation/system/filepath/init(_:)-2gkpw) (`import System`) returns `nil` when `url` does not refer to a local file and normalizes redundant or trailing separators on construction. Passing `url.absoluteURL` avoids depending on whether older OS runtimes resolve `baseURL` inside the initializer.

Complete helper covering the three common boundaries for iOS 16+ / macOS 13+ (or use only `withUnsafeFileSystemRepresentation` / `FilePath` when deploying to older OS versions).

```swift
import Foundation
import System

@available(iOS 16.0, macOS 13.0, tvOS 16.0, watchOS 9.0, *)
enum LocalFilePath {
    static func string(from url: URL, standardize: Bool = false) -> String? {
        guard url.isFileURL else { return nil }
        let resolved = standardize ? url.absoluteURL.standardizedFileURL : url.absoluteURL
        let path = resolved.path(percentEncoded: false)
        return path.isEmpty ? nil : path
    }

    static func filePath(from url: URL) -> FilePath? {
        guard let path = FilePath(url.absoluteURL), !path.isEmpty else { return nil }
        return path
    }

    static func withPOSIXPath<Result>(
        from url: URL,
        _ body: (UnsafePointer<CChar>) throws -> Result
    ) rethrows -> Result? {
        guard url.isFileURL else { return nil }
        return try url.absoluteURL.withUnsafeFileSystemRepresentation { pointer in
            guard let pointer, pointer.pointee != 0 else { return nil }
            return try body(pointer)
        }
    }
}
```

## Verification and remaining boundaries

The `LocalFilePath` helper above was extracted verbatim and passed `swiftc -typecheck` with Xcode 27.2 (27B52), Apple Swift 6.4.2 (swiftlang-6.4.2.1.9), targeting arm64 iOS 16.0 (`iPhoneOS27.2.sdk`) and arm64 macOS 13.0 (`MacOSX27.2.sdk`) under Swift 6 (`nonisolated` and `MainActor` default isolation) and Swift 5 (`nonisolated` default isolation), with complete strict concurrency and warnings as errors.

Inspected `MacOSX27.2.sdk` `Foundation.swiftinterface` on 2026-11-24: `URL.path` is declared `@available(..., deprecated: 100000.0, message: "Use path(percentEncoded:) instead")`, `URL.path(percentEncoded: Bool = true)` is available from macOS 13 / iOS 16 / tvOS 16 / watchOS 9, and `System.FilePath.init?(_ url: Foundation.URL)` is available from macOS 11 / iOS 14 / tvOS 14 / watchOS 7.

A local Swift probe executed on arm64 macOS 27.2 (Xcode 27.2) confirmed:

- `URL(filePath: "/Users/Jane Doe/My Docs/report #1 (100%).pdf")`: `path()` returned `"/Users/Jane%20Doe/My%20Docs/report%20%231%20(100%25).pdf"`; `path(percentEncoded: false)`, `absoluteURL.path(percentEncoded: false)`, `FilePath(url)?.string`, and `withUnsafeFileSystemRepresentation` all returned `"/Users/Jane Doe/My Docs/report #1 (100%).pdf"`.
- `URL(filePath: "/Users/Jane Doe/My Docs/", directoryHint: .isDirectory)`: deprecated `path`, `FilePath(url)?.string`, and `withUnsafeFileSystemRepresentation` returned `"/Users/Jane Doe/My Docs"`, while `path(percentEncoded: false)` and `absoluteURL.path(percentEncoded: false)` returned `"/Users/Jane Doe/My Docs/"`. Root `URL(filePath: "/")` returned `"/"` across all APIs.
- Relative file URLs (`URL(string: "Library", relativeTo: URL(fileURLWithPath: "/System"))!`, `URL(filePath: "sub dir/my tool", relativeTo: ...)`, and `Bundle(path: "/System/Applications/Calculator.app")?.url(forAuxiliaryExecutable: "Calculator")`): all carried non-nil `baseURL` and resolved to full absolute paths on macOS 27.2 across `path(percentEncoded: false)`, `absoluteURL.path(percentEncoded: false)`, `FilePath(url)`, and `withUnsafeFileSystemRepresentation`.
- `URL(filePath: "/tmp/foo/../bar/./baz.txt")`: `absoluteURL.path(percentEncoded: false)` kept `"/tmp/foo/../bar/./baz.txt"`, whereas `standardizedFileURL.path(percentEncoded: false)` returned `"/tmp/bar/baz.txt"`. On a file reference URL bridged from `/tmp/...`, `path(percentEncoded: false)` returned `"/private/tmp/..."` while `standardizedFileURL.path(percentEncoded: false)` returned `"/tmp/..."`.
- `URL(string: "https://example.com/a%20b/c")!`: `isFileURL` was `false`; `FilePath(url)` returned `nil`, whereas `path(percentEncoded: false)` and `withUnsafeFileSystemRepresentation` both returned `"/a b/c"`.
- `URL(string: "file:///tmp/bad%FFname")!`: `path(percentEncoded: false)` returned `""`, `FilePath(url)` returned an empty `FilePath("")`, and `withUnsafeFileSystemRepresentation` yielded an empty C string `""`.

Pre-macOS 26 / iOS 16–18 runtime behavior where `url.path(percentEncoded: false)` returned an unresolved relative path was not executed directly because this host runs macOS 27.2; that historical behavior relies on [swift-foundation #1011](https://github.com/swiftlang/swift-foundation/issues/1011) (including maintainer confirmation) and developer reports. Windows path separators and non-Apple Foundation runtimes were not exercised locally.

## Sources

- [Getting a File Path From a Swift URL](https://mjtsai.com/blog/2026/10/20/getting-a-file-path-from-a-swift-url/)
- [iOS Code Review Issue #93](https://ioscodereview.com/issues/issue-93-ios-27-2-is-out-uiview-margins-went-to-zero-and-the-duo-is-in-peoples-hands/)
