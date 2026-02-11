# Mobile Agentic Pipeline

AI-assisted development pipeline for iOS and Android — from idea to committed code.

Two variants per platform:

- **Full Pipeline** — Bootstrap, Pipeline Memory, Coordinator, Product Agent, Code Gen team (5 sub-agents), Test, Lint, Build, Deploy, Git. Configurable stages, MCP-agnostic, learns across runs.
- **Lite Pipeline** — 4 stages (Product + Design → Coder → Build → Git). No config, no memory, just start building. For indie devs and newcomers to AI-assisted development.

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

## Key Principles

- **Two human review gates** — approve spec before code, review PR before merge
- **Standards built in** — Clean Architecture, protocol/Hilt DI, layer separation enforced automatically
- **MCP-agnostic** — works with any task tracker, design tool, or code platform via MCP
- **Pipeline Memory** (Full only) — learns from every run, avoids past mistakes
- **Configurable stages** (Full only) — Test, Lint, Build, Deploy can each be always/never/per-task

## Roadmap

- [ ] Claude Code skills (iOS Full + Lite)
- [ ] Claude Code skills (Android Full + Lite)
- [ ] Standalone CLI
- [ ] KMP shared module support

## License

MIT
