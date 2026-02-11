# Mobile Agentic Pipeline

AI-assisted development pipeline for iOS and Android — from idea to committed code.

Two variants per platform:

- **Full Pipeline** — Bootstrap, Pipeline Memory, Coordinator, Product Agent, Code Gen team (5 sub-agents), Test, Lint, Build, Deploy, Git. Configurable stages, MCP-agnostic, learns across runs.
- **Lite Pipeline** — 4 stages (Product + Design → Coder → Build → Git). No config, no memory, just start building. For indie devs and newcomers to AI-assisted development.

## Quick Start

### Install

Copy the skill(s) you want to your Claude Code skills directory:

```bash
# Global install (available in all projects)
cp -r skills/ios-full ~/.claude/skills/ios-full-pipeline
cp -r skills/ios-lite ~/.claude/skills/ios-lite-pipeline
cp -r skills/android-full ~/.claude/skills/android-full-pipeline
cp -r skills/android-lite ~/.claude/skills/android-lite-pipeline

# Project-level install (available only in this project)
mkdir -p .claude/skills
cp -r skills/ios-full .claude/skills/ios-full-pipeline
```

### Starting a Specific Pipeline

**Explicit invocation** (recommended) — name the pipeline you want:

```
"Use the ios-full-pipeline to add a user profile screen"
"Run ios-lite-pipeline — build me a timer app"
"Start android-full-pipeline for this sprint's tickets"
```

**Slash command** — if you set up a Claude Code command alias:

```
/ios-full Add dark mode toggle to settings
/android-lite Build a simple counter app
```

**Implicit trigger** — if only one skill is installed for a platform, any iOS/Android prompt triggers it automatically. If both Full and Lite are installed, Claude matches based on trigger keywords in the frontmatter description, but this can be ambiguous. Explicit invocation avoids mismatches.

### Making a Pipeline the Default

**Option A: Install only one per platform.** If you only install `ios-full-pipeline` (not lite), then every iOS prompt automatically uses the full pipeline. This is the simplest approach.

**Option B: Project-level install.** Put the skill in `.claude/skills/` inside the project root. Project-level skills override global ones — so a production repo can default to Full while your side-project repo defaults to Lite.

**Option C: CLAUDE.md instruction.** Add to your project's `CLAUDE.md`:
```
When I ask you to build iOS features, always use the ios-full-pipeline skill.
```

### Which Pipeline Do I Need?

Full vs Lite is about your **process needs**, not the task's complexity. A "simple timer app" might need the Full pipeline (if you want tests, deploy, memory). A "complex auth flow" might use Lite (if you're prototyping and just want code fast).

| I need... | Use |
|-----------|-----|
| Configurable stages (skip tests, skip deploy) | Full |
| Pipeline Memory (learn across runs) | Full |
| MCP integration (Jira, Figma, GitHub) | Full |
| Sprint batch execution (parallel tasks) | Full |
| Smart routing (bugs vs features vs refactors) | Full |
| Zero config — just start coding | Lite |
| No deploy, no memory, no bootstrap | Lite |
| Fastest path from idea to PR | Lite |

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
