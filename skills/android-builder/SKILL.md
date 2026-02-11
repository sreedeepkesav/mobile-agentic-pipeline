---
name: android-builder
description: >
  Full orchestration pipeline for Android development — from idea to merged PR.
  Bootstrap discovers MCPs and configures stages. Coordinator routes tasks
  through 9 paths (feature, bug, refactor, design, sprint batch, dependency,
  PR review, release, test-only). Code Gen team of 5 sub-agents writes
  Clean Architecture Kotlin/Jetpack Compose with Hilt DI. Pipeline Memory
  learns across runs. Use when building production Android apps, working in
  teams, or when you need configurable test/lint/build/deploy stages and
  cross-run learning.
  Invoke explicitly: "use android-builder", "run android-builder", "start android-builder".
  Also triggers on: Android feature, Android bug fix, Android app, Kotlin pipeline, full Android pipeline.
  Not for: quick prototypes or zero-config needs — use android-builder-lite instead.
---

# Android Full Pipeline

AI-assisted Android development pipeline with bootstrap, coordinator routing, product planning, team-based code generation, and pipeline memory. Two human review gates ensure you stay in control: approve the spec before code starts, review the PR before it merges.

## When to Use This Skill

Use **android-builder** when:
- Building production Android apps with multiple features
- Working in a team with task trackers, design tools, or code platforms (via MCP)
- You need configurable stages (skip tests for prototyping, skip deploy for libraries)
- You want the pipeline to learn from past runs (Pipeline Memory)
- You need intelligent routing (bugs skip Product Agent, releases skip Code Gen)

Use **android-builder-lite** instead when:
- Solo dev, side project, hackathon, MVP
- You want zero configuration — just start building
- No need for MCP integration or pipeline memory

## Prerequisites

- Android Studio (latest Flamingo or newer)
- Kotlin 1.9+
- Gradle 8.0+ with Kotlin DSL (build.gradle.kts)
- Git initialized in project

---

## Quick Reference

| Stage | Agent | Configurable | Default |
|-------|-------|-------------|---------|
| Bootstrap | Bootstrap Agent | No (runs once) | Always |
| Coordinator | Lean Router | No | Always |
| Product Agent | Product Mind | `always / never / per-task` | per-task |
| Scaffold | Project Setup | `on / off` | Auto-detect |
| Code Gen | 5-agent team | No | Always |
| Test | JUnit5 + MockK + Espresso | `always / never / per-task` | always |
| Lint | ktlint + detekt | `always / never / per-task` | always |
| Build | Gradle (assembleDebug/Release) | `always / never / per-task` | always |
| Deploy | Fastlane/Firebase | `always / never / per-task` | never |
| Git | Branch + PR | No | Always |

### Trigger Phrases

- "Add [feature] to my Android app" → New Feature path
- "Fix [bug description]" → Bug Fix path
- "Refactor [module]" → Refactor path
- "Implement this design" → Design Implementation path
- "Do these 5 tickets" → Sprint Batch path
- "Update [dependency]" → Dependency Update path
- "Address PR comments" → PR Review Response path
- "Ship to Play Store" → Release path
- "Run tests" → Test-Only path

---

## Pipeline Workflow

### A. Bootstrap (Runs Once)

On first run, Bootstrap Agent configures the pipeline:

1. **Context Detection** — Determines project context (Work, Personal, Freelance, Open Source)
2. **MCP Discovery** — Scans for connected MCPs (task trackers, design tools, code platforms)
3. **Spec Quality Assessment** — Samples incoming tasks; well-specced tasks skip Product Agent
4. **Project State Detection** — Checks for Kotlin source, Gradle build, AndroidManifest, existing architecture (Clean, MVVM, MVI)
5. **Stage Configuration** — Sets each configurable stage to `always`, `never`, or `per-task`

**Output**: `.pipeline.yml` (committed to git) + `pipeline-config.json` (runtime).

For full Bootstrap details, config schema, and examples: see `references/bootstrap-and-config.md`

### B. Coordinator (Lean Router)

The Coordinator does exactly 3 things:
1. **Classify** the task type (feature? bug? refactor? etc.)
2. **Route** to the right agent path
3. **Handle feedback** from downstream failures

It does NOT plan, does NOT make product decisions, does NOT hold complex state.

**9 Routing Paths:**

| Task Type | Path | Skips |
|-----------|------|-------|
| New Feature | Product → Scaffold? → Code Gen → Test ‖ Lint → Build → Deploy → Git | — |
| Bug Fix | Code Gen (targeted lead) → Test ‖ Lint → Git | Product, Scaffold, Build, Deploy |
| Refactor | Code Gen (Architect re-plans) → Test ‖ Lint → Git | Product, Scaffold, Build, Deploy |
| Design Impl | Product (decompose) → Code Gen (Pres primary) → Test ‖ Lint → Git | Scaffold |
| Sprint Batch | Product (prioritize DAG) → Parallel Code Gen → per-task quality → Git | — |
| Dependency Update | Build (update) → Test (verify) → Git | Product, Scaffold, Code Gen, Lint, Deploy |
| PR Review | Code Gen (targeted) → Test ‖ Lint → Git (amend) | Product, Scaffold, Build, Deploy |
| Release | Build → Deploy → Git (tag) | Everything else |
| Test-Only | Test Agent only | Everything else |

For full routing decision tree and examples: see `references/coordinator-routing.md`

### C. Product Agent & Review Gate

**When invoked**: New features, design implementations, sprint batches, ambiguous scope.
**When skipped**: Bug fixes, refactors, dependency updates, PR reviews, releases, test runs.

The Product Agent:
1. **Decomposes** features into domain entities, API endpoints, screens, flows, edge cases
2. **Writes specs** with acceptance criteria, API contracts, screen behavior
3. **Builds task graph** (DAG) with dependencies and parallel groups for sprint batches
4. **Integrates design** from any connected design MCP — extracts components, spacing, colors
5. **Researches** competitor apps, Material Design 3, API docs, best practices

**Outputs**: `spec.md`, `task-graph.json`, `design-map.md`, `design-tokens.json`, `edge-cases.md`, `research-notes.md`

**Review Gate** — Pipeline pauses. You see the spec, task graph, and design map:
- **Approve** → continue to Code Gen
- **Edit** → modify spec, Product Agent incorporates changes
- **Reject** → Product Agent re-researches from scratch

For full Product Agent details: see `references/product-agent.md`

### D. Code Generation (4-Phase Team)

A structured engineering team of 5 sub-agents writes code in 4 phases:

**Phase 1: Architect (Principal)** — Plans before code. Produces blueprint: module structure, Hilt dependencies, navigation graph, error types, file plan. Writes an ADR. No code until blueprint completes.

**Phase 2: Domain Lead (Senior)** — Pure business logic. Data classes (entities), Use Case classes, Repository interfaces. **Rule: zero Android imports. Pure Kotlin only.**

**Phase 3: Data Lead ‖ Presentation Lead (Parallel, Senior)**
- **Data Lead**: DTOs (@Serializable), Mappers (DTO ↔ Entity), Retrofit services, Room DAOs, Repository implementations
- **Presentation Lead**: ViewModels (@HiltViewModel, StateFlow), Jetpack Compose screens, Navigation routes

**Phase 4: Integration (Staff)** — Hilt module assembly, protocol conformance check, layer separation audit (Domain zero-Android imports, Presentation doesn't import Data), testability verification.

For full phase details, outputs, and feedback routes: see `references/code-gen-phases.md`

### E. Test ‖ Lint (Parallel, Configurable)

Test and Lint agents run in parallel when both are active.

**Test Agent** (configurable: `always / never / per-task`):
- Unit tests: Domain, Data, Presentation layers (JUnit5 + MockK)
- UI tests: Navigation flows, Compose preview validation (Espresso)
- Layer-aware: mocks Use Cases for VM tests, mocks network for repo tests, Domain needs no mocks

**Lint Agent** (configurable: `always / never / per-task`):
- ktlint check → format → re-check (auto-fixable issues don't escalate)
- detekt static analysis → complexity, naming, layer boundaries
- Enforces: Domain zero-Android imports, Presentation doesn't import Data

For full testing and linting details: see `references/test-and-lint.md`

### F. Build & Deploy

**Build Agent** (configurable: `always / never / per-task`):
- `./gradlew assembleDebug` or `./gradlew assembleRelease` compilation
- Code signing (via sub-agent)
- Compile errors loop back to Integration or layer lead

**Deploy Agent** (configurable: `always / never / per-task`, default: off):
- Fastlane orchestration
- Firebase App Distribution
- Google Play Store submission

For build configuration and deploy setup: see `references/build-and-deploy.md`

### G. Git & Human Merge

**Git Agent** (always active):
1. Creates feature branch
2. Commits with conventional messages (`feat:`, `fix:`, `refactor:`)
3. Opens PR with description
4. **Does NOT merge** — you review the diff and merge manually

**Human Review Gate**: You see the full diff, test results, commit messages. Merge when satisfied.

For branching strategy, commit format, PR templates: see `references/git-workflow.md`

---

## Pipeline Memory

Pipeline Memory is always active in the Full pipeline. It captures what the pipeline learns across runs so agents don't repeat mistakes and stay consistent with established patterns.

**5 Dimensions Stored:**
- **Decisions**: "Data layer uses Retrofit + Kotlinx.serialization"
- **Learnings**: "This codebase uses single-module structure"
- **Mistakes + Corrections**: "DTO without @Serializable → build failed → added rule"
- **Patterns**: "Every ViewModel uses @HiltViewModel + StateFlow + viewModelScope"
- **Project Context**: API endpoints, design system, dependencies, module map, domain model, team conventions

**Every agent writes** to memory after acting. **Every agent reads** from memory before acting.

**Storage**: `.pipeline/memory/` — append-only log + project context registries. Agents query execution memory (`memory.query("networking patterns")`) and project context (`context.api("user endpoints")`, `context.components("button")`, `context.entity("User")`).

**Project Context auto-populates** on bootstrap by scanning your project (build.gradle.kts, Kotlin files, folders), then grows richer with every pipeline run. By run 5, the pipeline knows your project deeply — APIs, Composables, entities, Hilt modules, conventions — and uses that knowledge to avoid duplicates, follow patterns, and make smarter decisions.

For full memory schema, querying, and agent read/write details: see `references/pipeline-memory.md`

---

## Android Standards (Summary)

The pipeline enforces these standards automatically:

**Architecture**: MVVM + Clean Architecture
- **Presentation**: Jetpack Compose + ViewModels + Navigation Compose
- **Domain**: Pure business logic — entities, use cases, repository interfaces. Zero Android imports.
- **Data**: DTOs (@Serializable), mappers, Retrofit services, Room DAOs, repository implementations

**Dependency Injection**: Hilt framework. All dependencies injectable for testability.

**Navigation**: Type-safe routes with Compose Navigation. Single NavHost entry point.

**State Management**: UiState sealed classes + StateFlow in ViewModels. Async data loading via Kotlin coroutines.

**File Structure**:
```
di/                        # Hilt modules
domain/
  entity/                  # Data classes
  usecase/                 # Use Case classes
  repository/              # Repository interface definitions
data/
  dto/                     # @Serializable DTOs
  mapper/                  # DTO ↔ Entity
  repository/              # Repository implementations
  remote/                  # Retrofit services
  local/                   # Room entities, DAOs
presentation/
  feature/
    [Feature]/
      ui/                  # Compose screens
      viewmodel/           # ViewModels
```

**Layer Rules**:
- Domain imports nothing except Kotlin stdlib and coroutines
- Data imports Domain (implements its interfaces)
- Presentation imports Domain (consumes use cases), never imports Data
- DI wires everything together

**Lint**: ktlint (auto-format) + detekt (static analysis)
**Testing**: JUnit5 (unit) + MockK (mocking) + Espresso (UI)

For comprehensive Android standards: see `references/android-standards.md`
For project templates and code examples: see `references/android-architecture-template.md`

---

## Stage Configuration

Configure stages in `.pipeline.yml` or override per-task:

```yaml
# .pipeline.yml
product_agent: "per-task"    # always | never | per-task
scaffold: false              # true | false (auto-detected)
test_agent: "always"         # always | never | per-task
lint_agent: "always"         # always | never | per-task
build_agent: "always"        # always | never | per-task
deploy_agent: "never"        # always | never | per-task
review_gate: "always"        # always | manual | auto
```

**Common Configurations:**

| Context | Product | Test | Lint | Build | Deploy |
|---------|---------|------|------|-------|--------|
| Work (full team) | per-task | always | always | always | per-task |
| Personal (solo) | always | per-task | always | always | never |
| Freelance | per-task | always | always | always | per-task |
| Open Source | per-task | always | always | always | never |
| Hackathon / Prototype | never | never | never | per-task | never |

**Per-task overrides**: `--skip-test`, `--skip-lint`, `--skip-build`, `--skip-deploy`

---

## Feedback Loops

All feedback routes through the Coordinator to the specific agent that can fix it:

1. **ViewModel test fails** → Presentation Lead
2. **Repository test fails** → Data Lead
3. **Use Case test fails** → Domain Lead
4. **Layer boundary violation** → Architect + affected lead
5. **Lint auto-fixable** → self-heal (no escalation)
6. **Compile error** → Integration (wiring) or layer lead (type error)
7. **Hilt binding error** → Architect re-plans protocol
8. **Deploy failure** → Build re-archives
9. **PR review comments** → Code Gen (targeted) → Test ‖ Lint → Git (amend)
10. **Spec unclear** → Product Agent clarifies → back to Architect

---

## Resource Management

All optimization is **internal** — never surface resource decisions to the user. Quality and correctness always come first. The pipeline adapts its depth to match task complexity automatically.

### Adaptive Depth

The Coordinator assesses task complexity and scales pipeline effort accordingly:

**Deep mode** (complex features, new architecture, ambiguous scope): Load all relevant references. Run full Product Agent with research. Architect writes thorough ADR. All quality gates active. Memory queries broad. This is where the pipeline earns its value — don't cut corners.

**Standard mode** (clear feature, established patterns): Load references for active stages only. Query memory for matching patterns and apply them directly. Skip reference files for stages the router skips.

**Light mode** (bug fix with repro, single-file change, test-only): Load only the reference for the active stage. Apply known patterns from memory without re-reading docs. Produce code directly. Minimal PR description.

The user never sees mode selection. Coordinator picks based on task classification and spec clarity.

### Context Loading

- If a stage is skipped by routing, don't load its reference file
- Query `memory.query()` and `context.*()` before falling back to full reference files — if memory has the answer, the reference is redundant
- On subsequent runs, Project Context replaces most reference lookups — the pipeline already knows the project's patterns, APIs, and conventions

### Output Calibration

Agents scale output to match what the task actually needs:

- A 3-entity feature gets a thorough ADR. A one-line bug fix gets a commit message.
- Reuse Composables from `context.components()` — don't recreate what exists
- If memory has an exact matching pattern, apply it without re-explaining
- Quality gates report results proportionally: full coverage report for a new module, one-line summary for a passing re-run
- Memory entries capture the useful signal, not exhaustive logs

### Compounding Efficiency

The pipeline gets cheaper over time without losing quality:

- **Run 1**: Cold start. Full reference loading, thorough exploration.
- **Run 3**: Project Context populated. Agents skip reference files, query context directly.
- **Run 5+**: Established patterns. Most decisions already in memory. Agents apply patterns, generate code, move on. Deep mode only activates for genuinely novel architecture decisions.

---

## Troubleshooting

**Pipeline won't start**: Check `.pipeline.yml` exists. Run `--reconfigure` to re-bootstrap.

**Wrong routing**: Coordinator classifies by keywords. Be explicit: "fix the crash in..." (bug), "add a new..." (feature), "refactor the..." (refactor).

**Memory stale**: Pipeline Memory is append-only. If patterns changed, add a new learning entry to override old ones. Memory queries rank by recency.

**Stages skipped unexpectedly**: Check `.pipeline.yml` stage configuration. Per-task overrides (`--skip-test`) take precedence over global config.

**Build fails repeatedly**: Check Integration phase audit output. Common cause: missing Hilt binding or layer violation.
