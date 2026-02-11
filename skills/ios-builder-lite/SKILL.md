---
name: ios-builder-lite
description: >
  Lightweight iOS development pipeline — from idea to PR in 4 stages.
  No bootstrap, no config files, no pipeline memory. Just describe what
  you want, approve the spec, and get clean Swift/SwiftUI code following
  MVVM-C + Clean Architecture. Perfect for side projects, learning,
  MVPs, and hackathons.
  Invoke explicitly: "use ios-builder-lite", "run ios-builder-lite", "start ios-builder-lite".
  Also triggers on: quick iOS app, simple iOS feature, lite iOS pipeline, indie iOS dev, iOS prototype.
  Not for: production apps needing configurable stages or cross-run learning —
  use ios-builder instead.
---

# iOS Lite Pipeline

Lightweight, zero-configuration iOS development pipeline. Describe your idea, approve the design, and get production-ready Swift/SwiftUI code following MVVM-C + Clean Architecture.

## When to Use This Skill

Use **ios-builder-lite** when:
- Solo dev, side project, hackathon, MVP
- You want zero configuration — just start building
- No need for MCP integration or pipeline memory

Use **ios-builder** instead if you need configurable stages, cross-run learning, or production deploy.

## Prerequisites

- Xcode (latest stable)
- Swift 5.9+
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
Build (xcodebuild compile)
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
1. **Research**: iOS HIG, SwiftUI patterns, similar apps, API conventions
2. **Planning**: Feature spec with acceptance criteria, screen list, navigation flows, API contracts, data models
3. **Design**: UI wireframes, component layouts, color palettes, spacing tokens

**Outputs**:
- `spec.md` — Feature spec + acceptance criteria
- `screens.md` — Screen list + navigation flows + component breakdown
- `design-tokens.json` — Colors, typography, spacing
- `edge-cases.md` — Loading, empty, error, offline states

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
- Plans before code: protocol contracts, dependency graph, file plan, error types, coordinator navigation
- Writes Architecture Decision Record (ADR)
- No code until blueprint completes

#### Phase 2: Domain Lead (Senior)
- Pure business logic (pure struct entities, Use Case protocols, Repository protocols)
- Zero framework imports (Foundation only)

#### Phase 3a & 3b: Data Lead ‖ Presentation Lead (Parallel, Senior)
- **Data Lead**: DTOs (Codable), Mappers (DTO ↔ Entity), Repository implementations, APIClient, persistence
- **Presentation Lead**: ViewModels (@Published + async load), SwiftUI Views, Coordinators (MVVM-C)

#### Phase 4: Integration (Staff)
- Assembles DI container, verifies protocol conformance
- Layer separation audit (Domain has zero framework imports, Presentation doesn't import Data)
- Runs SwiftLint and SwiftFormat, fixes what it can

For full 4-phase details: see `references/code-gen-phases.md`

---

### D. Build

**Command**: `xcodebuild build`

**If compile fails**: Loop back to Integration phase, fix issue, retry.

---

### E. Git & Human Merge

**AI's role**:
1. Create feature branch
2. Commit with conventional messages (`feat:`, `fix:`, `refactor:`)
3. Open PR with description
4. Does NOT merge — waits for you

**Your role**: Review diff, commit messages, file structure. Manual merge when satisfied.

For branching strategy, commit format, PR templates: see `references/build-and-git.md`

---

## iOS Standards Summary

### Architecture: MVVM-C + Clean Architecture

**Three Layers**:
1. **Domain** (Pure business logic)
   - Entities (pure structs, no framework imports)
   - Use Cases (protocols + implementations)
   - Repository protocols
   - Domain errors

2. **Data** (Persistence & networking)
   - DTOs (Codable structs)
   - Mappers (DTO ↔ Entity)
   - Repository implementations
   - APIClient + persistence (CoreData, UserDefaults)

3. **Presentation** (UI & interaction)
   - ViewModels (ObservableObject, @Published, ViewState enums)
   - SwiftUI Views
   - Coordinators (NavigationPath-based)

### Dependency Injection: Protocol-Driven Manual
No DI frameworks. Use Swift protocols and manual wiring in a DIContainer.

### State Management: ViewState Enums + @Published
```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var state: ViewState = .loading

    enum ViewState { case loading, success(Data), failure(String), empty }
}
```

### File Structure
```
App/DI/Container.swift
Domain/Entities/, UseCases/, Repositories/, Errors/
Data/DTOs/, Mappers/, Repositories/, Network/, Storage/
Presentation/Features/[Feature]/Views/, ViewModels/, Coordinators/
```

### Layer Rules
- Domain imports only Foundation
- Data imports Domain (implements its protocols)
- Presentation imports Domain (consumes use cases), never imports Data
- App/DI wires everything together

### Code Quality
- **SwiftLint**: code style enforcement
- **SwiftFormat**: auto-formatting
- **XCTest**: unit + UI tests, protocol-based mocking

For comprehensive iOS standards: see `references/ios-standards.md`
For project templates and code examples: see `references/ios-architecture-template.md`

---

## Feedback Loops

### Loop 1: Build Fails (Compile Error)
**Trigger**: `xcodebuild` returns non-zero exit code

**Flow**: Build → Integration fixes issue → retry (max 3 times)

### Loop 2: Lint Violation (Auto-Fixable)
**Trigger**: SwiftLint reports violations

**Flow**: SwiftLint → SwiftFormat auto-fixes → re-lint → pass or escalate

### Loop 3: Layer Boundary Violation
**Trigger**: Integration audit detects import violations

**Flow**: Architect re-plans → affected layer adjusts imports → Integration re-audits

### Loop 4: Spec Unclear Mid-Coding
**Trigger**: Architect encounters ambiguous requirement

**Flow**: Pause → ask user to clarify → resume when clarified (max 10-min wait)

### Loop 5: PR Review Comments
**Trigger**: You leave comment on PR

**Flow**: Coder makes targeted fix → Build → Git amends commit → you re-review

For full feedback loop details: see `references/feedback-loops.md`

---

## When to Upgrade to ios-builder

Consider **ios-builder** if you need:
- Dynamic stage gating (skip certain phases per task)
- Pipeline memory (learn from previous runs)
- Deploy targets (App Store, TestFlight)
- MCP integrations (GitHub, Slack, Figma)
- Multi-feature coordination
- Team workflows with audit trails

---

## Troubleshooting

**Build fails repeatedly**: Check Integration audit output. Common cause: missing protocol conformance or layer violation.

**Wrong architecture decision**: Reject the PR, provide more detail in spec, restart from Architect phase.

**Spec too vague**: Reject at Review Gate 1, provide more detail in your original idea description.

---

## See Also

- `references/product-design-flow.md` — Product + Design phase details
- `references/code-gen-phases.md` — 4-phase Coder details
- `references/ios-standards.md` — Swift/SwiftUI standards
- `references/ios-architecture-template.md` — Code templates
- `references/build-and-git.md` — Build & Git details
- `references/feedback-loops.md` — Feedback loop details
