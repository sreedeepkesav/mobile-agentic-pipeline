---
name: android-lite-pipeline
description: >
  Lightweight Android development pipeline — from idea to PR in 4 stages.
  No bootstrap, no config files, no pipeline memory. Just describe what
  you want, approve the spec, and get clean Kotlin/Jetpack Compose code
  following MVVM + Clean Architecture with Hilt DI. Perfect for side
  projects, learning, MVPs, and hackathons. Triggers: quick Android app,
  simple Android feature, lite Android pipeline, indie Android dev,
  Android prototype, Kotlin app.
  Not for: production apps needing configurable stages or cross-run
  learning — use android-full-pipeline instead.
---

# Android Lite Pipeline

Lightweight, zero-configuration Android development pipeline. Describe your idea, approve the design, and get production-ready Kotlin/Jetpack Compose code following MVVM + Clean Architecture with Hilt DI.

## When to Use This Skill

Use **android-lite-pipeline** when:
- Solo dev, side project, hackathon, MVP
- You want zero configuration — just start building
- No need for MCP integration or pipeline memory

Use **android-full-pipeline** instead if you need configurable stages, cross-run learning, or production deploy.

## Prerequisites

- Android Studio (latest Flamingo or newer)
- Kotlin 1.9+
- Gradle 8.0+ with Kotlin DSL (build.gradle.kts)
- Git initialized in project

---

## Quick Start

Your role: Describe your idea → Approve the design → Review the PR

AI's role: Design, code, compile, commit, open PR

```
Your Idea
    ↓
Product + Design (research, plan, design)
    ↓
⏸ Review Gate 1: Approve Spec?
    ↓
Coder (4-phase: Architect → Domain → Data ∥ Pres → Integration)
    ↓
Build (./gradlew assembleDebug compile)
    ↓
Git (create branch, conventional commits, open PR)
    ↓
⏸ Review Gate 2: Manual Merge
```

---

## Pipeline Workflow

### A. Product + Design Agent

**Inputs**: Your idea (text description or feature request)

**Tasks**:
1. **Research**: Material Design 3 guidelines, similar apps on Google Play, Jetpack Compose best practices, Android architecture patterns
2. **Planning**: Feature spec with acceptance criteria, screen list, navigation graph, API contracts, data models
3. **Design**: Material Design 3 color scheme, spacing/typography tokens, component layouts, navigation flow

**Outputs**:
- `spec.md` — Feature spec + acceptance criteria
- `screens.md` — Screen list, wireframes, navigation graph
- `api-contracts.md` — Endpoints, payloads, error handling
- `data-models.md` — Domain entities, relationships
- `design-tokens.json` — Colors, spacing, typography
- `edge-cases.md` — Error scenarios, validation, empty states

For full Product + Design details: see `references/product-design-flow.md`

---

### B. Review Gate 1: Approve the Spec

You see the spec, screens, design tokens, and edge cases.

**Options**:
- **Approve**: Start coding immediately
- **Edit**: Request scope changes, design adjustments (Product+Design revises, gate repeats)
- **Reject**: Abandon this spec, start from scratch

---

### C. Coder — 4-Phase Code Generation

A team of AI specialists writes code in 4 sequential phases. Phase 3 has two parallel lanes (Data and Presentation).

#### Phase 1: Architect (Principal)
- Plans before code: module structure, Hilt setup, navigation graph, error handling strategy, testing plan
- Writes Architecture Decision Record (ADR) with module diagram
- No code until blueprint completes

#### Phase 2: Domain Lead (Senior)
- Pure business logic (data classes for entities, Use Case classes, Repository interfaces)
- Zero Android imports (Kotlin stdlib + coroutines only)

#### Phase 3a & 3b: Data Lead ‖ Presentation Lead (Parallel, Senior)
- **Data Lead**: DTOs (@Serializable), Mappers (DTO ↔ Entity), Retrofit services, Room DAOs, Repository implementations
- **Presentation Lead**: ViewModels (@HiltViewModel, StateFlow, UiState sealed classes), Jetpack Compose screens, Navigation Compose routes

#### Phase 4: Integration (Staff)
- Assembles Hilt modules (@Module, @Provides, @Binds)
- Configures NavHost with all routes
- Layer audits (Domain zero-Android imports, Presentation doesn't reference Data)
- Runs ktlint and detekt, auto-fixes what it can

For full 4-phase details: see `references/code-gen-phases.md`

---

### D. Build

**Command**: `./gradlew assembleDebug`

**If compile fails**: Loop back to Integration phase, fix issue, retry (max 2 times).

---

### E. Git & Human Merge

**AI's role**:
1. Create feature branch (`feature/[feature-name]`)
2. Commit with conventional messages (`feat:`, `fix:`, `refactor:`)
3. Open PR with description
4. Does NOT merge — waits for you

**Your role**: Review diff, commit messages, file structure. Manual merge when satisfied.

For branching strategy, commit format, PR templates: see `references/build-and-git.md`

---

## Android Standards Summary

### Architecture: MVVM + Clean Architecture

**Three Layers**:
1. **Domain** (Pure Kotlin business logic)
   - Data classes (entities)
   - Use Case classes with @Inject
   - Repository interfaces
   - Exception types

2. **Data** (Persistence & networking)
   - DTOs (@Serializable for Kotlinx.serialization)
   - Mappers (DTO ↔ Entity)
   - Retrofit services
   - Room entities, DAOs, Database
   - Repository implementations

3. **Presentation** (UI & interaction)
   - ViewModels (@HiltViewModel, StateFlow, UiState sealed classes)
   - Jetpack Compose screens
   - Navigation Compose routes

### Dependency Injection: Hilt
- **@HiltAndroidApp**: Application class
- **@AndroidEntryPoint**: Activities, Fragments
- **@HiltViewModel**: ViewModel with @Inject constructor
- **@Module + @InstallIn**: DI bindings
- **@Binds / @Provides**: interface → implementation

### State Management: UiState + StateFlow
```kotlin
sealed class LoginUiState { object Loading : LoginUiState() ... }

@HiltViewModel
class LoginViewModel @Inject constructor(...) : ViewModel() {
    private val _uiState = MutableStateFlow<LoginUiState>(LoginUiState.Idle)
    val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()
}
```

### File Structure
```
di/                           # Hilt modules
domain/entity/, usecase/, repository/
data/dto/, mapper/, remote/, local/, repository/
presentation/feature/[Feature]/ui/, viewmodel/
navigation/AppNavHost.kt
MainActivity.kt
```

### Layer Rules
- **Domain**: Zero Android imports. Pure Kotlin stdlib + coroutines only.
- **Data**: Can import Domain + Android (Room, Retrofit). Cannot import Presentation.
- **Presentation**: Can import Domain. Cannot import Data directly (use Domain interfaces only).
- **DI**: Wires all layers together.

### Code Quality
- **ktlint**: Kotlin style formatting (auto-fix with `./gradlew ktlintFormat`)
- **detekt**: Static analysis (complexity, naming, layer boundaries)
- **JUnit5 + MockK**: Unit tests for domain and data layers
- **Compose Preview**: @Preview for UI testing

For comprehensive Android standards: see `references/android-standards.md`
For project templates and code examples: see `references/android-architecture-template.md`

---

## Feedback Loops

### Loop 1: Build Failure Recovery
**Trigger**: `./gradlew assembleDebug` fails

**Flow**: Log error → route to Integration phase → fix root cause → retry (max 2 times)

### Loop 2: Lint Violation Auto-Fix
**Trigger**: ktlint finds formatting violations

**Flow**: Run `./gradlew ktlintFormat` → re-verify → continue or manual review

### Loop 3: Layer Boundary Violation
**Trigger**: Architect detects Domain importing Android, or Presentation importing Data

**Flow**: Architect clarifies layers → affected layer re-implements imports → Integration audits & fixes

### Loop 4: Spec Ambiguity During Coding
**Trigger**: Coder encounters unclear acceptance criteria or missing requirement

**Flow**: Pause → route to Product + Design for clarification → resume coding

### Loop 5: PR Feedback Comments
**Trigger**: Human review comments on PR

**Flow**: Coder implements targeted fix → Build → Git amends commit → you re-review

For full feedback loop details: see `references/feedback-loops.md`

---

## When to Upgrade to android-full-pipeline

Consider **android-full-pipeline** if you need:
- Multi-environment deployment (dev, staging, production)
- Cross-run learning (pipeline remembers architecture decisions)
- Configurable stages (enable/disable phases per task)
- Feature flags and A/B testing infrastructure
- Production monitoring (crash reporting, analytics)
- CI/CD integration (GitHub Actions, etc.)
- Team coordination (multiple developers, code owners)

Until then, Lite is ideal for solo devs, learning projects, MVPs, hackathons, and starting new apps.

---

## Troubleshooting

**Gradle Sync Fails**: Run `./gradlew --refresh-dependencies` and resync in Android Studio.

**Hilt Compilation Errors**: Check Hilt modules in Phase 4. Ensure all dependencies are provided or injected.

**Compose Preview Crashes**: Create preview-friendly parameter overloads. Use `hiltViewModel()` only at screen entry point.

**Navigation Graph Not Found**: Ensure all screens are added to NavHost composable block in Phase 4 Integration.

**ktlint or detekt Failures**: Run auto-fix (`./gradlew ktlintFormat`) or manually resolve detekt warnings.

---

## See Also

- `references/product-design-flow.md` — Product + Design phase details
- `references/code-gen-phases.md` — 4-phase Coder details
- `references/android-standards.md` — Kotlin/Jetpack Compose standards
- `references/android-architecture-template.md` — Code templates
- `references/build-and-git.md` — Build & Git details
- `references/feedback-loops.md` — Feedback loop details
