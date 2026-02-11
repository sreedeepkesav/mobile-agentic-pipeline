# Pipeline Memory Guide

Complete reference for the Pipeline Memory system in the ios-builder skill. Covers persistence, categories, agent contributions, querying, and examples.

## Overview

Pipeline Memory is an always-active append-only knowledge base that records decisions, learnings, mistakes, and patterns throughout the project's development. It enables the pipeline to:

1. **Learn progressively:** Each task informs future tasks
2. **Avoid repeating mistakes:** Corrections prevent future bugs
3. **Establish patterns:** Recognized code patterns become team conventions
4. **Make informed decisions:** Architecture choices reference past decisions

Pipeline Memory is local (per project) and can be shared (via Git) or private (via .gitignore).

## Persistence Layer

Memory is stored in `.pipeline/memory/` with an append-only log structure.

### Directory Structure

```
.pipeline/
└─ memory/
   ├─ decisions/
   │  ├─ 2024-01-15T10:30:00Z-async-await-over-combine.md
   │  ├─ 2024-01-15T14:00:00Z-userdefaults-for-drafts.md
   │  └─ index.json
   ├─ learnings/
   │  ├─ 2024-01-16T09:15:00Z-swiftui-state-management.md
   │  ├─ 2024-01-17T11:45:00Z-protocol-oriented-domain.md
   │  └─ index.json
   ├─ mistakes/
   │  ├─ 2024-01-16T15:30:00Z-circular-import-data-presentation.md
   │  ├─ 2024-01-17T13:20:00Z-forgot-codable-dto.md
   │  └─ index.json
   ├─ patterns/
   │  ├─ 2024-01-15T16:00:00Z-di-assembly-pattern.md
   │  ├─ 2024-01-16T10:00:00Z-viewmodel-with-async-load.md
   │  └─ index.json
   └─ config.json  # retention settings, enabled categories
```

### Index Files

Each category has an `index.json` that allows fast queries without scanning all files.

**decisions/index.json:**
```json
{
  "entries": [
    {
      "timestamp": "2024-01-15T10:30:00Z",
      "filename": "2024-01-15T10:30:00Z-async-await-over-combine.md",
      "agent": "architect",
      "title": "Use Async/Await Instead of Combine",
      "tags": ["architecture", "async", "swift"],
      "keywords": ["async/await", "combine", "readability", "error handling"]
    },
    {
      "timestamp": "2024-01-15T14:00:00Z",
      "filename": "2024-01-15T14:00:00Z-userdefaults-for-drafts.md",
      "agent": "data-lead",
      "title": "UserDefaults for Draft Storage",
      "tags": ["storage", "drafts", "userdefaults"],
      "keywords": ["userdefaults", "draft persistence", "codable"]
    }
  ]
}
```

## 4 Memory Categories

### Category 1: Decisions

Records architectural decisions, design choices, and technical direction.

**Triggered by:** Architect, Coordinator (major routing decisions), Code Gen leads (pattern choices)

**Example: decisions/2024-01-15T10:30:00Z-async-await-over-combine.md**

```markdown
# Decision: Use Async/Await Instead of Combine

**Date:** 2024-01-15
**Agent:** Principal Architect
**Feature:** User Profile
**Status:** Approved

## Context
We need to implement asynchronous data fetching in the User Profile feature.
The codebase has both Combine and async/await usage.

## Decision
Use async/await throughout User Profile (domain, data, presentation layers).

## Rationale
- **Readability:** Async/await is sequential (looks like sync code); Combine is declarative (chains of operators)
- **Error Handling:** Try/catch is clearer than `.mapError()` and `.catch()`
- **Swift Future:** Async/await is the Swift concurrency model; Combine is legacy
- **Team Expertise:** Team is growing in async/await; Combine knowledge is declining
- **iOS Support:** Our min target is 14.0; async/await available since iOS 13.0

## Tradeoffs
- **Pro:** Cleaner code, easier tests, better performance (structured concurrency)
- **Con:** Some Combine operators not available; integration with legacy Combine code requires bridges
- **Neutral:** Coexists with Combine in other features

## Alternatives
1. **Pure Combine:** More consistent with legacy features, but harder to read
2. **RxSwift:** Familiar to Android devs, but overkill for this app
3. **Callbacks:** Simple, but callback hell for complex flows

## Impact
- All new async operations use async/await
- Existing Combine code remains (no forced migration)
- Bridges created where async/await calls Combine publishers

## Related
- ADR-005 in docs/decisions/
- Code Gen Phase 1: Blueprint decision documented in ADR
```

### Category 2: Learnings

Records knowledge gained from implementing features, testing, and debugging.

**Triggered by:** Any agent after completing a task phase

**Example: learnings/2024-01-16T09:15:00Z-swiftui-state-management.md**

```markdown
# Learning: SwiftUI State Management Patterns

**Date:** 2024-01-16
**Agent:** Presentation Lead
**Feature:** User Profile Edit Screen
**Confidence:** High

## What We Learned
SwiftUI state management requires careful scoping:
1. **@State** for local, transient UI state (text input, toggle)
2. **@StateObject** for ViewModel (persistent across view recreations)
3. **@Published** on ViewModel properties (observable values)
4. **@EnvironmentObject** for app-wide state (user session, theme)

## Why This Matters
Incorrect state scoping causes bugs:
- Using @State for ViewModel → recreates ViewModel on every parent update
- Using @StateObject on child views → creates new instance per child
- Forgetting @Published → SwiftUI doesn't observe changes

## Code Pattern (Correct)

```swift
@MainActor
class UserProfileViewModel: ObservableObject {
    @Published var state: UserProfileViewState = .loading
    // ViewModel persists; @Published updates trigger view refresh
}

struct UserProfileView: View {
    @StateObject var viewModel: UserProfileViewModel
    @State var isEditing = false  // Local state
    // @StateObject persists ViewModel across redraws
}
```

## Where to Apply
- New ViewModels: Use @StateObject in parent view
- New Views: Use @State for local toggles, dropdowns
- App State: Use @EnvironmentObject for Session, Theme

## Related
- Design pattern: docs/patterns/swiftui-architecture.md
- Example code: Presentation/ViewModels/
- Common mistake: Using @State for observable data

## References
- WWDC 2021 Session 10145: "Demystify SwiftUI"
- https://developer.apple.com/tutorials/swiftui/state-and-data-flow
```

### Category 3: Mistakes + Corrections

Records bugs, compilation errors, and how they were fixed. Critical for prevention.

**Triggered by:** Any agent when encountering and fixing an error

**Example: mistakes/2024-01-16T15:30:00Z-circular-import-data-presentation.md**

```markdown
# Mistake: Circular Import Between Data and Presentation

**Date:** 2024-01-16
**Agent:** Integration (Staff)
**Feature:** User Profile
**Severity:** Critical (blocked compilation)

## What Went Wrong
Integration wired DependencyContainer incorrectly:

```swift
// WRONG:
import Presentation  // Data imports Presentation!

class DependencyContainer {
    let viewModel = UserProfileViewModel(...)  // Creates ViewModel in DependencyContainer
}
```

**Error:** Circular dependency: Presentation imports Data, Data imports Presentation.

## Why It Happened
- Misunderstood layer separation
- Thought DependencyContainer (Data layer) should create ViewModels
- Didn't review layer architecture diagram before wiring

## The Fix
Move DependencyContainer to App layer (not Data layer).
Data layer provides repositories and use cases via protocols.
App layer assembles ViewModels and injects dependencies.

```swift
// CORRECT:
// App/DependencyContainer.swift
import Domain
import Data
import Presentation

class AppDependencyContainer {
    let userRepository: UserRepository = UserRepositoryImpl(...)
    let fetchUseCase: FetchUserUseCase = FetchUserUseCaseImpl(...)

    func userProfileViewModel(userId: String) -> UserProfileViewModel {
        return UserProfileViewModel(fetchUseCase: fetchUseCase, ...)
    }
}

// Data layer
import Domain  // OK: Data depends on Domain
// Does not import Presentation or App

// Presentation layer
import Domain  // OK: Presentation depends on Domain
// Does not import Data or App
```

## Prevention Going Forward
1. **Layer Diagram:** Always review docs/architecture/layer-diagram.md before Phase 4 (Integration)
2. **Import Audit:** Lint step checks for forbidden imports (Presentation in Data, etc.)
3. **Code Review:** Integration stage gates check circular dependency

## Related
- Architecture: docs/architecture/layered-mvvm-c.md
- Integration phase: code-gen-phases.md Phase 4
- Lint violations: lib/lint-rules/no-circular-imports.swift

## Severity Impact
- **Compilation blocked:** 30 minutes delay
- **Team:** All devs blocked until resolved
- **Prevention cost:** 5 minutes in code review
```

### Category 4: Patterns

Records code patterns, conventions, and best practices established in the codebase.

**Triggered by:** Any agent when implementing a recognized, reusable pattern

**Example: patterns/2024-01-15T16:00:00Z-di-assembly-pattern.md**

```markdown
# Pattern: DI Assembly Pattern

**Date:** 2024-01-15
**Agent:** Principal Architect + Integration
**Feature:** User Profile (first feature in codebase)
**Maturity:** Established

## Pattern Description

A centralized DependencyContainer using lazy properties for dependency injection.

### Structure

```swift
final class DependencyContainer {
    // MARK: - Singletons (network, storage)
    private lazy var apiClient: APIClient = {
        APIClientImpl(baseURL: config.apiBaseURL)
    }()

    private lazy var localStorage: UserLocalStorage = {
        UserLocalStorageImpl()
    }()

    // MARK: - Domain Layer
    private lazy var userRepository: UserRepository = {
        UserRepositoryImpl(apiClient: apiClient, localStorage: localStorage)
    }()

    // MARK: - Presentation (factory methods, not singletons)
    func userProfileViewModel(userId: String) -> UserProfileViewModel {
        UserProfileViewModel(
            userId: userId,
            fetchUseCase: FetchUserUseCaseImpl(repository: userRepository),
            updateUseCase: UpdateUserUseCaseImpl(repository: userRepository)
        )
    }
}
```

### Rules

1. **Singletons:** Network clients, databases, file system → lazy + private
2. **Factory Methods:** ViewModels, Coordinators → factory functions, not lazy
3. **Injection:** All dependencies passed via constructor; no service locator
4. **Testing:** MockDependencyContainer overrides lazy properties for tests

### Benefits

- **Centralized:** All dependencies in one place; easy to audit
- **Lazy:** Dependencies created only when first used
- **Testable:** MockDependencyContainer swaps implementations
- **Scalable:** Grows linearly with features

### Drawbacks

- **Single File:** DependencyContainer can grow large (100+ lines per feature)
- **Compile Time:** Large lazy blocks can slow compilation
- **Testing Boilerplate:** Each test needs MockDependencyContainer

### When to Use

- Simple to medium projects (< 20 features)
- Stable architecture (patterns established)
- Team learning async/await + protocol-based design

### Alternative Patterns

1. **Modular Containers:** Separate container per feature module
2. **Service Locator:** Global registry (not recommended; testing harder)
3. **Factory Functions:** Loose factories instead of centralized container

## Code Examples

### Creating a ViewModel
```swift
let container = AppDependencyContainer()
let viewModel = container.userProfileViewModel(userId: "123")
```

### Testing with Mocks
```swift
class MockDependencyContainer: DependencyContainer {
    override var apiClient: APIClient {
        return MockAPIClient()
    }
}
```

### Adding a New Feature
1. Create use case classes in Domain
2. Create repository implementation in Data
3. Add lazy properties to DependencyContainer
4. Add factory method for ViewModels/Coordinators

## Related Patterns

- VMM-C Navigation: patterns/2024-01-16T10:00:00Z-viewmodel-with-async-load.md
- Error Handling: patterns/custom-error-types.md

## References

- Apple: Dependency Injection Frameworks
- Hacking with Swift: Dependency Injection
- Internal: docs/architecture/dependency-injection.md
```

## Project Context (Project Brain)

Beyond execution memory (decisions, learnings, mistakes, patterns), Pipeline Memory also maintains a **Project Context** — a living snapshot of the project itself. This context grows automatically as the pipeline works on the project and agents discover its shape.

### Storage

```
.pipeline/memory/
└─ context/
   ├─ api-registry.json       # Known endpoints, auth patterns, response shapes
   ├─ design-system.json      # Components, tokens, theme conventions
   ├─ dependency-registry.json # Libraries, versions, rationale
   ├─ module-map.json          # App modules, ownership, relationships
   ├─ conventions.json         # Team naming, code style, PR process
   └─ domain-model.json       # Entities, relationships, business glossary
```

### API Registry

Tracks every API endpoint the pipeline discovers or creates.

```json
{
  "base_url": "https://api.myapp.com/v2",
  "auth": {
    "type": "bearer_jwt",
    "token_endpoint": "/auth/token",
    "refresh_endpoint": "/auth/refresh",
    "header": "Authorization: Bearer {token}"
  },
  "endpoints": [
    {
      "path": "/users/{id}",
      "method": "GET",
      "request": null,
      "response": "UserDTO",
      "auth_required": true,
      "added_by_feature": "user-profile",
      "date_added": "2024-01-15"
    },
    {
      "path": "/posts",
      "method": "POST",
      "request": "CreatePostRequest",
      "response": "PostDTO",
      "auth_required": true,
      "added_by_feature": "create-post",
      "date_added": "2024-01-17"
    }
  ]
}
```

**How agents use it:**
- **Product Agent**: Checks existing endpoints before proposing new ones (avoids duplicates)
- **Data Lead**: Knows the auth pattern, base URL, and existing DTOs
- **Architect**: Sees the full API surface when planning new features

### Design System Registry

Captures the project's design language as it evolves.

```json
{
  "theme": {
    "primary": "#007AFF",
    "secondary": "#5856D6",
    "background": "#F2F2F7",
    "surface": "#FFFFFF",
    "error": "#FF3B30",
    "typography": {
      "title": "SF Pro Display, 28pt, Bold",
      "body": "SF Pro Text, 17pt, Regular",
      "caption": "SF Pro Text, 13pt, Regular"
    },
    "spacing": { "xs": 4, "sm": 8, "md": 16, "lg": 24, "xl": 32 }
  },
  "components": [
    {
      "name": "PrimaryButton",
      "file": "Presentation/Components/PrimaryButton.swift",
      "props": ["title: String", "action: () -> Void", "isLoading: Bool"],
      "usage_count": 8
    },
    {
      "name": "ErrorBanner",
      "file": "Presentation/Components/ErrorBanner.swift",
      "props": ["message: String", "retryAction: (() -> Void)?"],
      "usage_count": 5
    }
  ],
  "conventions": {
    "loading_state": "ProgressView() centered in container",
    "empty_state": "VStack with illustration + message + CTA button",
    "error_state": "ErrorBanner at top + retry button"
  }
}
```

**How agents use it:**
- **Presentation Lead**: Reuses existing components instead of creating duplicates
- **Product Agent**: References established UI patterns when writing specs
- **Architect**: Knows the component library size when estimating effort

### Dependency Registry

Tracks all third-party dependencies and why they were chosen.

```json
{
  "package_manager": "spm",
  "dependencies": [
    {
      "name": "Alamofire",
      "version": "5.8.0",
      "purpose": "HTTP networking",
      "rationale": "Simpler than raw URLSession for complex request chains",
      "layer": "data",
      "added_by_feature": "initial-setup",
      "date_added": "2024-01-10"
    },
    {
      "name": "KeychainAccess",
      "version": "4.2.2",
      "purpose": "Secure token storage",
      "rationale": "Type-safe Keychain wrapper, avoids raw SecItem APIs",
      "layer": "data",
      "added_by_feature": "auth",
      "date_added": "2024-01-15"
    }
  ]
}
```

**How agents use it:**
- **Architect**: Knows what's available before proposing new libraries
- **Data Lead**: Uses existing networking/storage dependencies instead of adding alternatives
- **Integration**: Verifies no duplicate or conflicting dependencies

### Module Map

Captures how the app is organized — what owns what.

```json
{
  "modules": [
    {
      "name": "Auth",
      "type": "feature",
      "layers": ["domain", "data", "presentation"],
      "entities": ["User", "AuthToken", "Session"],
      "screens": ["LoginScreen", "RegisterScreen", "ForgotPasswordScreen"],
      "dependencies": ["Networking", "Storage"],
      "owner": "auth-team"
    },
    {
      "name": "Networking",
      "type": "infrastructure",
      "layers": ["data"],
      "provides": ["APIClient", "TokenInterceptor", "NetworkMonitor"],
      "used_by": ["Auth", "Posts", "Profile"]
    }
  ],
  "shared_modules": [
    {
      "name": "DesignSystem",
      "type": "ui-kit",
      "components": 12,
      "used_by": ["Auth", "Posts", "Profile", "Settings"]
    }
  ]
}
```

**How agents use it:**
- **Coordinator**: Knows which module a task belongs to
- **Architect**: Sees module dependencies before adding cross-module calls
- **Product Agent**: Understands existing feature boundaries

### Team Conventions

Captures naming, formatting, and process conventions as they emerge.

```json
{
  "naming": {
    "branches": "feature/{ticket-id}-{short-description}",
    "commits": "conventional: feat:, fix:, refactor:, test:, docs:",
    "files": {
      "views": "{Feature}View.swift",
      "viewmodels": "{Feature}ViewModel.swift",
      "coordinators": "{Feature}Coordinator.swift",
      "use_cases": "{Action}{Entity}UseCase.swift",
      "repositories": "{Entity}Repository.swift",
      "dtos": "{Entity}DTO.swift"
    }
  },
  "code_style": {
    "access_control": "explicit (internal is written, not inferred)",
    "optionals": "guard-let preferred over if-let for early exit",
    "closures": "trailing closure syntax for single-closure params",
    "async": "async/await preferred over Combine"
  },
  "process": {
    "pr_size": "max 400 lines changed",
    "review_required": true,
    "ci_must_pass": true,
    "squash_on_merge": true
  }
}
```

**How agents use it:**
- **All agents**: Follow established naming and style from first line of code
- **Git Agent**: Uses the branch naming and commit conventions
- **Integration**: Validates conformance to code style conventions

### Domain Model

Captures the business domain — entities, relationships, and a glossary of terms.

```json
{
  "entities": [
    {
      "name": "User",
      "fields": ["id: UUID", "name: String", "email: String", "avatarURL: URL?"],
      "relationships": ["has many Posts", "has one Session"],
      "business_rules": ["Email must be unique", "Name 2-50 chars"]
    },
    {
      "name": "Post",
      "fields": ["id: UUID", "title: String", "body: String", "authorId: UUID", "createdAt: Date"],
      "relationships": ["belongs to User", "has many Comments"],
      "business_rules": ["Title required, max 200 chars", "Body required, max 5000 chars"]
    }
  ],
  "glossary": {
    "Session": "Active authentication state with JWT token and refresh token",
    "Draft": "Unsaved post stored locally in UserDefaults until published"
  }
}
```

**How agents use it:**
- **Product Agent**: References existing entities when decomposing features
- **Domain Lead**: Reuses existing entities, avoids creating duplicates
- **Data Lead**: Knows entity relationships for API contract design

### Context Auto-Population

Project Context is populated **automatically** — you don't configure it manually.

**On bootstrap (first run):**
- Scans `Package.swift` or `Podfile` → populates dependency registry
- Scans project folders → populates module map
- Scans Swift files for entity definitions → populates domain model
- Detects Xcode settings → populates conventions

**During each run:**
- Product Agent adds new entities to domain model
- Data Lead adds new endpoints to API registry
- Presentation Lead registers new components in design system
- Integration updates module dependencies

**Progressive enrichment:** Context starts sparse and gets richer with every pipeline run. By run 5, the pipeline knows your project deeply.

### Context Queries

Agents query project context the same way they query execution memory:

```
context.api("user endpoints")         → all /users/* endpoints
context.components("button")          → PrimaryButton, SecondaryButton
context.module("Auth")                → full Auth module info
context.entity("User")               → User fields, relationships, rules
context.dependency("networking")      → Alamofire details
context.convention("branch naming")   → feature/{ticket-id}-{desc}
```

## Agent Contributions

### What Each Agent Writes and Reads

**Bootstrap Agent**
- Writes: Project context, detected architecture, MCP capabilities
- Reads: (Initial run only; no prior memory)
- Context: Writes module-map, dependency-registry, domain-model on first run

**Product Agent**
- Writes: Feature specs, decomposition rationale, design maps
- Reads: Learnings (design patterns), patterns (UX conventions)
- Context: Reads context.entity() before proposing new domain objects; writes new entities to domain-model

**Architect**
- Writes: Blueprint decisions, ADRs, architectural patterns
- Reads: Prior decisions (architecture consistency), mistakes (avoid repeating)
- Context: Reads context.module() and context.dependency() when planning features; validates module dependencies

**Domain Lead**
- Writes: Domain entity patterns, use case structure
- Reads: Learnings (domain modeling), patterns (error handling)
- Context: Reads context.entity() to avoid duplicates; writes new entities and relationships to domain-model

**Data Lead**
- Writes: API integration patterns, storage strategies, mapper conventions
- Reads: Learnings (API design), patterns (repository implementation), mistakes (integration bugs)
- Context: Reads context.api() for existing endpoints; writes new endpoints and auth patterns to api-registry

**Presentation Lead**
- Writes: ViewModel patterns, SwiftUI component structures, navigation flows
- Reads: Learnings (state management), patterns (ViewModel structure), mistakes (SwiftUI pitfalls)
- Context: Reads context.components() to reuse existing UI; writes new components to design-system

**Integration (Staff)**
- Writes: DI decisions, layer separation audits, conformance patterns
- Reads: Decisions (architecture), patterns (DI assembly), mistakes (circular dependencies)
- Context: Reads context.module() and context.dependency() for wiring; updates module-map dependencies

**Test Agent**
- Writes: Test patterns, mock strategies, coverage insights
- Reads: Patterns (mock setup), learnings (testability patterns)
- Context: Reads context.api() and context.module() for test scope

**Lint Agent**
- Writes: Violation patterns, auto-fix strategies, architectural violations
- Reads: Patterns (layer separation), mistakes (common violations)
- Context: Reads context.convention() and context.module() to validate code style and architecture

**Build/Deploy Agent**
- Writes: Build configuration patterns, signing, deployment strategies
- Reads: Learnings (build optimization), mistakes (versioning errors)
- Context: Reads context.dependency() for version resolution

**Coordinator**
- Writes: Routing decisions, feedback loop resolutions, task classification learnings
- Reads: Decisions (major routing), patterns (task types)
- Context: Reads context.module() to route tasks to correct feature owners

## Querying Memory

### Query API

Memory provides query methods for agents.

**Syntax:**
```swift
memory.query("networking patterns")
  .category(.patterns)
  .tags(["async", "network"])
  .recency(.last_7_days)
  .limit(5)
  .execute()
```

**Results:** Ranked by recency + relevance

**Methods:**

```swift
// By keyword
let results = memory.query("error handling")

// By agent
let results = memory.query(agent: .architect)

// By category
let results = memory.query(category: .learnings)

// By tag
let results = memory.query(tags: ["swiftui", "state"])

// Mistakes only
let mistakes = memory.mistakes("circular import")

// Patterns only
let patterns = memory.patterns("repository")

// Decisions only
let decisions = memory.decisions("async/await")

// Complex
let results = memory
  .query("viewmodel")
  .inCategory(.patterns)
  .byAgent(.presentation_lead)
  .from(Date().addingTimeInterval(-7*24*3600))  // Last 7 days
  .limit(3)
  .execute()
```

### Query Result

```json
{
  "timestamp": "2024-01-16T09:15:00Z",
  "filename": "2024-01-16T09:15:00Z-swiftui-state-management.md",
  "category": "learnings",
  "agent": "presentation-lead",
  "title": "SwiftUI State Management Patterns",
  "relevance_score": 0.92,
  "excerpt": "SwiftUI state management requires careful scoping: @State for local UI state, @StateObject for ViewModel...",
  "tags": ["swiftui", "state", "viewmodel"],
  "related_entries": [
    "patterns/2024-01-16T10:00:00Z-viewmodel-with-async-load.md"
  ]
}
```

## Example: Progressive Learning Over 3 Runs

### Run 1: User Profile Feature

**Decision:** Use async/await for networking
- decisions/2024-01-15T10:30:00Z-async-await-over-combine.md

**Mistake:** Forgot Codable on DTO
- mistakes/2024-01-15T14:30:00Z-forgot-codable-dto.md
- Fix: Add Codable conformance
- Prevention: Add lint rule to detect non-Codable DTOs

**Pattern:** DI Assembly pattern established
- patterns/2024-01-15T16:00:00Z-di-assembly-pattern.md

### Run 2: Create Post Feature

**Learning:** Applied DI pattern successfully
- learnings/2024-01-16T09:15:00Z-swiftui-state-management.md

**Mistake:** Circular import (Data → Presentation)
- mistakes/2024-01-16T15:30:00Z-circular-import-data-presentation.md
- Fix: Move DependencyContainer to App layer
- Prevention: Lint rule checks layer imports

**Decision:** Extend async/await pattern to post creation
- decisions/2024-01-16T16:00:00Z-consistent-async-await-pattern.md

### Run 3: Notifications Feature

**Applied Learnings:**
- Query: `memory.patterns("viewmodel")`
  - Returns: Pattern from Run 1 + Run 2
  - Result: Consistently apply ViewModel with @StateObject
- Query: `memory.mistakes("circular import")`
  - Returns: Run 2 mistake + fix
  - Result: Immediately aware of layer separation issue

**New Pattern:** Identified async error handling strategy
- patterns/2024-01-17T11:00:00Z-async-error-handling.md

**Collective Knowledge:**
- 5+ decisions recorded
- 12+ learnings recorded
- 4+ mistakes + fixes recorded
- 8+ patterns established

**Pipeline Benefit:** Run 3 implements notifications 40% faster due to established patterns and documented mistakes.

## Configuration and Sharing

### .pipeline/memory/config.json

```json
{
  "enabled": true,
  "categories": {
    "decisions": true,
    "learnings": true,
    "mistakes": true,
    "patterns": true
  },
  "retention": {
    "days": 90,
    "max_entries_per_category": 100
  },
  "sharing": {
    "git_committed": false,
    "gitignore_path": ".gitignore"
  },
  "privacy": {
    "sensitive_patterns": ["api_key", "token", "password"]
  }
}
```

### Git Sharing

**For Team Projects:**
```bash
# Commit memory to Git
git_committed: true

# Result: All team members share decisions, learnings, mistakes, patterns
# Benefit: New team members learn project history
```

**For Personal/Solo Projects:**
```bash
# Don't commit memory (keep local)
git_committed: false

# Result: .pipeline/memory/ added to .gitignore
# Benefit: Memory is personal; not pushed to remote
```

### Memory Lifecycle

**Auto-Cleanup:**
- Entries older than 90 days archived (optional)
- Mistakes archived after fix is applied (but kept for reference)
- Patterns kept indefinitely (they're "permanent" team knowledge)

**Manual Management:**
```
.pipeline/memory/archive/  (optional, for old entries)
.pipeline/memory/current/  (active entries)
```

## Integration with Pipeline Stages

### Bootstrap Stage
- Reads: None (first run)
- Writes: Project context, detected architecture

### Product Agent Stage
- Reads: Learnings (UX patterns), patterns (feature decomposition)
- Writes: Spec rationale, design decisions

### Code Gen Stage
- Reads: Decisions (architecture), patterns (DI, error handling), mistakes (avoid bugs)
- Writes: Blueprint decisions, implementation patterns, mistakes + fixes

### Test Stage
- Reads: Patterns (mock setup), mistakes (test coverage)
- Writes: Test patterns, coverage insights

### Lint Stage
- Reads: Patterns (layer separation), mistakes (violations)
- Writes: Violation patterns, auto-fix strategies

### Build Stage
- Reads: Learnings (build optimization)
- Writes: Build patterns, dependency resolution insights

### Deploy Stage
- Reads: Decisions (versioning), mistakes (deployment issues)
- Writes: Deployment patterns, release strategies

### Git Stage
- Reads: Memory summary (for commit message context)
- Writes: Commit reference to memory (optional)
