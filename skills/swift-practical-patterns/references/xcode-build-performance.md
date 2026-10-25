# Xcode build timing and compilation caching

Use this when builds are slow, a no-change build repeats work, or enabling a cache appears to regress a clean build. App runtime traces measure a different problem.

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

## Evidence and validation limits

Official documentation and Xcode 27.1 (27A9275) command help checked 2026-10-25 on arm64 macOS 27.0.1. The timing and effective-settings options were confirmed in local help. No app project was benchmarked, no settings or build phases were changed, and no cache hit rate or speedup was measured. The workload protocol above is implementation advice; the source article's benchmark numbers and claim of a normal initial slowdown were not reproduced.

## Sources

- [Xcode Build Optimization using 6 Agent Skills](https://www.avanderlee.com/xcode/xcode-build-optimization-using-6-agent-skills/) (2026-03-30; updated 2026-04-16)
