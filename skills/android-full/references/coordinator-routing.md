# Coordinator Routing

## Overview

The **Coordinator** is a router agent that analyzes the user's request and determines which of 9 workflow paths to follow. It ensures the right sub-agents (Product, Code Gen team, Test, Lint, Build, Deploy, Git) are engaged in the right sequence.

## 9 Routing Paths

### Path 1: Feature (New User Story)

**Triggers:**
- "Add user authentication"
- "Implement offline sync with Room"
- "New payment screen with Jetpack Compose"
- "Add Firebase Cloud Messaging"

**Workflow:**
```
Request
  ↓
Coordinator: "This is a new feature (multi-layer)"
  ↓
Product Agent: Gather spec (screens, API, error handling, persistence)
  ↓
Scaffold: Create folder structure, Hilt module skeleton
  ↓
Code Gen 4-Phase:
  Phase 1: Architect → Domain contracts
  Phase 2a & 2b (parallel): Domain Lead → Data Lead & Pres Lead
  Phase 3: Integration → Hilt assembly + validation
  ↓
Test Agent ‖ Lint Agent (parallel):
  Test: JUnit5 + MockK + Espresso domain/data/presentation tests
  Lint: ktlint format + detekt analysis
  ↓
Build Agent: ./gradlew assembleDebug
  ↓
Deploy Agent: Push to Firebase App Distribution (or local device)
  ↓
Git Agent: Create PR + push
```

**Duration:** 30–60 minutes (full feature with test + lint + build)
**Success Criteria:** All layers generated, tested, linted, compiled, PR created

---

### Path 2: Bug Fix (Targeted Layer Repair)

**Triggers:**
- "Fix login crash (NullPointerException in ViewModel)"
- "StateFlow not emitting in UserScreen"
- "Room migration error on app startup"
- "Hilt binding missing for AuthRepository"

**Coordinator Actions:**
1. Analyze error message/stack trace
2. Identify layer(s) involved:
   - **Domain error**: Use Case logic fault → Domain Lead
   - **Data error**: Retrofit/Room/mapper fault → Data Lead
   - **Presentation error**: ViewModel/Compose fault → Pres Lead
   - **Integration error**: Hilt binding fault → Integration

**Workflow (Example: ViewModel StateFlow Bug):**
```
Request: "StateFlow not emitting in UserScreen"
  ↓
Coordinator: Diagnose as Presentation layer bug
  ↓
Product Agent: Clarify scope (which screen, what state, when detected)
  ↓
Pres Lead: Examine ViewModel + Compose screen
  Potential fixes:
  - StateFlow not being collected with collectAsStateWithLifecycle()
  - viewModelScope.launch missing lifecycle awareness
  - UiState not being updated in viewModelScope
  ↓
Fix applied to ViewModel code
  ↓
Pres Lead: Write/update Compose test + ViewModel test
  ↓
Test Agent: Run tests (verify fix works)
  ↓
Lint Agent: ktlint + detekt on modified files
  ↓
Build Agent: ./gradlew build (compile check)
  ↓
Git Agent: Create PR (bug fix branch)
```

**Duration:** 10–20 minutes (targeted, single layer)
**Success Criteria:** Bug reproduced, fix applied, test passing, lint clean

---

### Path 3: Refactor (Code Reorganization)

**Triggers:**
- "Extract common ViewModel logic into BaseViewModel"
- "Reorganize Hilt modules (current: monolithic AppModule)"
- "Consolidate multiple Compose screens into single feature module"
- "Update Navigation graph for new feature structure"

**Workflow:**
```
Request: "Reorganize Hilt modules"
  ↓
Coordinator: Refactor path (no new features, code reorganization)
  ↓
Product Agent: Clarify scope (which modules, new structure, dependencies)
  ↓
Architect: Design new module structure + new @Module @InstallIn directives
  ↓
Integration: Execute refactor (move code, update imports, update bindings)
  ↓
Test Agent: Run existing tests (ensure refactor didn't break)
  ↓
Lint Agent: ktlint + detekt
  ↓
Build Agent: ./gradlew build
  ↓
Git Agent: Create PR (refactor branch with detailed changelog)
```

**Duration:** 20–40 minutes (code reorganization, minimal logic changes)
**Success Criteria:** Tests pass, lint clean, compile succeeds, structure cleaner

---

### Path 4: Design System (UI-Only Update)

**Triggers:**
- "Update Material Design 3 theme colors"
- "Create new reusable Compose component (Button, Card variant)"
- "Fix typography tokens across app"
- "Add accessibility labels to all screens"

**Workflow:**
```
Request: "Create Card component variant for dark mode"
  ↓
Coordinator: UI/Design layer only (Presentation)
  ↓
Product Agent: Spec new Card variant (colors, dimensions, material design rules)
  ↓
Pres Lead: Create CardVariant.kt Composable + Material Design 3 theming
  ↓
Pres Lead: Create Compose preview test
  ↓
Lint Agent: ktlint + detekt (style check)
  ↓
Build Agent: ./gradlew build (compile check)
  ↓
Git Agent: Create PR (design system update)
```

**Duration:** 5–15 minutes (UI-only, no domain/data logic)
**Success Criteria:** Component renders, preview works, compile succeeds

---

### Path 5: Sprint Batch (Multiple Features in Parallel)

**Triggers:**
- "Implement 3 user stories for sprint Q1"
- "Parallel feature development (auth + payment + notifications)"
- "Execute sprint task list with team"

**Workflow:**
```
Request: Sprint tasks [Auth, Payment, Notifications]
  ↓
Coordinator: Split into 3 parallel feature tasks
  ↓
Product Agent: Spec each feature independently
  ↓
For each feature (parallel):
  ├─ Scaffold
  ├─ Code Gen 4-Phase
  └─ (other features proceeding simultaneously)
  ↓
Consolidate: Merge all features
  ↓
Integration: Cross-feature dependency check (imports, Hilt bindings)
  ↓
Test Agent ‖ Lint Agent (full test suite, all features)
  ↓
Build Agent: ./gradlew build (all features)
  ↓
Deploy Agent: Firebase App Distribution (build with all features)
  ↓
Git Agent: Create multi-feature PR (with feature branch naming)
```

**Duration:** 60–90 minutes (parallel speedup vs. serial)
**Success Criteria:** All features working, no cross-feature conflicts, tests passing

---

### Path 6: Dependency Update (Gradle Upgrade)

**Triggers:**
- "Update Retrofit to 2.11"
- "Add Room 2.6 + update migrations"
- "Upgrade Gradle to 8.2"
- "Update Hilt to 2.48"
- "Add Kotlinx Serialization 1.6"

**Workflow:**
```
Request: "Update Retrofit 2.10 → 2.11"
  ↓
Coordinator: Dependency update (potential breaking changes)
  ↓
Product Agent: Clarify target version + known breaking changes
  ↓
Architect: Check Retrofit 2.11 changelog for API changes
  ↓
Data Lead: Update Retrofit service interfaces if API changed
  - Test (@Mock, @MockServer)
  ↓
Integration: Update build.gradle.kts dependency line
  Verify no import changes needed
  ↓
Test Agent: Run full test suite (ensure compatibility)
  ↓
Lint Agent: ktlint + detekt
  ↓
Build Agent: ./gradlew build (compile check)
  ↓
If version compatible:
  ↓
Git Agent: Create PR (dependency bump)

If version breaking:
  ↓
Integration: Manual intervention (code changes required, route to relevant lead)
```

**Duration:** 5–30 minutes (simple bump) or 30–60 minutes (breaking changes)
**Success Criteria:** Tests pass, compile succeeds, no API mismatches

---

### Path 7: PR Review (Code Review + Automated Checks)

**Triggers:**
- "Review PR #234"
- "Check Compose preview in PR"
- "Validate Hilt wiring in feature branch"
- "PR automation: layer boundaries check"

**Workflow:**
```
Request: "Review PR #234"
  ↓
Coordinator: PR review path
  ↓
Product Agent: Gather PR context (what was changed, why)
  ↓
Integration: Run automated checks:
  1. Layer boundary validation (Domain zero-Android, Pres no Data)
  2. Hilt binding completeness (@Binds, @Provides all repos/services)
  3. Navigation routes in NavHost
  4. UiState sealed class for all ViewModels
  ↓
Test Agent: Run test suite (check coverage %)
  ↓
Lint Agent: ktlint + detekt (report violations)
  ↓
Integration: Generate review report:
  - ✓ Boundaries valid
  - ✓ Hilt bindings complete
  - ✓ Navigation coverage
  - ⚠ Test coverage low (55%, recommend 70%+)
  - ✗ detekt: function complexity too high (suggest refactor)
  ↓
Output to GitHub PR comments (or human review tool)

Human maintainer reviews + merges (or requests changes)
```

**Duration:** 5–15 minutes (automated checks)
**Success Criteria:** All automated checks pass, report generated, human review ready

---

### Path 8: Release (Ship to App Store)

**Triggers:**
- "Ship v1.5.0 to Play Store"
- "Firebase App Distribution QA build"
- "Production release with versioning"

**Workflow:**
```
Request: "Release v1.5.0 to Play Store"
  ↓
Coordinator: Release path (high-stakes, signing required)
  ↓
Product Agent: Confirm release version, release notes, signing config
  ↓
Build Agent: ./gradlew assembleRelease (build with production signing config)
  - Requires env vars: KEYSTORE_PATH, KEYSTORE_PASSWORD, KEY_PASSWORD
  ↓
Deploy Agent: Upload signed APK to Google Play Console
  - Track: internal (staging), then production
  - Fastlane: `fastlane supply --apk app-release.apk --track internal`
  ↓
Git Agent: Create release tag
  - Tag name: v1.5.0
  - Commit message: "Release v1.5.0: [summary of changes]"
  ↓
Human approval: QA sign-off before production rollout
  ↓
Deploy Agent: Promote from internal → production (staged rollout 25%, 50%, 100%)
```

**Duration:** 15–45 minutes (build + upload + staging checks)
**Success Criteria:** Release APK built, signed, uploaded, staged rollout ready for human approval

---

### Path 9: Test Only (No Code Changes)

**Triggers:**
- "Add test coverage for UserRepository"
- "Write Compose UI tests for LoginScreen"
- "ViewModel state flow test"
- "Increase test coverage to 80%"

**Workflow:**
```
Request: "Add test coverage for UserRepository"
  ↓
Coordinator: Test-only path (no code changes, test code only)
  ↓
Product Agent: Clarify scope (which layer, what scenarios)
  ↓
Test Agent:
  1. Analyze existing UserRepository code
  2. Identify untested paths
  3. Generate JUnit5 + MockK test cases
  4. Create @Test methods for each scenario
  ↓
Test Agent: Run tests (verify new tests pass + coverage increases)
  ↓
Lint Agent: ktlint + detekt on test code
  ↓
Build Agent: ./gradlew test (compile + run tests)
  ↓
Coverage report generated (% increase shown)
  ↓
Git Agent: Create PR (test coverage branch)
```

**Duration:** 10–20 minutes (test-only work)
**Success Criteria:** Tests generated, passing, coverage increased, lint clean

---

## Coordinator Decision Tree

```
User Request
  │
  ├─ "Add/implement/new"?
  │  ├─ Multi-layer (user story) → Path 1: Feature
  │  └─ UI only → Path 4: Design System
  │
  ├─ "Fix/bug/crash"?
  │  └─ Path 2: Bug Fix
  │
  ├─ "Refactor/reorganize"?
  │  └─ Path 3: Refactor
  │
  ├─ "Update/upgrade/bump [dependency]"?
  │  └─ Path 6: Dependency Update
  │
  ├─ "Review PR/check/validate"?
  │  └─ Path 7: PR Review
  │
  ├─ "Release/ship/publish/v[version]"?
  │  └─ Path 8: Release
  │
  ├─ "Test/coverage/[layer]"?
  │  └─ Path 9: Test Only
  │
  └─ Multiple sprint tasks in list?
     └─ Path 5: Sprint Batch
```

## Coordinator State Management

Coordinator maintains context across routing decision:

```json
{
  "request_id": "coord-2025-02-11-001",
  "user_request": "Add user authentication",
  "routing_path": 1,
  "path_name": "Feature",
  "context": {
    "estimated_duration": "45 minutes",
    "stages": ["product", "scaffold", "code-gen-4phase", "test", "lint", "build", "deploy", "git"],
    "parallel_stages": ["test", "lint"],
    "deploy_target": "firebase-app-distribution"
  },
  "memory_patterns_loaded": 3,
  "next_agent": "product-agent",
  "timestamp": "2025-02-11T10:30:00Z"
}
```

Each downstream agent reads this context to understand scope, timeline, and memory patterns.

---

## Integration Points

### Coordinator ↔ Product Agent
- **Input:** Request + routing path + config + memory patterns
- **Output:** Detailed spec + mockup

### Coordinator ↔ Code Gen Team (for Feature path)
- **Input:** Spec + scaffold folder structure
- **Output:** Generated domain/data/presentation code

### Coordinator ↔ Git Agent (all paths)
- **Input:** All artifacts (code, tests, linting results)
- **Output:** PR URL + branch name

**Return to Coordinator:** "PR #345 created, awaiting human review at https://github.com/..."

## Troubleshooting Coordinator

### Problem: Request Ambiguous (Multiple Paths Match)
**Example:** "Add authentication tests to existing Auth module"
- Could be: Feature (Path 1) + Test Only (Path 9)

**Solution:**
- Coordinator asks for clarification:
  ```
  Is this:
  1. A new feature (new domain logic + screens)?
  2. Or test code only (no new functionality)?
  ```
- User selects → route confirmed

### Problem: Request Out of Scope
**Example:** "Rewrite app in Rust" (out of Android scope)

**Solution:**
- Coordinator reports:
  ```
  This request is outside Android Full Pipeline scope.
  Pipeline specializes in Kotlin/Jetpack Compose/Clean Architecture.

  Suggestions:
  - For Kotlin Multiplatform shared code: use shared-kit-pipeline
  - For quick prototypes: use android-lite-pipeline
  ```

### Problem: Memory Pattern Conflicts
**Example:** Previous run used Result<T>, current task uses Either<L, R>

**Solution:**
- Coordinator prompts:
  ```
  Pipeline Memory suggests Result<T> for error handling (from 3 prior runs).
  Current approach uses Either<L, R>.

  Override memory pattern? [Yes | No]
  ```
- User confirms override → memory updates on successful run
