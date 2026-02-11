<h1 align="center">
  🚀 Mobile Agentic Pipeline
</h1>

<p align="center">
  <strong>Describe a feature. Get a PR.</strong><br/>
  AI-powered iOS & Android development — from idea to merge, with you in the loop.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/iOS-Swift_%7C_SwiftUI-F05138?style=for-the-badge&logo=swift&logoColor=white" alt="iOS">
  <img src="https://img.shields.io/badge/Android-Kotlin_%7C_Compose-7F52FF?style=for-the-badge&logo=kotlin&logoColor=white" alt="Android">
  <img src="https://img.shields.io/badge/Powered_by-Claude_Code-D97706?style=for-the-badge&logo=anthropic&logoColor=white" alt="Claude Code">
  <img src="https://img.shields.io/badge/License-MIT-22C55E?style=for-the-badge" alt="MIT License">
</p>

<p align="center">
  <a href="docs/usage-guide.md"><strong>📖 Usage Guide</strong></a> &nbsp;·&nbsp;
  <a href="docs/battle-testing.md"><strong>🧪 Battle Testing</strong></a> &nbsp;·&nbsp;
  <a href="#-architecture"><strong>🏗️ Architecture</strong></a>
</p>

---

## ✨ What This Does

You tell Claude Code what you want. A team of **5 AI agents** handles the rest.

```
You:  "Use ios-builder to add a dark mode toggle in Settings."

  📋  Spec — screens, edge cases, API contracts
  ⏸   You approve
  🧠  5-agent Code Gen team → 4 phases of Clean Architecture code
  ✅  Tests + Lint + Build
  🔀  PR with conventional commits
  ⏸   You review and merge
```

**Two human checkpoints. Zero surprises. Production-grade code.**

---

## ⚡ Quick Start

**1.** Clone and install into your project:

```bash
git clone https://github.com/sreedeepkesav/mobile-agentic-pipeline.git
cd your-ios-or-android-project
mkdir -p .claude/skills

# Pick one:
cp -r path/to/mobile-agentic-pipeline/skills/ios-builder .claude/skills/ios-builder
cp -r path/to/mobile-agentic-pipeline/skills/android-builder .claude/skills/android-builder
```

> 💡 **Install per-project.** The pipeline builds a **Project Context** — it learns your APIs, design system, dependencies, and domain model. This compounds across runs. By run 5, it knows your project deeply. Global install (`~/.claude/skills/`) works but won't retain project-specific learning.

**2.** Use it:

```
Use ios-builder to add user authentication with email and password.
```

That's it. The pipeline takes it from here.

---

## 🧩 You Don't Have to Run Everything

Use the full pipeline, or just the parts you need:

```
💬  "Use ios-builder — just spec out the payment flow, no code yet."
💬  "Use android-builder — run tests only, report coverage."
💬  "Use ios-builder — build and ship to TestFlight."
💬  "Use android-builder — just lint the codebase."
💬  "Use ios-builder — create a PR for my current changes."
```

Mix and match. **[See all scenarios →](docs/usage-guide.md)**

---

## 📦 4 Variants

<table>
  <tr>
    <td></td>
    <td align="center"><strong>🍎 iOS</strong></td>
    <td align="center"><strong>🤖 Android</strong></td>
  </tr>
  <tr>
    <td><strong>Builder</strong></td>
    <td align="center"><code>ios-builder</code></td>
    <td align="center"><code>android-builder</code></td>
  </tr>
  <tr>
    <td><strong>Builder Lite</strong></td>
    <td align="center"><code>ios-builder-lite</code></td>
    <td align="center"><code>android-builder-lite</code></td>
  </tr>
</table>

### 🏗️ Builder (Full)

For projects that need structure. Bootstrap auto-configures your environment. Coordinator intelligently routes tasks — bugs skip the spec phase, releases skip code gen, sprint batches run in parallel. **Pipeline Memory** learns your patterns, mistakes, and conventions. Configurable stages let you skip tests, skip deploy, or run just one agent.

### ⚡ Builder Lite

For speed. No config, no memory, no bootstrap. Describe what you want, approve the spec, get a PR. Same 5-agent Code Gen team, same Clean Architecture output — just a simpler process.

> **How to choose**: It's about your *process*, not the task's complexity. A timer app might use Builder (you want tests + deploy + memory). A complex feature might use Lite (you're prototyping fast).

---

## 🧠 The Pipeline Learns Your Project

Builder variants build a **Project Context** that grows automatically:

```
🔵 Run 1   Cold start. Full exploration.
🟡 Run 3   Knows your APIs, components, and patterns.
🟢 Run 5+  Applies established patterns instantly. Reuses your components.
            Follows your naming conventions. Avoids past mistakes.
```

**What it tracks**: API endpoints & auth patterns · design system components · dependencies & versions · module structure · team conventions · domain entities & business rules.

This isn't configuration — it's automatic. The pipeline discovers your project as it works on it.

---

<a name="-architecture"></a>
## 🏗️ Architecture

**Interactive diagrams** — click to explore the full pipeline flow:

<table>
  <tr>
    <td></td>
    <td align="center"><strong>Builder</strong></td>
    <td align="center"><strong>Builder Lite</strong></td>
  </tr>
  <tr>
    <td>🍎 <strong>iOS</strong> — Swift · SwiftUI · MVVM-C</td>
    <td align="center"><a href="https://sreedeepkesav.github.io/mobile-agentic-pipeline/ios/full-pipeline.html">🔗 View</a></td>
    <td align="center"><a href="https://sreedeepkesav.github.io/mobile-agentic-pipeline/ios/lite-pipeline.html">🔗 View</a></td>
  </tr>
  <tr>
    <td>🤖 <strong>Android</strong> — Kotlin · Compose · MVVM</td>
    <td align="center"><a href="https://sreedeepkesav.github.io/mobile-agentic-pipeline/android/full-pipeline.html">🔗 View</a></td>
    <td align="center"><a href="https://sreedeepkesav.github.io/mobile-agentic-pipeline/android/lite-pipeline.html">🔗 View</a></td>
  </tr>
</table>

### Code Gen Team — 5 Agents, 4 Phases

```
Phase 1  🏛️  Architect (Principal)     Blueprint, ADR, file plan — no code yet
Phase 2  🧬  Domain Lead (Senior)      Entities, use cases — pure business logic
Phase 3  📡  Data Lead ‖ 🎨 Pres Lead  Parallel: API/DB layer ↔ UI/ViewModel layer
Phase 4  🔧  Integration (Staff)       DI wiring, layer audit, conformance check
```

**Hard rule**: Domain layer has zero framework imports. Pure Swift / pure Kotlin only.

### Platform Standards

| | 🍎 iOS | 🤖 Android |
|-|--------|-----------|
| **UI** | SwiftUI | Jetpack Compose |
| **Architecture** | MVVM-C + Clean | MVVM + Clean |
| **DI** | Protocol-driven (manual) | Hilt |
| **Navigation** | Coordinators + NavigationPath | Navigation Compose |
| **State** | @Published + ViewState enums | StateFlow + UiState sealed |
| **Testing** | XCTest + XCUITest | JUnit5 + MockK + Espresso |
| **Lint** | SwiftLint + SwiftFormat | ktlint + detekt |

---

## 🗺️ Roadmap

- [x] 📐 Interactive architecture diagrams
- [x] 🛠️ Claude Code skills — iOS + Android, Builder + Lite
- [x] 🧠 Project Context — auto-learning project brain
- [ ] 🔄 KMP shared module support

---

<p align="center">
  <strong>MIT License</strong> · Built with ❤️ and Claude Code
</p>
