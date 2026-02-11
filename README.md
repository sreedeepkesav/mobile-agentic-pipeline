# Mobile Agentic Pipeline

AI-assisted development pipeline for iOS and Android — from idea to committed code.

Two variants per platform:

- **Full Pipeline** — Bootstrap, Pipeline Memory, Coordinator, Product Agent, Code Gen team (5 sub-agents), Test, Lint, Build, Deploy, Git. Configurable stages, MCP-agnostic, learns across runs.
- **Lite Pipeline** — 4 stages (Product + Design → Coder → Build → Git). No config, no memory, just start building. For indie devs and newcomers to AI-assisted development.

## Quick Start

### Using as Claude Code Skills

Install a skill by copying its folder to your Claude Code skills directory:

```bash
# iOS Full Pipeline (production apps, teams, configurable stages)
cp -r skills/ios-full ~/.claude/skills/ios-full-pipeline

# iOS Lite Pipeline (side projects, MVPs, hackathons)
cp -r skills/ios-lite ~/.claude/skills/ios-lite-pipeline

# Android Full Pipeline (production apps, teams, configurable stages)
cp -r skills/android-full ~/.claude/skills/android-full-pipeline

# Android Lite Pipeline (side projects, MVPs, hackathons)
cp -r skills/android-lite ~/.claude/skills/android-lite-pipeline
```

Then use naturally in Claude Code:
```
"Add a user profile screen with avatar and settings" → triggers ios-full or android-full
"Build me a simple timer app" → triggers ios-lite or android-lite
```

### Which Skill Should I Use?

| Situation | Skill |
|-----------|-------|
| Production iOS app, team, need deploy | `ios-full-pipeline` |
| Side project, MVP, hackathon (iOS) | `ios-lite-pipeline` |
| Production Android app, team, need deploy | `android-full-pipeline` |
| Side project, MVP, hackathon (Android) | `android-lite-pipeline` |

**Full vs Lite decision**: Use Full if you need any of: configurable stages, pipeline memory (cross-run learning), MCP integration, sprint batch execution, 9-path routing. Use Lite if you want zero config and just want to start building.

## Architecture Diagrams

Open these in a browser to explore the full interactive architecture:

| Platform | Full | Lite |
|----------|------|------|
| **iOS** (Swift, SwiftUI, MVVM-C) | [View](architecture/ios/full-pipeline.html) | [View](architecture/ios/lite-pipeline.html) |
| **Android** (Kotlin, Compose, MVVM) | [View](architecture/android/full-pipeline.html) | [View](architecture/android/lite-pipeline.html) |

## Architecture at a Glance

### Full Pipeline Flow

```
Bootstrap → MCP Sources → Coordinator → Product Agent → ⏸ Review
→ Scaffold (if new) → Code Gen Team → Test ‖ Lint → Build
→ Deploy (if enabled) → Git → ⏸ Human Merge
```

### Lite Pipeline Flow

```
Your Idea → Product + Design → ⏸ Review → Coder (4-phase)
→ Build → Git → ⏸ Human Merge
```

### Code Gen Team (both variants)

```
Phase 1: Architect (Principal) — plans before code, writes ADRs
Phase 2: Domain Lead (Senior) — pure business logic, zero framework imports
Phase 3: Data Lead ‖ Presentation Lead (parallel, Senior) — implements Domain interfaces
Phase 4: Integration (Staff) — DI wiring, lint, layer boundary audit
```

## Skills Structure

Each skill is self-contained with progressive disclosure:

```
skills/
├── ios-full/                    # iOS Full Pipeline
│   ├── SKILL.md                 # Overview + workflow (<500 lines)
│   ├── references/              # Detailed guides (10 files)
│   └── evals/                   # Test cases
├── ios-lite/                    # iOS Lite Pipeline
│   ├── SKILL.md
│   ├── references/              # 6 files
│   └── evals/
├── android-full/                # Android Full Pipeline
│   ├── SKILL.md
│   ├── references/              # 10 files
│   └── evals/
└── android-lite/                # Android Lite Pipeline
    ├── SKILL.md
    ├── references/              # 6 files
    └── evals/
```

For detailed skill documentation, see [skills/README.md](skills/README.md).

## Key Principles

- **Two human review gates** — approve spec before code, review PR before merge
- **Standards built in** — Clean Architecture, protocol/Hilt DI, layer separation enforced automatically
- **MCP-agnostic** — works with any task tracker, design tool, or code platform via MCP
- **Pipeline Memory** (Full only) — learns from every run, avoids past mistakes
- **Configurable stages** (Full only) — Test, Lint, Build, Deploy can each be always/never/per-task

## Platform Standards

| Aspect | iOS | Android |
|--------|-----|---------|
| Language | Swift 5.9+ | Kotlin 1.9+ |
| UI | SwiftUI | Jetpack Compose |
| Architecture | MVVM-C + Clean Architecture | MVVM + Clean Architecture |
| DI | Protocol-driven (manual) | Hilt |
| Navigation | Coordinators (NavigationPath) | Navigation Compose |
| State | ViewState enums + @Published | UiState sealed classes + StateFlow |
| Lint | SwiftLint + SwiftFormat | ktlint + detekt |
| Build | xcodebuild | Gradle |
| Testing | XCTest + XCUITest | JUnit5 + MockK + Espresso |

## Roadmap

- [x] Architecture diagrams (iOS Full + Lite, Android Full + Lite)
- [x] Claude Code skills (iOS Full + Lite)
- [x] Claude Code skills (Android Full + Lite)
- [ ] Standalone CLI
- [ ] KMP shared module support

## License

MIT
