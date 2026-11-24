# Xcode build timing, caching, and MCP vs `xcodebuild`

Use this when builds are slow, a no-change build repeats work, enabling a cache appears to regress a clean build, or choosing between Xcode's built-in MCP server and `xcodebuild` for local agent loops versus CI. App runtime traces measure a different problem.

## Define the build being compared

Keep the Xcode version, SDK, scheme, configuration, destination, architecture, package resolution, and build command constant. Record the source revision and Derived Data location. Measure these workloads separately:

- A build after cleaning its build products, with the compilation cache state recorded.
- A rebuild with no source changes.
- A rebuild after the same representative source edit.

This is a measurement recommendation, not an Xcode requirement. Repeat each workload enough to expose variation and retain individual durations alongside a median or range. A practical starting point is three runs per workload. Keep cache warm-up separate; a clean build does not by itself establish that every cache is cold. Avoid concurrent builds and record background work that affects the comparison.

## Diagnose repeated work before changing settings

Use Product > Perform Action > Build With Timing Summary, or append `-showBuildTimingSummary` to the project's existing successful `xcodebuild` invocation. Keep total elapsed time as well as task timings; summing parallel task durations is not elapsed time. Apple's [incremental-build guidance](https://developer.apple.com/documentation/xcode/improving-the-speed-of-incremental-builds) explains timing summaries, script dependencies, and target ordering.

For an unexpectedly slow no-change build, inspect preparation and scripts that run again. A script's declared inputs and outputs must describe its actual dependencies and generated files. Keep dependency analysis enabled for a script that can be skipped, and use `.xcfilelist` files when lists are long. Validate that relevant input edits regenerate the output and an unchanged rebuild skips the work. Do not add a dummy output that hides a generator's undeclared inputs.

Review unnecessary target dependencies only when timings justify it. Preserve real build-order dependencies; removing one can produce stale embedded code. Splitting targets is a measured tradeoff with added project complexity, not a universal optimization. Rebuild and run affected tests after graph or script changes.

## Evaluate compilation caching under the actual toolchain

[Xcode 26 release notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-26-release-notes) introduced compilation caching as opt-in, reusing compilation results for matching inputs and particularly benefiting branch switches and clean builds. That historical default does not establish the setting in a later Xcode or an existing project. Check the selected toolchain and effective settings for the actual configuration.

The [build settings reference](https://developer.apple.com/documentation/xcode/build-settings-reference#Enable-Compilation-Caching) names `COMPILATION_CACHE_ENABLE_CACHING`; its adjacent `COMPILATION_CACHE_ENABLE_DIAGNOSTIC_REMARKS` setting emits cache diagnostics. Read current values through Xcode's settings or the same scheme/configuration with `xcodebuild -showBuildSettings`. When evaluating a change, compare equivalent cache states and inspect diagnostics rather than attributing any timing change to cache hits.

Retest clean, no-change, and edited builds independently. A slower first run is not proof of a beneficial warm cache, and a faster no-change build does not establish faster edit builds. Report the measured workloads and correctness checks; do not reuse another project's speedup or promise that caching always helps. Storage cleanup can change the next measurement; see [build-artifact-retention.md](build-artifact-retention.md).

## Choose between Xcode MCP and `xcodebuild` in local loops

Xcode's MCP bridge (`xcrun mcpbridge`) connects external coding agents to Xcode's build and test tools, and Xcode 27 Beta 5+ adds a preview headless server (`xcrun mcp-server`) that runs without an open workspace window. [External agent setup](https://developer.apple.com/documentation/xcode/giving-external-agents-access-to-xcode), [Xcode 27 release notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27-release-notes)

Under the hood, [Swift Build's architecture](https://github.com/swiftlang/swift-build/blob/main/SwiftBuild.docc/Architecture/build-system-architecture.md) specifies that clients spawn one `SWBBuildService` process per client process on demand, and `BuildDescriptionManager` reuses cached in-memory or on-disk build descriptions when inputs match. Consequently, a one-shot `xcodebuild` invocation starts a fresh build-service process each time, whereas a long-running Xcode or headless MCP service can retain process state between builds. Both invoke the same compiler work; any latency advantage comes from per-invocation startup/planning overhead rather than faster compilation, so it matters most for repeated short builds and small test runs and diminishes on large test suites.

Use these operational boundaries when structuring agent and developer workflows (implementation advice informed by single-project observations, not Apple performance guarantees):

- **Use one tool per worktree:** avoid alternating between Xcode MCP (`BuildProject` / `RunSomeTests`) and `xcodebuild` against the same workspace and DerivedData. Author measurements on a ~200-target workspace showed large first-run slowdowns when switching tools (likely from differing build settings triggering extra DerivedData work), erasing steady-state savings.
- **Keep `xcodebuild` for CI and as a fallback:** use `xcodebuild` on CI, headless environments without the Xcode MCP service, or when you need result bundles and reproducible raw logs. When agents invoke `xcodebuild`, filter its output (for example with a log parser) so megabytes of raw build logs do not flood the context window; Xcode MCP returns compact structured error summaries by default.
- **Verify toolchain alignment before comparing or debugging:** confirm that `MCP_XCODE_PID` (or `xcode-select`) for `xcrun mcpbridge` matches the `DEVELOPER_DIR` used by `xcodebuild`. Comparing different Xcode installations invalidates measurements and can churn shared caches.
- **Account for shared server state and preview limits:** active scheme and test plan selection live in the Xcode MCP server, so concurrent agents sharing one server should not switch schemes mid-session. In headless mode, grant directory-tree access explicitly (`sudo xcrun mcp-server enable`, `sudo xcrun mcp-server allow-folder <path> --always`, inspect with `xcrun mcp-server status`) rather than approving every new worktree interactively; reserve `--unsafe-always-allow-all-agents` for unattended environments as Apple warns. Apple's Xcode 27 release notes mark `xcrun mcp-server` as an early preview where some configurations or permission updates may require relaunching Xcode or rebooting.

## Evidence and validation limits

Official documentation, Swift Build architecture documentation, and Xcode 27.2 (27B52) command help checked 2026-11-24 on arm64 macOS 27.2. Local CLI checks confirmed `xcodebuild` timing/settings flags, `xcrun mcpbridge --help` (`MCP_XCODE_PID`, `MCP_XCODE_SESSION_ID`), `xcrun mcp-server --help`, `xcrun mcp-server allow-folder --help` (`--always`, `--for-24-hours`), and read-only `xcrun mcp-server status`.

No app project was benchmarked, no settings or build phases were changed, and no cache hit rate, MCP speedup, tool-switching penalty, or intermittent empty test response was reproduced locally. The workload protocol and MCP-versus-`xcodebuild` workflow guidance are implementation advice; the source articles' numeric timings and hypothesized causes are single-project observations, not verified generalizations.

## Sources

- [Xcode Build Optimization using 6 Agent Skills](https://www.avanderlee.com/xcode/xcode-build-optimization-using-6-agent-skills/) (2026-03-30; updated 2026-04-16)
- [Xcode MCP vs xcodebuild: build and test speed compared](https://www.avanderlee.com/ai-development/xcode-mcp-vs-xcodebuild/) (2026-11-23)
