# Feedback Loops: Self-Healing Pipeline

## Overview

The iOS Lite Pipeline has 5 self-healing feedback loops that catch and automatically fix common issues without requiring full re-runs of earlier phases.

Each loop has:
- **Trigger**: What causes the loop to activate
- **Route**: Which phase handles the fix
- **Fix**: What gets corrected
- **Resolution**: How success is verified
- **Max retries**: How many times to attempt before escalating to user

---

## Loop 1: Build Fails (Compile Error)

### Trigger

`xcodebuild` returns non-zero exit code. Errors reported:
- Missing import
- Undeclared identifier
- Type mismatch
- Protocol not conformed
- Circular dependency

### Example Errors

```
error: [Domain/Entities/Timer.swift:12] Use of undeclared identifier 'TimeInterval'

error: [Presentation/ViewModels/TimerListViewModel.swift:8]
'TimerListViewModel' does not conform to protocol 'ObservableObject'

error: [Data/Repositories/DefaultTimerRepository.swift:15]
'DefaultTimerRepository' does not conform to required protocol 'TimerRepository'
```

### Route

**Build** → detects error → **Integration Phase** (protocol conformance, DI wiring, imports)

### Fix Process

1. **Integration receives error details**:
   - File path
   - Line number
   - Error message

2. **Integration analyzes root cause**:
   - Is it a missing import? Add it to the file.
   - Is it a protocol not conformed? Check DI container or implementation.
   - Is it a type mismatch? Check Domain entity vs DTO mapping.
   - Is it a circular dependency? Re-plan imports.

3. **Integration makes targeted fix**:
   ```swift
   // BEFORE: Missing import
   struct Timer {
       let duration: TimeInterval  // ❌ Error: Use of undeclared identifier
   }

   // AFTER: Integration adds import
   import Foundation
   struct Timer {
       let duration: TimeInterval  // ✅ Fixed
   }
   ```

4. **Build retries**:
   ```bash
   xcodebuild build ...  # Retry
   ```

### Resolution

- **Success**: Build exits with code 0 → proceed to Git
- **Failure**: Build still exits non-zero → integrate tries again

### Max Retries

**3 attempts**. If build fails all 3 times:
1. Integration creates detailed error log
2. Pipeline halts
3. Escalates to user with error log and request for manual intervention

### Example Scenario

```
Attempt 1: Build fails
  Error: Protocol 'GetTimersUseCase' does not conform to 'ObservableObject'
  Root cause: ViewModel expects UseCase to be ObservableObject (wrong pattern)
  Integration fix: Realizes UseCase should not be ObservableObject, ViewModel should be
  Adds @MainActor, @Published to ViewModel

Retry 1: Build succeeds ✅
  Route to Git phase
```

---

## Loop 2: Lint Violations (Auto-Fixable)

### Trigger

SwiftLint reports violations after build succeeds. Types:
- **Auto-fixable** (SwiftFormat can fix): trailing whitespace, indentation, formatting
- **Non-auto-fixable** (requires manual fix): force unwraps, implicitly unwrapped optionals

### Example Violations

```
warning: Force unwrap should be avoided
  Presentation/ViewModels/TimerListViewModel.swift:25:12

warning: Implicitly unwrapped optionals should be avoided
  Data/Network/APIClient.swift:8:10

error: Trailing whitespace
  Domain/Entities/Timer.swift:42:20
```

### Route

**Integration Phase** (code quality)

### Fix Process

1. **Integration runs SwiftLint**:
   ```bash
   swiftlint lint --strict
   ```

2. **Integration categorizes violations**:
   - Trailing whitespace, indentation, spacing → auto-fixable
   - Force unwraps, implicitly unwrapped optionals → non-auto-fixable

3. **Integration runs SwiftFormat on auto-fixable**:
   ```bash
   swiftformat . --indent 4
   ```

4. **Integration re-runs SwiftLint**:
   - All auto-fixable violations should be gone
   - Non-auto-fixable violations remain

5. **Integration handles remaining violations**:
   - If non-auto-fixable violations exist, Integration flags them in audit log
   - User will see them in PR review
   - User can approve anyway, or request Coder to manually fix

### Resolution

- **Success**: All auto-fixable violations fixed, non-auto-fixable flagged → proceed to Git
- **Failure** (unlikely): SwiftFormat error → escalate to user

### Max Retries

**1 attempt**. Formatting is idempotent. If SwiftFormat fixes violations, they stay fixed. If it can't fix them, they're flagged and proceed to human review in PR.

### Example Scenario

```
Integration runs SwiftLint
  violation: Force unwrap on line 25 of TimerListViewModel.swift
  violation: Trailing whitespace on line 42 of Timer.swift

Integration runs SwiftFormat
  Fixes trailing whitespace ✅
  Cannot auto-fix force unwrap (left as-is)

Integration re-runs SwiftLint
  violation: Force unwrap (flagged in audit log for PR review)
  All trailing whitespace fixed ✅

Proceed to Git with audit log showing 1 non-auto-fixable violation
```

---

## Loop 3: Layer Boundary Violation

### Trigger

Integration audit detects import rule violation:
- Presentation imports Data (should only import Domain)
- Domain imports UIKit/SwiftUI (should only import Foundation)
- Data imports Presentation (should only import Domain)

### Example Violations

```
error: Presentation/ViewModels/TimerListViewModel.swift imports Data
  "import Data" found on line 5
  Violation: Presentation must not import Data (use Domain protocols only)

error: Domain/Entities/Timer.swift imports SwiftUI
  "import SwiftUI" found on line 2
  Violation: Domain must only import Foundation
```

### Route

**Integration Phase** (layer separation audit)

### Fix Process

1. **Integration audit runs**:
   - Scan all Domain files for non-Foundation imports
   - Scan all Data files for non-Domain imports
   - Scan all Presentation files for Data imports

2. **Integration detects violation**:
   - Identifies file(s) with wrong imports
   - Reports import statement and why it violates rules

3. **Integration determines fix strategy**:
   - Is the import actually needed? Remove it.
   - Should ViewModel use a Repository directly? No, use a UseCase instead.
   - Should Domain use SwiftUI? No, refactor to pure struct.

4. **Integration makes fix**:
   ```swift
   // BEFORE: Presentation imports Data
   import Data  // ❌ Violation

   class TimerListViewModel {
       let repository: DefaultTimerRepository  // ❌ Direct Data import
   }

   // AFTER: Presentation uses Domain protocol
   import Domain  // ✅ Correct

   class TimerListViewModel {
       let useCase: GetTimersUseCase  // ✅ Uses Domain protocol
   }
   ```

5. **Integration re-audits**:
   - Runs layer separation check again
   - Verifies violation is fixed

### Resolution

- **Success**: Layer separation audit passes → proceed to Build
- **Failure**: Violation still exists after fix attempt → escalate with detailed explanation

### Max Retries

**2 attempts**. If Integration can't resolve boundary violation on second try, escalate to user with:
- Detailed audit report
- Which layer violates which rule
- Suggested architectural fix

### Example Scenario

```
Integration audit
  violation: Presentation/ViewModels/TimerListViewModel.swift imports Data

Integration analyzes:
  - ViewModel uses DefaultTimerRepository directly
  - Should use GetTimersUseCase from Domain instead

Integration fixes:
  - Changes import from "import Data" to "import Domain"
  - Changes repository: DefaultTimerRepository to useCase: GetTimersUseCase
  - Updates ViewModel to call useCase.execute() instead of repository.fetch()

Retry audit
  No violations ✅
  Proceed to Build
```

---

## Loop 4: Spec Unclear Mid-Coding

### Trigger

Architect phase (or any phase) encounters ambiguous requirement in spec:
- "What happens if timer goes negative?"
- "Should empty state show 'Create first timer' or 'No timers'?"
- "Is offline mode required or optional?"
- "Should pull-to-refresh show cached data while loading?"

### Route

**Coder (Architect phase)** → pauses → asks user for clarification

### Fix Process

1. **Architect identifies ambiguity**:
   - Reads spec and finds unclear requirement
   - Identifies what's missing

2. **Architect pauses Coder phase**:
   - Creates clarification question
   - Includes context (e.g., "Pull-to-refresh spec doesn't say how long refresh spinner shows")

3. **User responds**:
   - Provides clarification
   - Example: "Show spinner until data arrives, max 10 seconds timeout"

4. **Architect resumes with updated context**:
   - Incorporates clarification into design decisions
   - Continues with Domain Lead → Data Lead ∥ Presentation Lead → Integration

### Resolution

- **Success**: User provides clear clarification → Architect resumes and finishes
- **Timeout**: No user response after 10 minutes → flag as blocker and halt

### Max Retries

**Up to 3 clarifications per phase**. If Architect needs more than 3 clarifications, escalate to user with note that spec is too vague and should be revised before continuing.

### Example Scenario

```
Architect reads spec:
  "When user pulls down, show loading indicator"
  Question: "What happens if pull-to-refresh is already happening? Allow re-pull?"

Architect pauses Coder
Sends question to user

User responds:
  "Disable pull-to-refresh while loading is in progress"

Architect resumes:
  Updates navigation plan to show disabled state during load
  Continues to Domain phase
```

---

## Loop 5: PR Review Comments (From You)

### Trigger

You review the PR and leave comments requesting changes.

**Example comments**:
- "Change button color from blue to green"
- "The error message should say 'Failed to load timers. Please check your connection.'"
- "Add loading spinner to create timer button"
- "Fix typo: 'Timmers' should be 'Timers'"

### Route

**Git** → detects comment → **Coder (Integration + affected phase)**

### Fix Process

1. **You comment on PR**:
   ```
   "The button text says 'Start' but should say 'Start Timer' for clarity"
   ```

2. **Git detects comment**:
   - Parses user feedback
   - Determines scope (small change vs. major refactor)

3. **Coder analyzes scope**:
   - Is this a scoped fix? (text, color, spacing, single method)
     - Route to Integration or affected phase (Presentation for UI)
   - Is this an architectural change? (restructure layers, change navigation)
     - Reject and suggest new PR

4. **Coder makes targeted fix**:
   ```swift
   // BEFORE
   Button("Start") { ... }

   // AFTER
   Button("Start Timer") { ... }
   ```

5. **Build verifies fix**:
   ```bash
   xcodebuild build ...  # Must succeed
   ```

6. **Git amends commit**:
   ```bash
   git add Presentation/Views/TimerListView.swift
   git commit --amend --no-edit
   git push --force-with-lease
   ```
   PR updates automatically.

7. **You re-review amended commit**:
   - See updated code in PR
   - Approve or request more changes

### Resolution

- **Success**: Build passes, changes look good → you approve and merge
- **Build fails**: Fix broken something → Integration debugging → fix → retry
- **Architectural conflict**: Change conflicts with other code → reject, request new PR

### Max Retries

**Unlimited small changes**. Each comment-fix cycle can be repeated. However:
- If 5+ rounds of feedback, consider whether PR should have been designed better upfront
- If feedback requires architectural change, reject and request new PR

### Scope Guidelines

#### Approve amend for:
- Text/label changes
- Color changes
- Spacing/padding adjustments
- Single-method logic fixes
- Variable naming
- Comment/documentation updates

#### Reject and request new PR for:
- Adding new screens
- Removing screens
- Changing navigation flow
- Restructuring layers
- Major refactoring (rename many files, reorganize architecture)
- Changes affecting multiple screens or layers

### Example Scenario

```
PR created with TimerListView

You review and comment:
  "Error message should be red. Currently it's gray."

Coder reads comment
  Scope: Presentation layer only, single file
  Route: Integration + Presentation

Integration/Presentation fixes:
  Text("Error: \(message)")
    .foregroundColor(.gray)  // ❌
  →
  Text("Error: \(message)")
    .foregroundColor(.red)   // ✅

Build verifies:
  xcodebuild build ... ✓ Success

Git amends commit:
  git commit --amend

You re-review:
  Color is now red ✅
  Approve and merge
```

---

## Feedback Loop Priority & Interaction

When multiple loops trigger, they execute in this order:

1. **Build fails** → Integration fixes → Build retries
2. **Lint violations** → Integration fixes → Git (if build passes)
3. **Layer violations** → Integration fixes → Build retries
4. **Spec unclear** → Architect pauses, gets clarification (during Coder phase)
5. **PR comments** → Coder fixes → Build → Git amend (after PR created)

### Example Complex Scenario

```
Coder Phase 3: Data Lead writes APIClient
  Accidentally imports SwiftUI

Build runs
  ❌ Fail: Domain layer imports SwiftUI (not allowed)

Loop 1 activates: Build Fails
  Integration detects: "import SwiftUI" in Data/Network/APIClient.swift
  Integration fixes: Removes SwiftUI import
  Build retries
  ✅ Success

Integration runs SwiftLint
  ❌ Lint violation: Trailing whitespace on line 42

Loop 2 activates: Lint Violation
  Integration runs SwiftFormat
  ✅ Trailing whitespace fixed

Integration audit runs
  ✅ All layer boundaries correct
  ✅ All imports valid

Git creates PR

You review and comment:
  "APIClient should have timeout constant"

Loop 5 activates: PR Comments
  Coder adds timeout constant
  Build verifies
  Git amends commit

You approve and merge
```

---

## When Loops Escalate to User

A loop escalates (asks for user help) when:

1. **Build Loop**: 3 retries fail, error log shows unclear issue
2. **Lint Loop**: SwiftFormat fails (rare), or violation can't be auto-fixed and user rejects it in PR
3. **Layer Boundary Loop**: 2 retries fail, architectural redesign needed
4. **Spec Clarification Loop**: No user response after 10 minutes, or 3+ clarifications needed
5. **PR Comments Loop**: Change requires major refactoring (reject and ask for new PR)

**Escalation includes**:
- Clear explanation of what went wrong
- Detailed error log or audit report
- Suggested next steps
- Request for user input or decision

---

## See Also

- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-builder-lite/SKILL.md` — Overview of all feedback loops
- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-builder-lite/references/build-and-git.md` — Build and Git phase details
- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-builder-lite/references/code-gen-phases.md` — 4-phase Coder (where some loops trigger)
