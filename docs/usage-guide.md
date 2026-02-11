# 📖 Usage Guide

Everything you can do with the Mobile Agentic Pipeline — end-to-end flows, single-agent runs, invocation methods, and configuration.

---

## 🚀 End-to-End Examples

### New Feature

```
💬  Use ios-builder to add a dark mode toggle in Settings.

What happens:
  1. Bootstrap checks config (or runs setup on first use)
  2. Coordinator classifies → "New Feature"
  3. Product Agent writes spec: screens, components, edge cases
  4. ⏸  You review the spec → approve / edit / reject
  5. Code Gen Team (4 phases):
     🏛️ Architect → 🧬 Domain → 📡 Data ‖ 🎨 Presentation → 🔧 Integration
  6. Tests + Lint (parallel)
  7. Build
  8. Git → branch + PR
  9. ⏸  You review → merge
```

### Bug Fix

```
💬  Use ios-builder — fix the crash when tapping Forgot Password offline.

What happens:
  1. Coordinator classifies → "Bug Fix"
  2. Skips Product Agent (scope is clear)
  3. Code Gen targets the affected layer only
  4. Test + Lint verify the fix
  5. Git → commit with "fix:" prefix → PR
```

### Sprint Batch

```
💬  Use android-builder for these 4 tickets:
     add dark mode, fix settings crash, notification prefs, refactor networking.

What happens:
  1. Product Agent builds a task dependency graph
  2. Independent tasks run in parallel (each gets its own Code Gen team)
  3. Dependent tasks wait for upstream
  4. Each task → its own branch and PR
```

### Quick Build (Lite)

```
💬  Use ios-builder-lite — recipe app with search and favorites.

What happens:
  1. Product+Design writes spec + design tokens
  2. ⏸  You approve
  3. Coder writes code (4-phase)
  4. Build → Git → PR
```

### From Scratch (Lite)

```
💬  Use android-builder-lite — Pomodoro timer with Material Design 3.

What happens:
  1. Spec + design tokens
  2. You approve
  3. Full project scaffolded (Hilt, Navigation Compose, etc.)
  4. Code → Build → PR
```

---

## 🧩 Single-Agent Scenarios

You don't have to run the entire pipeline. Ask for just the agent you need.

### 📋 Product Agent Only — Spec Without Code

```
💬  Use ios-builder — just run Product Agent to spec out a payment flow.
     Don't write any code yet.

Output:
  spec.md            — feature spec with acceptance criteria
  screens.md         — screen-by-screen breakdown
  design-tokens.json — colors, spacing, typography
  edge-cases.md      — error states, loading, offline
  task-graph.json    — task dependencies
```

Re-run Product Agent as many times as you want before committing to code.

### 🏛️ Architect Only — Plan Without Code

```
💬  Use android-builder — just run the Architect to plan the offline sync feature.
     Write the ADR, don't implement.

Output:
  docs/decisions/ADR-NNN.md — architecture decision record
  Blueprint: module structure, interface contracts, dependency graph
  File plan (what gets created and where)
```

### ✅ Test Only

```
💬  Use ios-builder — run tests only. Report coverage.

Runs unit tests (Domain, Data, Presentation) and UI tests.
Reports coverage and failures. No code changes.
```

### 🧹 Lint Only

```
💬  Use android-builder — just lint the codebase.

Runs ktlint + detekt. Auto-fixes what it can.
Reports unfixable violations. Checks layer boundaries.
```

### 📦 Build + Deploy Only

```
💬  Use ios-builder — build and ship to TestFlight.

Build → Deploy (Fastlane → TestFlight) → Git tag.
Skips Product Agent, Code Gen, Test, Lint.
```

### 🔀 Git Only

```
💬  Use ios-builder — create a PR for my current changes.

Branch → conventional commit → PR. You review and merge.
```

### 💻 Code Gen Only — Skip Product Agent

```
💬  Use android-builder — skip Product Agent, here's my spec:
     [paste]. Generate code, test, lint.

Code Gen (4-phase) → Test + Lint → Git PR.
```

---

## 🎯 How to Invoke

### Explicit (Recommended)

Name the skill in your prompt:

```
"Use ios-builder to ..."
"Run android-builder-lite — ..."
"Start ios-builder-lite: ..."
```

### Slash Commands

Set up command aliases for shorthand:

```
/ios-builder add dark mode to settings
/android-builder-lite build a counter app
```

### Make It the Default

**Install only one per platform.** If only `ios-builder` is installed, every iOS prompt uses it automatically.

**Project-level install** — put the skill in `.claude/skills/` inside the project root. Project-level overrides global.

```bash
cd your-project
mkdir -p .claude/skills
cp -r path/to/skills/ios-builder .claude/skills/ios-builder
```

**CLAUDE.md instruction** — add to your project's `CLAUDE.md`:

```markdown
Always use ios-builder for iOS development tasks.
Always use android-builder for Android development tasks.
```

---

## ⚖️ Builder vs Builder Lite

This is about your **process needs**, not the task's complexity.

| I need... | Use |
|-----------|-----|
| Configurable stages (skip tests, skip deploy) | **Builder** |
| Pipeline Memory (learn across runs) | **Builder** |
| MCP integration (Jira, Figma, GitHub) | **Builder** |
| Sprint batch (parallel tasks) | **Builder** |
| Smart routing (bugs vs features vs refactors) | **Builder** |
| Single-agent runs (just test, just lint) | **Builder** |
| Zero config | **Builder Lite** |
| Fastest path from idea to PR | **Builder Lite** |

---

## 🔧 Pipeline Stages (Builder)

| Stage | What It Does | Configurable |
|-------|-------------|:------------:|
| Bootstrap | Auto-detects project, discovers MCPs, sets config | Runs once |
| Coordinator | Classifies task type, routes to correct path | Always on |
| Product Agent | Specs features, builds task graphs, integrates design | `always` · `never` · `per-task` |
| Scaffold | Creates project structure (new projects only) | `on` · `off` |
| Code Gen | 5-agent team writes code in 4 phases | Always on |
| Test | Unit + UI tests, layer-aware | `always` · `never` · `per-task` |
| Lint | Format + static analysis + layer boundaries | `always` · `never` · `per-task` |
| Build | Compiles the project | `always` · `never` · `per-task` |
| Deploy | Ships to TestFlight / Play Store / Firebase | `always` · `never` · `per-task` |
| Git | Branch, commit, PR | Always on |

### 9 Routing Paths

The Coordinator automatically picks the right path based on your task:

| Task Type | What Runs | What's Skipped |
|-----------|-----------|----------------|
| 🆕 New Feature | Product → Scaffold? → Code Gen → Test ‖ Lint → Build → Deploy → Git | — |
| 🐛 Bug Fix | Code Gen (targeted) → Test ‖ Lint → Git | Product, Scaffold, Build, Deploy |
| ♻️ Refactor | Code Gen (Architect re-plans) → Test ‖ Lint → Git | Product, Scaffold, Build, Deploy |
| 🎨 Design Impl | Product → Code Gen (Pres primary) → Test ‖ Lint → Git | Scaffold |
| 📋 Sprint Batch | Product (DAG) → Parallel Code Gen → per-task quality → Git | — |
| 📦 Dependency Update | Build → Test → Git | Product, Scaffold, Code Gen, Lint, Deploy |
| 💬 PR Review | Code Gen (targeted) → Test ‖ Lint → Git (amend) | Product, Scaffold, Build, Deploy |
| 🚢 Release | Build → Deploy → Git (tag) | Everything else |
| 🧪 Test-Only | Test only | Everything else |

---

## 📱 Platform Standards

| | 🍎 iOS | 🤖 Android |
|-|--------|-----------|
| **Language** | Swift 5.9+ | Kotlin 1.9+ |
| **UI** | SwiftUI | Jetpack Compose |
| **Architecture** | MVVM-C + Clean | MVVM + Clean |
| **DI** | Protocol-driven (manual) | Hilt |
| **Navigation** | Coordinators (NavigationPath) | Navigation Compose |
| **State** | ViewState enums + @Published | UiState sealed + StateFlow |
| **Persistence** | CoreData / UserDefaults | Room / SharedPreferences |
| **Networking** | URLSession / Alamofire | Retrofit / Ktor |
| **Lint** | SwiftLint + SwiftFormat | ktlint + detekt |
| **Build** | xcodebuild | ./gradlew |
| **Deploy** | TestFlight / App Store | Play Console / Firebase |
| **Testing** | XCTest + XCUITest | JUnit5 + MockK + Espresso |
| **Domain Rule** | Zero UIKit/SwiftUI imports | Zero Android SDK imports |
