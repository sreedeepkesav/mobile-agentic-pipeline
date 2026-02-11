# Skills

Four self-contained Claude Code skills for AI-assisted mobile development.

| Skill | Platform | Type | Best For |
|-------|----------|------|----------|
| `ios-full` | iOS (Swift/SwiftUI) | Full | Production apps, teams |
| `ios-lite` | iOS (Swift/SwiftUI) | Lite | Side projects, MVPs |
| `android-full` | Android (Kotlin/Compose) | Full | Production apps, teams |
| `android-lite` | Android (Kotlin/Compose) | Lite | Side projects, MVPs |

## Structure

Each skill is self-contained:

```
{skill}/
├── SKILL.md          # Workflow overview (<500 lines)
├── references/       # Detailed guides (loaded on demand)
└── evals/            # Test scenarios
```

Full skills have 10 reference files (bootstrap, coordinator routing, product agent, code gen phases, pipeline memory, platform standards, architecture templates, test+lint, build+deploy, git workflow).

Lite skills have 6 (product-design flow, code gen phases, platform standards, architecture templates, build+git, feedback loops).

## Installation

```bash
cp -r ios-full ~/.claude/skills/ios-full-pipeline
```

For project-level defaults, copy to `.claude/skills/` in the project root instead.

## Invocation

Name the pipeline explicitly: `"Use ios-full-pipeline to add dark mode"`. See the [usage guide](../docs/usage-guide.md) for all invocation methods, single-agent scenarios, and configuration.

## Contributing

1. Keep SKILL.md under 500 lines — move detail to `references/`
2. Imperative style ("Create the entity", not "Creating the entity")
3. Concrete examples, not generic descriptions
4. No cross-platform contamination (no Swift in Android files)
5. Test with eval prompts before committing
