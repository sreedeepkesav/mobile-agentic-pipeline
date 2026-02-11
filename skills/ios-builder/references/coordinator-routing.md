# Coordinator Routing Guide

Complete reference for the Coordinator agent in the ios-builder skill. Covers task classification, 9 routing paths, and feedback loop handling.

## Coordinator Role

The Coordinator performs exactly 3 functions:

1. **Classify** the incoming task type (feature, bug, refactor, design implementation, sprint batch, dependency update, PR review, release, test-only)
2. **Route** to the appropriate agent sequence
3. **Handle** feedback from agents (approvals, rejections, modifications)

The Coordinator does NOT write code, design, or tests. It orchestrates.

## Task Classification

Input: Task title, description, optional links (design, issue tracker, PR)

Classification logic uses keywords, context, and structural patterns:

```
Task Input: "Add user profile screen"
Keywords: add, screen, design
Links: [Figma design URL]
Classification: NEW FEATURE

Task Input: "Fix login crash on iOS 17"
Keywords: fix, crash, issue
Links: [GitHub issue #234]
Classification: BUG FIX

Task Input: "Refactor networking layer"
Keywords: refactor, networking, architecture
Classification: REFACTOR

Task Input: "Implement the new design from Figma"
Keywords: implement, design, Figma
Links: [Figma file]
Classification: DESIGN IMPLEMENTATION

Task Input: "5 tickets from sprint 12: login, settings, notifications"
Keywords: sprint, batch, multiple tickets
Classification: SPRINT BATCH

Task Input: "Update Alamofire to 5.8"
Keywords: update, dependency, version
Classification: DEPENDENCY/CONFIG UPDATE

Task Input: "Address comments on PR #89"
Keywords: PR, comments, review, address
Classification: PR REVIEW RESPONSE

Task Input: "Ship to TestFlight"
Keywords: ship, deploy, testflight, release
Classification: RELEASE/DEPLOY ONLY

Task Input: "Run unit tests and report coverage"
Keywords: tests, coverage, run
Classification: TEST ONLY
```

## 9 Routing Paths with Full Detail

### Path 1: New Feature

**Triggers:**
- "Add user profile", "Build settings page"
- Design file or wireframe link
- Epic or feature spec
- Keyword: "new", "add", "feature", "screen"

**Route:**
```
New Feature Task
  ↓
Product Agent (decompose, spec, design integration)
  ↓
Spec Approval Gate
  ├─ Approved → continue
  ├─ Edit → Product Agent incorporates changes → re-review
  └─ Reject → Product Agent re-researches → re-review
  ↓
Scaffold (if new module/layer needed)
  ↓
Code Gen (full team or single dev)
  │ ├─ Architect: blueprint, protocols, ADR
  │ ├─ Domain Lead: entities, use cases
  │ ├─ Data Lead: DTOs, mappers, repo impls
  │ └─ Presentation Lead: VMs, views, coordinators
  ↓
Test (parallel with Lint)
  ├─ Unit tests for each layer
  └─ Snapshot tests for views
  ↓
Lint (parallel with Test)
  ├─ SwiftLint violations
  └─ Architecture audit
  ↓
Build (compile, link, validate)
  ↓
Deploy (if configured; typically TestFlight)
  ↓
Git (create branch, commit, push, create PR)
```

**Code Gen Team Composition:**
- **Full Team (3+ developers):** Architect assigns layers; Domain Lead, Data Lead, Presentation Lead work in parallel; Staff Integration connects
- **Pair (2 developers):** Architect + one combined Data/Presentation Lead
- **Solo:** Single developer through all phases sequentially

**Example Output:**
```
Branch: feature/user-profile
Commits:
  - feat(domain): add UserProfile entity and protocols
  - feat(data): implement UserRepository
  - feat(presentation): build UserProfileView and ViewModel
  - feat(app): integrate UserProfile module
PR: "[FEATURE] Add user profile screen - Implements AC1-5, matches Figma design"
```

### Path 2: Bug Fix

**Triggers:**
- "Fix login crash", "Resolve background sync issue"
- GitHub issue or crash report
- Keyword: "fix", "bug", "crash", "issue"

**Route:**
```
Bug Fix Task
  ↓
Code Gen (targeted to affected layer only)
  │ ├─ Identify layer: Domain? Data? Presentation?
  │ └─ Layer Lead (not full team)
  ↓
Test (verify fix)
  ├─ Reproduce original issue
  ├─ Verify fix resolves it
  └─ Regression tests
  ↓
Lint (check violations introduced)
  ↓
Git (commit, push, reference issue)
```

**Skip Stages:** Product Agent, Scaffold, Build, Deploy

**Example Output:**
```
Branch: fix/login-crash
Commits:
  - fix(data): handle nil URLResponse in login flow
  - test(data): add test for nil URLResponse scenario
PR: "[BUG FIX] Fix login crash on background request timeout - Closes #234"
```

**Layer Lead Determination:**
- Crash in SwiftUI view → Presentation Lead
- Crash in decoding JSON → Data Lead
- Crash in business logic → Domain Lead
- Crash in DI container → Integration (Staff)

### Path 3: Refactor

**Triggers:**
- "Refactor networking layer", "Tech debt: simplify error handling"
- Tech debt ticket
- Keyword: "refactor", "tech debt", "simplify", "improve"

**Route:**
```
Refactor Task
  ↓
Code Gen (Principal Architect first)
  │ ├─ Architect re-plans the layer
  │ ├─ Reads existing code, determines refactor scope
  │ ├─ Writes ADR (why the refactor, tradeoffs)
  │ └─ Routes to affected layer leads
  │ ├─ Domain Lead (if domain logic refactor)
  │ ├─ Data Lead (if repository/API refactor)
  │ └─ Presentation Lead (if UI/VM refactor)
  ↓
Test (verify behavior unchanged)
  ├─ Run existing tests
  ├─ Add tests for edge cases uncovered
  └─ Coverage report
  ↓
Lint (check new patterns)
  ↓
Git (commit, link to tech debt)
```

**Skip Stages:** Product Agent, Scaffold, Build, Deploy

**Example Output:**
```
Branch: refactor/networking-consolidation
Docs: docs/decisions/ADR-003-consolidate-api-layers.md
Commits:
  - refactor(data): consolidate APIClient and NetworkManager
  - test(data): update tests for new structure
  - docs: add ADR for consolidation rationale
PR: "[REFACTOR] Consolidate networking layer - Reduces duplication, improves testability"
```

### Path 4: Design Implementation

**Triggers:**
- "Implement the design from Figma"
- Design file link (Figma, Sketch, etc.)
- Keyword: "design", "implement", "screen mockup"

**Route:**
```
Design Implementation Task
  ↓
Product Agent (decompose design to specs)
  │ ├─ Extract components from design file
  │ ├─ Map colors, spacing, typography to design tokens
  │ ├─ Identify reusable view components
  │ └─ Write AC: "Button matches design specs", "Spacing matches grid"
  ↓
Code Gen (Presentation primary, Data secondary)
  │ ├─ Presentation Lead: implement SwiftUI views, match design exactly
  │ ├─ Data Lead (if new API): handle data that feeds design
  │ └─ Architect: guide dependency injection for views
  ↓
Test (visual + data tests)
  ├─ Snapshot tests for views
  ├─ Test data loading behavior
  └─ Test interaction flows
  ↓
Lint (check view structure)
  ↓
Git (commit views, design tokens, snapshots)
```

**Skip Stages:** Scaffold (unless new module), Build, Deploy

**Example Output:**
```
Branch: design/new-checkout-flow
Outputs:
  - Presentation/Checkout/CheckoutView.swift (matches Figma)
  - Presentation/Checkout/CheckoutViewModel.swift
  - Presentation/DesignTokens/CheckoutTokens.swift
  - Tests: CheckoutViewSnapshotTests.swift
Commits:
  - feat(presentation): implement checkout flow from design
  - feat(design): add checkout design tokens
  - test(presentation): add snapshot tests for checkout screens
PR: "[DESIGN] Implement new checkout flow - Matches Figma specs, AC1-8 complete"
```

### Path 5: Sprint Batch

**Triggers:**
- "Do these 5 tickets from sprint 12"
- Sprint link or list of task IDs
- Keyword: "sprint", "batch", "5 tickets"

**Route:**
```
Sprint Batch (5+ tasks)
  ↓
Product Agent (prioritize and build DAG)
  │ ├─ Read all task specs
  │ ├─ Identify dependencies (task A blocks B)
  │ ├─ Order: independent tasks first, dependent follow
  │ └─ Output: task-graph.json with execution order
  ↓
Parallel Code Gen Execution
  ├─ Independent tasks run in parallel (separate Code Gen teams)
  ├─ Task A (feature): Arch → Domain → Data/Pres || → Test/Lint
  ├─ Task B (bug fix): Code Gen → Test/Lint (parallel with A)
  ├─ Task C (design): Product → Code Gen → Test/Lint (parallel with A, B)
  └─ Task D (depends on A): Wait for A complete, then Code Gen
  ↓
Consolidated Build
  ├─ Compile all changes
  └─ Verify no conflicts
  ↓
Consolidated Test
  ├─ Run all tests
  └─ Report coverage
  ↓
Git (one commit per task OR one squash commit per user story)
```

**Parallel Execution Rules:**
- Independent tasks execute simultaneously with separate agents
- Dependent tasks block until dependencies complete
- Same layer lead may work on multiple parallel tasks (managed by Coordinator)
- Git merges are sequential (to avoid conflicts); compile to main after each task completes

**Example DAG:**
```json
{
  "tasks": [
    {"id": "T1", "title": "Add user profile", "dependencies": [], "assigned": "dev1"},
    {"id": "T2", "title": "Fix login crash", "dependencies": [], "assigned": "dev2"},
    {"id": "T3", "title": "Implement settings design", "dependencies": ["T1"], "assigned": "dev1"},
    {"id": "T4", "title": "Update API client", "dependencies": ["T1", "T2"], "assigned": "dev3"}
  ],
  "execution_groups": [
    {
      "wave": 1,
      "tasks": ["T1", "T2"],
      "parallel": true
    },
    {
      "wave": 2,
      "tasks": ["T3"],
      "parallel": false,
      "blocks_on": ["T1"]
    },
    {
      "wave": 3,
      "tasks": ["T4"],
      "parallel": false,
      "blocks_on": ["T1", "T2"]
    }
  ]
}
```

**Example Output:**
```
Wave 1: T1 (Add Profile) + T2 (Fix Login Crash)
  - dev1 works on T1 Code Gen
  - dev2 works on T2 Code Gen (parallel)
  - Merge T1 to main
  - Merge T2 to main

Wave 2: T3 (Settings Design) depends on T1
  - dev1 starts T3 Code Gen (Profile VMs available)
  - Merge T3 to main

Wave 3: T4 (API Client) depends on T1 + T2
  - dev3 starts T4 Code Gen (Profile + Login changes integrated)
  - Merge T4 to main
```

### Path 6: Dependency/Config Update

**Triggers:**
- "Update Alamofire to 5.8"
- "Update Swift to 5.10"
- Keyword: "update", "dependency", "version", "SPM", "CocoaPods"

**Route:**
```
Dependency/Config Update Task
  ↓
Build Agent (handle dependency updates)
  │ ├─ Update Package.swift or Podfile
  │ ├─ Resolve version constraints
  │ └─ Fetch new dependency versions
  ↓
Test (verify compatibility)
  ├─ Build and link with new version
  ├─ Run existing test suite
  └─ Check for deprecation warnings
  ↓
Git (commit, tag if major version)
```

**Skip Stages:** Product Agent, Scaffold, Code Gen, Lint, Deploy

**Example Output:**
```
Branch: deps/update-alamofire-5.8
Commits:
  - build: update Alamofire to 5.8
  - build: update other compatible deps (URLSessionConfiguration)
  - test: verify network requests work with 5.8
PR: "[DEPS] Update Alamofire to 5.8 - Resolves compatibility issues with Xcode 15"
```

### Path 7: PR Review Response

**Triggers:**
- "Address comments on PR #89"
- PR review with requested changes
- Code review conversation
- Keyword: "PR", "review", "comments", "address"

**Route:**
```
PR Review Response Task
  ↓
Code Gen (targeted layer leads)
  │ ├─ Read PR and review comments
  │ ├─ Identify affected code sections
  │ └─ Route to layer lead
  │ ├─ Domain Lead (if domain logic change)
  │ ├─ Data Lead (if API/storage change)
  │ └─ Presentation Lead (if view/VM change)
  ↓
Test (verify changes still work)
  ├─ Run affected tests
  └─ Add new tests if suggested
  ↓
Lint (check new violations)
  ↓
Git (amend previous commit, force push to PR branch)
```

**Skip Stages:** Product Agent, Scaffold, Build, Deploy

**Example Output:**
```
PR #89: Add user profile
Review Comments:
  - "Extract profile editing to separate view"
  - "Add error handling for network failures"
  - "Match spacing to design tokens"

Code Gen Response:
  - Presentation Lead refactors views
  - Data Lead adds error handling
  - Updates design token usage

Amended Commits:
  - feat(presentation): split profile view into display and edit
  - feat(data): handle API errors in profile fetch
  - test(presentation): add tests for edit flow

Git: git push --force-with-lease origin feature/user-profile
```

### Path 8: Release/Deploy Only

**Triggers:**
- "Ship to TestFlight"
- "Release v2.0.0"
- Keyword: "release", "ship", "deploy", "testflight", "app store"

**Route:**
```
Release/Deploy Task
  ↓
Build (prepare release build)
  │ ├─ Verify all tests pass on main
  │ ├─ Generate release notes
  │ └─ Create release candidate
  ↓
Deploy (push to distribution channel)
  │ ├─ Upload to TestFlight or App Store
  │ └─ Monitor deployment
  ↓
Git (tag release, create release branch)
  │ └─ git tag v2.0.0
```

**Skip Stages:** Product Agent, Scaffold, Code Gen, Test, Lint (assume already passing)

**Example Output:**
```
Branch: release/v2.0.0
Builds:
  - Created release build (MyApp-2.0.0.ipa)
  - Signed with distribution certificate
Deploy:
  - Uploaded to TestFlight
  - Notifications sent to testers
Git:
  - git tag v2.0.0
  - git checkout -b release/2.0.x
  - Pushed release tag and branch
```

### Path 9: Test Only

**Triggers:**
- "Run tests and report coverage"
- "Check test coverage for Data layer"
- Keyword: "test", "coverage", "run tests"

**Route:**
```
Test-Only Task
  ↓
Test Agent (execute and report)
  │ ├─ Run specified tests (or all)
  │ ├─ Collect coverage metrics
  │ └─ Report gaps
  ↓
No other stages run
```

**Skip Stages:** Product Agent, Scaffold, Code Gen, Lint, Build, Deploy, Git

**Example Output:**
```
Test Results:
  - Data Layer: 82% coverage (3 methods untested)
  - Domain Layer: 95% coverage
  - Presentation Layer: 71% coverage (10 views missing snapshots)

Report: test-coverage-report.html
Recommendations:
  - Add tests for error scenarios in Data layer
  - Add snapshot tests for presentation components
```

## Task Routing Decision Tree

```
                    Task Input
                       ↓
                   CLASSIFY TYPE
                   ↙ ↓ ↓ ↓ ↘
              Feature Bug Refactor Design Sprint Deps PR Release Test
                ↓      ↓     ↓       ↓      ↓     ↓   ↓   ↓       ↓
                Path1  Path2 Path3  Path4  Path5 Path6 Path7 Path8  Path9

Keywords determine classification:
  - "add", "new", "feature" → Feature (Path 1)
  - "fix", "bug", "crash" → Bug Fix (Path 2)
  - "refactor", "tech debt" → Refactor (Path 3)
  - "design", "implement" + design file → Design (Path 4)
  - "sprint", "batch", multiple tasks → Sprint Batch (Path 5)
  - "update", "dependency" → Dependency (Path 6)
  - "PR", "review", "comments" → PR Review (Path 7)
  - "release", "deploy", "ship" → Release (Path 8)
  - "test", "coverage", "run" → Test Only (Path 9)

If ambiguous, check:
  - Is there a design file link? → Design Implementation
  - Does it mention multiple tasks? → Sprint Batch
  - Does it reference a PR? → PR Review Response
  - Is the goal deployment? → Release/Deploy
```

## Feedback Loop Handling

After each stage completes, the Coordinator receives output and decides:

### Approval
- **Condition:** Stage output meets requirements (spec approved, code compiles, tests pass)
- **Action:** Continue to next stage
- **Example:** Product Agent spec approved → proceed to Scaffold

### Edit/Modify
- **Condition:** Stage output needs tweaks but is on track
- **Action:** Return to same stage with feedback; agent incorporates and resubmits
- **Example:** "Spec is good but split this user story into 2 tasks" → Product Agent edits spec → resubmit

### Reject
- **Condition:** Stage output doesn't meet requirements; needs major rework
- **Action:** Return to same stage or back to prior stage; agent retries
- **Example:** "Compiler errors in Domain layer" → Domain Lead fixes → recompile → resubmit

### Escalate
- **Condition:** Issue involves multiple layers or architecture concern
- **Action:** Route to Architect or Integration agent for resolution
- **Example:** "Data layer changes break Presentation layer dependency" → Architect + leads align → resubmit

## Coordinator State and Memory

Coordinator tracks:
- **Current task:** What's being executed
- **Stage status:** Which stage, which agents active
- **Agent outputs:** Specs, code, test results, feedback
- **History:** All completed tasks in session (for reference and learning)

Stored in `.pipeline/coordinator-state.json`:

```json
{
  "current_task": {
    "id": "T1",
    "title": "Add user profile",
    "type": "feature",
    "route": "path_1",
    "status": "in_progress",
    "current_stage": "code_gen"
  },
  "completed_tasks": [
    {
      "id": "T0",
      "title": "Fix login crash",
      "type": "bug_fix",
      "route": "path_2",
      "status": "completed",
      "duration": "45 minutes"
    }
  ],
  "feedback_log": [
    {
      "task": "T1",
      "stage": "product_agent",
      "feedback": "Spec approved",
      "timestamp": "2024-01-15T10:30:00Z"
    }
  ]
}
```

## Ambiguous Task Handling

**If classification is unclear:**

Coordinator asks clarifying questions:

```
Task: "Fix the search"

Coordinator: Is this a bug (search is broken), a feature (new search capability),
or a refactor (improve search performance)?
User: Bug - search results are empty
Classification: BUG FIX → Route to Path 2
```

**If multiple paths apply:**

Coordinator defaults to most specific:
- Feature + Design file → Design Implementation (Path 4), not New Feature
- Refactor + Dependency update → Split into two tasks
- PR Review + new code → PR Review Response (Path 7) for review, then New Feature (Path 1) if follow-up is needed
