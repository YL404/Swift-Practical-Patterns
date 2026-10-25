# Build artifacts under Xcode disk pressure

Use this when deciding which Xcode artifacts are reproducible and which preserve evidence for a shipped build. This guidance does not implement an inventory or automatic cleanup tool.

## Separate regenerated work from release evidence

Derived Data can contain build products, indexes, logs, and package-related data. Before removing a project's generated directory, identify its owner and current use; account for the rebuild, reindex, and possible dependency-download cost. Multiple worktrees can use different directories, so a missing path from one checkout is not sufficient evidence that the directory is unused. Read-only inspection and a scoped removal proposal are implementation recommendations, not guarantees that a directory is safe to discard.

An archive is different. Apple requires retaining [the Xcode archive for each distributed build](https://developer.apple.com/documentation/xcode/building-your-app-to-include-debugging-information), including beta distributions. Each executable, framework, and extension has its own matching symbols. A newer archive, the same version string, or a source checkout does not establish that the old build's debugging evidence is replaceable. Unknown distribution or backup status is unresolved evidence, not a cleanup criterion. Preserve release archives rather than treating all generated-looking directories as disposable caches.

## Verify symbols by build UUID

The app binary and its dSYM must have matching build UUIDs. Compare the UUID and architecture, not the filename or build date. Apple's [missing-symbol workflow](https://developer.apple.com/documentation/xcode/locating-a-missing-debug-symbol-file) explains how to extract the binary's UUID from a crash report, search for its dSYM, and check the result.

Read-only command templates; replace the paths with the actual archived binary and companion dSYM:

```bash
xcrun dwarfdump --uuid "/path/to/App.xcarchive/Products/Applications/App.app/App"
xcrun dwarfdump --uuid "/path/to/App.xcarchive/dSYMs/App.app.dSYM"
```

A UUID match confirms the pairing of those files; also match the UUID in the crash report for the binary image and architecture being diagnosed. Verify embedded frameworks and extensions separately when their frames need symbolication. A different build can have different UUIDs even from the same source, so rebuilding is not a substitute for locating the distributed build's symbols.

For a known crash UUID, Apple's workflow also uses `mdfind` with `com_apple_xcode_dsym_uuids`. An empty search is not proof of absence: Spotlight indexing can be disabled, and directories with `.noindex` are excluded. Check the retained archive and its indexing conditions before declaring the symbols lost.

## Keep cleanup claims within the evidence

This archive/UUID rule does not validate the source article's automated cleanup classifications, path fingerprints, Simulator/runtime deletion, or APFS space-recovery estimates. Those require the actual tool, current resource ownership, and separate validation. No deletion command is supplied here. When measuring build changes after clearing generated data, record the resulting cache state using [xcode-build-performance.md](xcode-build-performance.md).

## Evidence and validation limits

Official archive and UUID documentation checked 2026-10-25. On arm64 macOS 27.0.1 (26A434), Xcode 27.1 (27A9275) and Apple Swift 6.4 (swiftlang-6.4.0.34.1), two tiny Swift command-line programs were compiled with debug information in Swift 6 mode, complete strict concurrency, explicit nonisolated default isolation, and warnings as errors. `dwarfdump --uuid` matched each arm64 binary to its own dSYM, and the cross-build UUIDs differed. These are local compiler artifacts, not Xcode archives or distributed apps. No storage audit, deletion, Spotlight search, crash symbolication, backup restoration, or space recovery was executed.

## Sources

- [How to free up Xcode disk space safely with an AI Agent](https://www.avanderlee.com/ai-development/how-to-free-up-xcode-disk-space-safely-with-an-ai-agent/) (2026-07-27)
