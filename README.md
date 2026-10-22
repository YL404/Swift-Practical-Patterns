# Swift Practical Patterns

Practical Swift and SwiftUI guidance for AI coding agents, distilled from technical articles and official documentation.

This skill offers focused guidance and works well alongside other community-maintained skills. Updates are published as useful additions, corrections, and SDK changes are reviewed.

## What's included

- App integrations: background refresh, SwiftData and CloudKit identity, Core Spotlight, and in-app translation.
- Swift patterns: asynchronous cleanup, string formatting, memberwise initialization, optional protocol types, and protocol method selection.
- SwiftUI details: localized inline symbols, animation timing and timelines, system margins, and adaptive layout and presentation.

See the [skill's task routing](skills/swift-practical-patterns/SKILL.md#task-routing) for individual references.

## Installation

With Node.js 22.20 or later, run this from your app project:

```sh
npx skills add https://github.com/YL404/Swift-Practical-Patterns --skill swift-practical-patterns
```

Choose your coding agent when prompted. Add `--global` to install across projects.

You can also install the `skills/swift-practical-patterns/` folder manually, following your agent's installation instructions. The folder is self-contained and uses the [Agent Skills format](https://agentskills.io/specification).

## Usage

Ask your agent to use `swift-practical-patterns` for a matching task. For example:

> Use swift-practical-patterns to review this app's background refresh registration and cancellation handling.

> Use swift-practical-patterns to investigate duplicate seeded records in this SwiftData and CloudKit app.

The skill routes the agent to the relevant reference. Each reference states its platform requirements and validation scope; compilation, local execution, simulator checks, and device behavior are distinguished.

## Recommended companion skills

- [SwiftUI Expert Skill](https://github.com/AvdLee/SwiftUI-Agent-Skill)
- [SwiftUI Pro](https://github.com/twostraws/SwiftUI-Agent-Skill)
- [Swift Concurrency Agent Skill](https://github.com/AvdLee/Swift-Concurrency-Agent-Skill)

Choose the skills that fit your workflow. These are optional companions.

## Sources and contributions

Blog sources appear in a short `Sources` section at the end of each reference. Official documentation is linked beside the claims it supports.

Suggestions and corrections are welcome. Include the affected scenario, relevant platform and toolchain versions, supporting sources, and any reproducible evidence. Project content is maintained in English.

## License

[MIT](LICENSE)
