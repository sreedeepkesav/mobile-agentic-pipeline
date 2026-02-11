# Build & Git Phase

## Overview

After the Coder finishes all 4 phases, the pipeline enters Build and then Git phases.

- **Build**: Compiles the project with xcodebuild, verifies no errors
- **Git**: Creates feature branch, writes conventional commits, opens PR

**Duration**: 5-10 minutes total

---

## Build Phase

### Purpose

Verify that the Coder's output compiles without errors. If compilation fails, route back to Integration phase for fixes.

### Process

**1. Prepare Build Environment**
```bash
# Set environment variables (if needed)
export DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer

# Verify xcodebuild is available
xcodebuild -version
```

**2. Build the Project**
```bash
xcodebuild build \
  -project TimerApp.xcodeproj \
  -scheme TimerApp \
  -configuration Release \
  -derivedDataPath ./build \
  -quiet
```

**3. Check for Errors**
- Exit code 0: Build succeeded
- Exit code non-zero: Build failed

**Example error output**:
```
error: [Domain/Entities/Timer.swift:12] Use of undeclared identifier 'TimeInterval'
error: [Presentation/ViewModels/TimerListViewModel.swift:8] Protocol 'GetTimersUseCase' does not conform to 'ObservableObject'
```

### Build Failure Feedback Loop

**If build fails**:
1. Extract error details (file, line, message)
2. Route back to Integration phase
3. Integration analyzes error:
   - Missing import? Add it.
   - Type mismatch? Fix the type.
   - Protocol not conformed? Add conformance or fix implementation.
   - Circular dependency? Re-architect imports.
4. Integration makes targeted fix
5. Build retries

**Example Integration Fix**:
```swift
// BEFORE: Build error - "Use of undeclared identifier 'TimeInterval'"
import Foundation

struct Timer {
    let duration: TimeInterval  // ❌ TimeInterval not imported
}

// AFTER: Integration adds import
import Foundation

struct Timer {
    let duration: TimeInterval  // ✅ TimeInterval from Foundation
}
```

**Max retries**: 3 attempts. If build still fails after 3 retries, escalate to user with detailed error log.

### Build Success Output

**If build succeeds**:
```
Build complete!
Build time: 45 seconds
Warnings: 0
Errors: 0
Output: ./build/Release/TimerApp.app
```

---

## Git Phase

### Purpose

Create a clean git history with conventional commits and open a PR for human review.

### Process

**1. Create Feature Branch**

Branch naming: `feature/<feature-name>` or `fix/<issue-name>`

```bash
git checkout -b feature/timer-app
# or
git checkout -b feature/pull-to-refresh-timeline
```

**2. Stage All Generated Code**

```bash
git add App/ Domain/ Data/ Presentation/
git add .gitignore  # If applicable
```

**3. Write Conventional Commits**

Follow Conventional Commits format:
```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types**:
- `feat`: New feature
- `fix`: Bug fix
- `refactor`: Code refactoring (no behavior change)
- `docs`: Documentation
- `test`: Adding tests

**Scopes** (optional): Architecture layer or component (e.g., `domain`, `data`, `presentation`, `di`)

**Subject**: Imperative, lowercase, no period, max 50 chars

**Body** (optional): Explain what and why, not how

**Footer** (optional): Reference to issue (e.g., "Closes #123")

### Example Commits

#### Single-file feature:
```
git commit -m "feat(presentation): add pull-to-refresh to timer list"
```

#### Multi-part feature:
```
git commit -m "feat(domain): create Timer entity and use cases

Defines Timer struct with start/pause/reset methods.
Implements GetTimersUseCase, StartTimerUseCase, DeleteTimerUseCase.
Defines DomainError enum for error handling."

git commit -m "feat(data): implement TimerRepository and APIClient

Adds TimerDTO for API communication.
Adds TimerMapper for DTO ↔ Entity transformation.
Adds APIClient for network requests.
Implements DefaultTimerRepository conforming to Domain protocol."

git commit -m "feat(presentation): create TimerListView and ViewModel

Implements TimerListViewModel with ViewState enum.
Creates TimerListView with list rendering and action buttons.
Integrates TimerCoordinator for navigation to detail screen."

git commit -m "feat(app): wire dependency injection container

Creates DIContainer.make() factory.
Instantiates all repositories and use cases.
Sets up app entry point with coordinator and root view."
```

#### Refactoring example:
```
git commit -m "refactor(presentation): extract TimerRowView component

Separate TimerRowView into reusable component.
Improves TimerListView readability.
Enables reuse in other screens if needed."
```

#### Fix example:
```
git commit -m "fix(domain): correct timer countdown logic

Prevent negative remainingTime.
Stop timer when remainingTime reaches 0.
Add edge case handling for pause during countdown."
```

### 4. Create Pull Request

```bash
git push -u origin feature/timer-app
```

Then use GitHub CLI (or web interface):
```bash
gh pr create \
  --title "feat: Add timer feature" \
  --body "$(cat <<'EOF'
## Summary
Add complete timer feature with create, list, detail, and delete screens.

## Changes
- Domain: Timer entity, use cases, repository protocol
- Data: APIClient, DTOs, mappers, repository implementation
- Presentation: ViewModels, Views, Coordinator
- App: DI container and entry point

## Acceptance Criteria
- [x] List shows all user timers
- [x] Can create new timer with name and duration
- [x] Can start/pause/reset individual timers
- [x] Can delete timers
- [x] Error states display properly
- [x] Empty state shows helpful message
- [x] Loading state displays spinner

## Files Changed
- App/DI/Container.swift (created)
- App/main.swift (created)
- Domain/Entities/Timer.swift (created)
- Domain/UseCases/* (created)
- Domain/Repositories/TimerRepository.swift (created)
- Domain/Errors/DomainError.swift (created)
- Data/DTOs/TimerDTO.swift (created)
- Data/Mappers/TimerMapper.swift (created)
- Data/Repositories/DefaultTimerRepository.swift (created)
- Data/Network/APIClient.swift (created)
- Presentation/Views/* (created)
- Presentation/ViewModels/* (created)
- Presentation/Coordinators/TimerCoordinator.swift (created)

## Notes
- Follows MVVM-C + Clean Architecture
- Domain layer has zero framework imports
- All code passes SwiftLint and SwiftFormat
- No breaking changes
EOF
)" \
  --draft
```

**PR Description includes**:
- Summary of what was built
- Changes organized by layer
- Acceptance criteria checklist
- List of files created/modified
- Any breaking changes
- Notes about architecture decisions

---

## Review Gate 2: Human Merge

After PR is created, the pipeline pauses. **You** review the PR.

### What to Check

1. **Diff Review**
   - Read each changed/created file
   - Verify code follows standards
   - Check for logic errors

2. **Commit Messages**
   - Are they conventional? (feat:, fix:, refactor:)
   - Do they explain the why?
   - Are they logically grouped?

3. **File Structure**
   - Domain layer has no UIKit/SwiftUI imports?
   - Data layer only imports Domain + Foundation?
   - Presentation only imports Domain + SwiftUI?
   - DI container properly wires all dependencies?

4. **Code Quality**
   - SwiftLint violations: 0
   - Forced unwraps: 0 (except constants)
   - Proper naming conventions?
   - Consistent indentation?

5. **Architecture**
   - Clear layer separation?
   - Dependency injection working?
   - Navigation coordinator properly integrated?
   - ViewState enum covering all cases?

### Review Actions

#### Approve & Merge
```bash
gh pr merge <pr-number> --squash
# or --rebase or --create-commit depending on preference
```

#### Request Changes
Comment on specific lines or overall:
```
gh pr comment <pr-number> -b "Please update the color of the error message to red"
```

Then Coder fixes → Build → Git (amend) → you re-review

#### Reject (Close Without Merge)
```bash
gh pr close <pr-number>
```

Then provide detailed feedback and start new PR.

---

## Feedback Loop: PR Comments → Coder → Fix → Amend

If you request changes in PR comments, the pipeline handles it:

**1. You comment on PR**:
```
"The button text should be 'Start Timer' not 'Start' for clarity"
```

**2. Coder reads comment and identifies scope**:
- Small, scoped change (button text, color, spacing)
- Or architectural change (requires new PR)

**3. Coder makes targeted fix**:
```swift
// BEFORE
Button("Start") { ... }

// AFTER
Button("Start Timer") { ... }
```

**4. Build verifies fix**:
```bash
xcodebuild build ...  # Succeeds
```

**5. Git amends commit**:
```bash
git add Presentation/Views/TimerListView.swift
git commit --amend --no-edit
git push --force-with-lease
```

PR updates automatically with new commit.

**6. You re-review amended commit**:
```bash
gh pr view <pr-number> --web  # See updated diff
```

### When to Request Amend vs. Reject

**Request amend if**:
- Small, isolated change (text, color, spacing, logic fix)
- Change is within same file or layer
- No architectural rethink needed

**Reject and request new PR if**:
- Major refactoring (reorganize layers, rename many files)
- Architectural change (change from MVVM-C to something else)
- Change touches multiple layers and requires re-design

---

## Git Workflow Summary

```
After Coder Integration Phase
    ↓
Run Build (xcodebuild)
    ↓
Build fails?
    ├─ YES: Route to Integration, fix, retry Build
    └─ NO: Proceed to Git
    ↓
Create feature branch (feature/...)
    ↓
Write conventional commits (feat:, fix:, refactor:)
    ↓
Create PR with summary and checklist
    ↓
You review (diff, commits, files, lint, architecture)
    ↓
You approve/request changes/reject
    ↓
If changes requested:
    └─ Coder fixes → Build → Git amend → you re-review
    ↓
You merge PR manually
    ↓
Done! Feature is merged to main
```

---

## Troubleshooting

### Build Fails with Import Errors

**Symptom**: `Use of undeclared identifier 'UIView'` in Domain layer

**Root cause**: Accidental import of UIKit/SwiftUI in Domain

**Fix**: Integration removes the import and verifies domain uses only Foundation

### Build Succeeds but Lint Fails

**Symptom**: Build passes, but SwiftLint reports force unwraps in Presentation code

**Root cause**: Integration ran SwiftLint after build

**Fix**: Integration flags violation in Integration Audit Log. If auto-fixable, SwiftFormat fixes it. If not, flag for PR review.

### PR Has Merge Conflicts

**Symptom**: Merge button disabled, "Conflicts must be resolved"

**Root cause**: main branch changed since PR was created

**Fix**: User must manually resolve conflicts or ask Coder to rebase

### PR Has Too Many Commits

**Symptom**: Want to squash many small commits into one

**Fix**: User can squash-merge instead of regular merge
```bash
gh pr merge <pr-number> --squash
```

---

## See Also

- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-builder-lite/SKILL.md` — Full pipeline overview
- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-builder-lite/references/feedback-loops.md` — Feedback loops (including Build fail loop)
- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-builder-lite/references/ios-standards.md` — Standards that Build/Git verify
