# Battle-Testing Guide for Mobile Agentic Pipeline

This guide helps you verify that the four mobile skills (ios-builder, ios-builder-lite, android-builder, android-builder-lite) work correctly in real-world scenarios.

---

## 1. Setup

### Prerequisites

- Claude Code installed and configured (`~/.claude/config.json`)
- **iOS:** Xcode 15+ with Command Line Tools
- **Android:** Android Studio 2024+ with Kotlin compiler
- **Verify:** `which xcodebuild` and `which gradlew`

### Installing a Skill

Copy the skill directory to your global Claude Code skills location:

```bash
cp -r skills/ios-builder ~/.claude/skills/ios-builder
cp -r skills/ios-builder-lite ~/.claude/skills/ios-builder-lite
cp -r skills/android-builder ~/.claude/skills/android-builder
cp -r skills/android-builder-lite ~/.claude/skills/android-builder-lite
```

Verify installation:
```bash
ls ~/.claude/skills/
# Should list: ios-builder, ios-builder-lite, android-builder, android-builder-lite
```

### Verify Claude Code Recognizes Skills

In Claude Code, ask:

```
What skills do you have?
```

Expected response: Lists all four mobile skills with brief descriptions.

---

## 2. Test Matrix

### iOS Builder (Full)

| Priority | Scenario | Prompt to Use | What to Verify |
|----------|----------|---------------|-----------------|
| 1 | Bootstrap on new project | "Use ios-builder to set up a new Notes app." | `.pipeline.yml` created; context questions asked (team size, deployment target, privacy needs) |
| 1 | New feature end-to-end | "Use ios-builder to add dark mode toggle in Settings." | All stages fire: spec → review ⏸ → code gen (4 phases) → test → lint → build → PR |
| 2 | Bug fix routing | "Use ios-builder — fix crash when tapping Forgot Password offline." | Product Agent skipped; Code Gen targets only affected layer; conventional commit "fix:" prefix |
| 2 | Single-agent: Product only | "Use ios-builder — just spec out a payments flow. Don't code." | `spec.md`, `screens.md`, `design-tokens.json`, `edge-cases.md` created; no Swift files generated |
| 3 | Single-agent: Test only | "Use ios-builder — run tests only. Report coverage." | Unit tests and UI tests execute; coverage report generated; no code changes |
| 3 | Pipeline Memory | "Use ios-builder — add auth layer." | Memory files exist in `.pipeline/memory/`; previous decisions referenced |

### iOS Builder Lite

| Priority | Scenario | Prompt to Use | What to Verify |
|----------|----------|---------------|-----------------|
| 1 | New app from scratch | "Use ios-builder-lite — Recipe app with search and favorites." | Full lite flow: spec + design tokens → review ⏸ → 4-phase code gen → build → PR; completes in <5 min |
| 2 | Feature addition | "Use ios-builder-lite — add sharing feature to the recipe app." | Generated code follows existing patterns (MVVM-C, Coordinator); no structural changes |
| 3 | Feedback loop: build fails | "Build failed with error X. Retry the code gen." | Pipeline restarts code gen phase only; preserves spec and design tokens |
| 3 | Review gate rejection | "Reject the spec — we need offline sync too." | Pipeline resets; Product Agent re-runs; no code generated until approved |

### Android Builder (Full)

| Priority | Scenario | Prompt to Use | What to Verify |
|----------|----------|---------------|-----------------|
| 1 | Bootstrap on Kotlin project | "Use android-builder to add a feature to our existing Todo app." | Architecture detected (MVVM, Clean layers); `.pipeline.yml` reflects project structure |
| 1 | New feature with DI | "Use android-builder to add notification settings with Hilt DI and Compose UI." | Code uses Hilt for DI; Compose for UI; Clean Architecture (Domain → Data → Presentation) visible |
| 2 | Sprint batch | "Use android-builder — process these 4 tasks: dark mode, settings crash, notification prefs, refactor API layer." | Parallel task execution; each task gets its own Code Gen team; separate PRs generated |

### Android Builder Lite

| Priority | Scenario | Prompt to Use | What to Verify |
|----------|----------|---------------|-----------------|
| 1 | New app from scratch | "Use android-builder-lite — Pomodoro timer with Material Design 3." | Full lite flow: spec + design tokens → review ⏸ → 4-phase code → build → PR |
| 2 | Feature addition | "Use android-builder-lite — add a stats dashboard to the Pomodoro app." | Generated code respects Material Design 3; Navigation Compose used; no framework violations |

---

## 3. What to Look For (Pass/Fail Criteria)

### Code Generation

**Pass:**
- 4 phases visible in logs: `Architect` → `Domain Lead` → `Data ‖ Presentation` → `Integration`
- Files generated in correct directories (Domain → Data → Presentation for Android; Model → Repository → View for iOS)
- No placeholder comments like "TODO: implement later"

**Fail:**
- Flat structure (all files in root or single folder)
- Missing phases (only 2-3 phases executed)
- Placeholder-heavy code

### Layer Boundaries (Android Critical)

**Pass:**
- `Domain/` contains only Kotlin classes, interfaces, use cases — **zero imports from `android.*`, `androidx.*`, or `com.google`**
- `Data/` imports Domain types and Android APIs
- `Presentation/` imports Domain and Compose

**Fail:**
- Domain imports Android SDK (e.g., `import android.content.Context`)
- Data layer tries to use Compose
- Presentation logic in Domain

### Layer Boundaries (iOS Critical)

**Pass:**
- Model (Domain) contains only Swift types and Foundation imports
- Repository (Data) imports Model types and URLSession/CoreData
- View (Presentation) imports Repository and SwiftUI

**Fail:**
- Model imports SwiftUI or UIKit
- Networking logic in Views
- Coordinator decisions in Models

### Review Gates

**Pass:**
- Pipeline pauses with "⏸ Review spec — approve / edit / reject"
- When you respond "reject", pipeline resets (no code generated)
- When you "approve", pipeline continues to next stage

**Fail:**
- Pipeline skips review even on first run
- Rejection doesn't reset state
- Approval triggers code gen twice

### Git Output

**Pass:**
- Conventional commits: "feat:", "fix:", "refactor:", "test:", "docs:"
- Branch name: `feature/dark-mode` or `fix/offline-crash`
- PR opened with title and description matching commit message
- No merge commits or force pushes

**Fail:**
- Generic commit messages ("update code", "WIP")
- Branch names like `master-changes` or `feature1`
- PR never created
- Commits to main/master directly

### Architecture

**Pass (iOS):**
- MVVM-C pattern: Models → ViewModels + Coordinators → Views
- Coordinator routes navigation using `NavigationPath`
- ViewState enum for each screen state
- `@Published` for reactive bindings

**Pass (Android):**
- MVVM + Clean Architecture: Domain (use cases) → Data (repositories) → Presentation (ViewModels)
- Hilt `@Module` / `@Provides` for DI
- `UiState` sealed class or data class for state
- `StateFlow` or `LiveData` for reactive bindings

**Fail:**
- Missed architecture layers
- Hardcoded dependencies
- No state management pattern

### Build

**Pass:**
- `xcodebuild` completes successfully (iOS); see "Build complete!" or similar
- `./gradlew build` completes successfully (Android); see "BUILD SUCCESSFUL"
- No unresolved symbol errors

**Fail:**
- Build hangs or errors
- Syntax errors in generated code
- Missing dependencies in manifest/podfile

---

## 4. Common Issues & Fixes

| Issue | Diagnosis | Fix |
|-------|-----------|-----|
| Skill not found | Claude says "I don't know ios-builder" | Verify `~/.claude/skills/ios-builder` exists; restart Claude Code |
| Claude ignores skill | Runs without skill invocation | Use explicit: "Use ios-builder to ..." instead of "Add dark mode" |
| Stages skipped | Missing test/lint/build logs | Check `.pipeline.yml` stage config; verify skill installed to global `~/.claude/skills/` |
| Code gen produces flat | All files in root, no module structure | Verify architecture template loaded; check logs for "Architect phase" messages |
| Layer violations (Android) | Domain imports `android.*` | Check generated files; if present, request: "Regenerate Domain layer without Android imports" |
| Build fails on generated code | Syntax or missing import errors | Run: "Use android-builder — lint only" to auto-fix; then rebuild |
| Review gate never pauses | Pipeline skips approval | Check `.pipeline.yml` has `review_gates: [spec, build]`; re-run with explicit approval prompt |
| PR not created | Code generated, Git says "done", no PR | Verify GitHub token in Claude Code config; check logs for "Opening PR..." line |

---

## 5. Reporting Issues

File GitHub issues in `mobile-agentic-pipeline` with this template:

```markdown
## Battle Test Issue

**Skill:** [ios-builder | ios-builder-lite | android-builder | android-builder-lite]

**Prompt Used:**
[Exact prompt you gave Claude Code]

**Claude Code Version:**
[Output of: `claude --version`]

**Expected Behavior:**
[What should happen according to this guide]

**Actual Behavior:**
[What actually happened]

**Logs/Evidence:**
[Paste .pipeline/logs/latest.log or relevant console output]

**Reproducible:**
[Always / Sometimes / Once]

**Environment:**
- Platform: [macOS 13+ / Windows 11 / Linux]
- Xcode version (iOS): [e.g., 15.0]
- Android Studio version: [e.g., 2024.1.1]
```

### Example Issue

```markdown
## Battle Test Issue

**Skill:** ios-builder

**Prompt Used:**
"Use ios-builder to add dark mode toggle in Settings."

**Claude Code Version:**
claude 1.4.2

**Expected Behavior:**
All stages fire (spec → code gen → test → build → PR). Domain layer has zero UIKit imports.

**Actual Behavior:**
Pipeline skipped Test stage. Domain layer imports SwiftUI for Color constants.

**Logs/Evidence:**
[...stages log showing Test phase skipped...]

**Reproducible:**
Always

**Environment:**
- Platform: macOS 14.2
- Xcode: 15.1
```

---

## Quick Reference

**Fastest way to validate a skill:**

```bash
# 1. Install
cp -r skills/ios-builder ~/.claude/skills/ios-builder

# 2. Test
# In Claude Code, run:
# "Use ios-builder to add a simple counter screen."

# 3. Verify
# ✓ spec.md, code generated, tests run, PR opened
# ✓ All files in correct directories (Model → Repository → View)
# ✓ Conventional commits with "feat:" prefix
# ✓ Build succeeds

# Done!
```

**Single-agent quick test (no full pipeline):**

```
"Use ios-builder — just run Product Agent to spec a payments feature."
# Verify: spec.md created, no Swift files
```

**Layer boundary quick test:**

```
"Use android-builder to add authentication."
# Verify: grep -r "import android" skills/android-builder/generated/Domain/
# Should return empty (no Android SDK in Domain)
```
