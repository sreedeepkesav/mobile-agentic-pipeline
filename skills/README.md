# Claude Code Skills — Mobile Agentic Pipeline

Four self-contained skills for AI-assisted iOS and Android development. Each skill orchestrates a complete development pipeline from idea to PR.

## Skills Overview

| Skill | Platform | Type | Stages | Best For |
|-------|----------|------|--------|----------|
| `ios-full` | iOS (Swift/SwiftUI) | Full | Bootstrap → Coordinator → Product → Code Gen → Test ‖ Lint → Build → Deploy → Git | Production apps, teams |
| `ios-lite` | iOS (Swift/SwiftUI) | Lite | Product+Design → Coder → Build → Git | Side projects, MVPs |
| `android-full` | Android (Kotlin/Compose) | Full | Bootstrap → Coordinator → Product → Code Gen → Test ‖ Lint → Build → Deploy → Git | Production apps, teams |
| `android-lite` | Android (Kotlin/Compose) | Lite | Product+Design → Coder → Build → Git | Side projects, MVPs |

## Installation

Copy the skill folder to your Claude Code skills directory:

```bash
# Pick the one(s) you need:
cp -r ios-full ~/.claude/skills/ios-full-pipeline
cp -r ios-lite ~/.claude/skills/ios-lite-pipeline
cp -r android-full ~/.claude/skills/android-full-pipeline
cp -r android-lite ~/.claude/skills/android-lite-pipeline
```

## How Skills Work

### Triggering

Claude Code reads the `description` field in each SKILL.md's YAML frontmatter to decide when to activate a skill. Natural language prompts trigger the right skill automatically:

- "Add a login screen to my iOS app" → `ios-full-pipeline`
- "Quick Android prototype for a timer" → `android-lite-pipeline`
- "Fix the crash in the settings screen" → routes through coordinator (Full) or coder (Lite)

### Progressive Disclosure

Each skill uses a 3-level system:

1. **Frontmatter** (~5 lines) — always loaded, used for skill selection
2. **SKILL.md body** (<500 lines) — loaded when skill triggers, provides workflow overview
3. **references/** (unlimited) — loaded on demand when Claude needs detailed guidance

This keeps context windows efficient while providing comprehensive knowledge when needed.

## Skill Structure

### Full Pipeline Skills (ios-full, android-full)

```
{platform}-full/
├── SKILL.md                         # Workflow overview, routing table, standards summary
├── references/
│   ├── bootstrap-and-config.md      # Bootstrap steps, .pipeline.yml schema
│   ├── coordinator-routing.md       # 9 task routing paths with decision tree
│   ├── product-agent.md             # Feature decomposition, spec writing, review gate
│   ├── code-gen-phases.md           # 4-phase team: Architect → Domain → Data‖Pres → Integration
│   ├── pipeline-memory.md           # Cross-run learning, what agents read/write
│   ├── {platform}-standards.md      # Architecture, DI, naming, layers, lint rules
│   ├── {platform}-architecture-template.md  # Project structure, code templates
│   ├── test-and-lint.md             # Test + Lint agents, parallel execution
│   ├── build-and-deploy.md          # Build compilation, signing, deploy targets
│   └── git-workflow.md              # Branching, conventional commits, PR templates
└── evals/
    └── evals.json                   # 5 test scenarios
```

### Lite Pipeline Skills (ios-lite, android-lite)

```
{platform}-lite/
├── SKILL.md                         # Workflow overview, standards summary
├── references/
│   ├── product-design-flow.md       # Idea → spec → design tokens → review gate
│   ├── code-gen-phases.md           # 4-phase coder (same phases, simpler context)
│   ├── {platform}-standards.md      # Architecture, DI, naming, layers
│   ├── {platform}-architecture-template.md  # Project structure, code templates
│   ├── build-and-git.md             # Build + Git combined
│   └── feedback-loops.md            # 5 self-healing patterns
└── evals/
    └── evals.json                   # 3 test scenarios
```

## Full vs Lite Comparison

| Feature | Full | Lite |
|---------|------|------|
| Bootstrap (auto-config) | Yes | No |
| Pipeline Memory | Yes (learns across runs) | No |
| Coordinator (smart routing) | Yes (9 paths) | No (single flow) |
| Product Agent | Configurable (always/never/per-task) | Always (built-in) |
| Code Gen Team | 5 sub-agents, 4 phases | Same 4 phases, simpler |
| Test Agent | Configurable | Built into Integration |
| Lint Agent | Configurable | Built into Integration |
| Build Agent | Configurable | Always |
| Deploy Agent | Configurable | No |
| Sprint Batch (parallel) | Yes | No |
| Stage Configuration | `.pipeline.yml` | None needed |
| Human Review Gates | 2 (spec + PR) | 2 (spec + PR) |

## Code Gen Team (All Skills)

All 4 skills share the same 4-phase code generation structure:

```
Phase 1: Architect (Principal)
  → Researches, plans, writes ADR. No code until done.

Phase 2: Domain Lead (Senior)
  → Pure business logic. Zero framework imports.

Phase 3: Data Lead ‖ Presentation Lead (Parallel, Senior)
  → Data: DTOs, mappers, API, persistence
  → Pres: ViewModels, UI, navigation

Phase 4: Integration (Staff)
  → DI wiring, lint, layer boundary audit
```

## Platform Differences

| | iOS Skills | Android Skills |
|-|-----------|---------------|
| **Language** | Swift 5.9+ | Kotlin 1.9+ |
| **UI** | SwiftUI | Jetpack Compose |
| **Architecture** | MVVM-C + Clean | MVVM + Clean |
| **DI** | Protocol-driven (manual) | Hilt (@Inject, @Module) |
| **Navigation** | Coordinators + NavigationPath | Navigation Compose |
| **State** | ViewState enums + @Published | UiState sealed + StateFlow |
| **Persistence** | CoreData / UserDefaults | Room / SharedPreferences |
| **Networking** | URLSession / Alamofire | Retrofit / Ktor |
| **DTOs** | Codable structs | @Serializable data classes |
| **Lint** | SwiftLint + SwiftFormat | ktlint + detekt |
| **Build** | xcodebuild | ./gradlew |
| **Deploy** | TestFlight / App Store | Play Console / Firebase |
| **Design** | iOS HIG | Material Design 3 |
| **Testing** | XCTest + XCUITest | JUnit5 + MockK + Espresso |
| **Domain Rule** | Zero UIKit/SwiftUI imports | Zero Android SDK imports |

## Evaluations

Each skill includes test scenarios in `evals/evals.json`:

**Full Pipeline evals** test: bootstrap setup, new feature routing, bug fix routing, sprint batch execution, pipeline memory learning.

**Lite Pipeline evals** test: quick feature (full flow), new app from idea, feedback loop recovery.

## Contributing

When modifying skills:

1. Keep SKILL.md under 500 lines — move detail to references/
2. Use imperative writing style ("Create the entity", not "Creating the entity")
3. Include concrete examples, not generic descriptions
4. Mark critical rules with **CRITICAL:** callouts
5. Use `references/` for anything over 20 lines of code or detailed explanation
6. Test with eval prompts before committing
7. Ensure no cross-platform contamination (no Swift in Android files)
