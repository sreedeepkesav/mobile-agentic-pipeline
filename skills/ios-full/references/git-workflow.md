# Git Workflow Reference

This document defines the Git workflow, branching strategy, commit conventions, and merge procedures for the ios-full-pipeline.

## Git Agent Overview

The Git Agent manages version control operations and GitHub interactions for the ios-full-pipeline.

**Responsibilities:**
- Create feature/bugfix branches
- Commit code with conventional commits
- Generate pull requests with automated descriptions
- Track review feedback and iterate
- Coordinate merges after manual review

**Configuration Options:**
- `git_strategy`: always (Git Agent always active)
- `git_auto_merge`: true | false (default: false)
- `conventional_commits`: true | false (default: true)
- `require_pr_review`: true | false (default: true)

**Important:** Git Agent does NOT merge automatically. Developers review and merge manually after code quality gates pass.

## Branch Strategy

All feature development follows a standardized branching model.

### Feature Branches

**Naming Convention:** `feature/[task-id]-[short-description]`

**Examples:**
```
feature/ios-123-user-authentication
feature/ios-456-dark-mode-support
feature/ios-789-payment-integration
```

**Branch Lifecycle:**
```
main
│
└── feature/ios-123-auth (create branch)
    │
    ├── commit: feat(auth): add login screen
    ├── commit: feat(auth): implement login use case
    ├── commit: test(auth): add login flow tests
    │
    └── → Pull Request (review gate)
        │
        ├── Review feedback received
        ├── commit: fix(auth): handle edge cases
        ├── commit: test(auth): add missing tests
        │
        └── Approved & merged to main
```

**When to Create Feature Branch:**
- New feature implementation
- User interface updates
- API integration
- Complex bug fixes requiring multiple commits

### Bugfix Branches

**Naming Convention:** `fix/[task-id]-[short-description]`

**Examples:**
```
fix/ios-234-login-crash
fix/ios-567-memory-leak
fix/ios-890-ui-rendering-bug
```

**Branch Lifecycle:**
```
main (v1.2.3)
│
└── fix/ios-234-login-crash (hotfix)
    │
    ├── commit: fix(auth): prevent crash on invalid input
    ├── commit: test(auth): add regression test
    │
    └── Pull Request (fast-track review for critical bugs)
```

**When to Create Bugfix Branch:**
- Defect resolution
- Performance optimization
- Security patching
- Data handling corrections

### Refactor Branches

**Naming Convention:** `refactor/[short-description]`

**Examples:**
```
refactor/extract-common-mappers
refactor/improve-error-handling
refactor/consolidate-viewmodels
```

**Branch Lifecycle:**
```
main
│
└── refactor/extract-common-mappers
    │
    ├── commit: refactor: extract UserMapper to shared
    ├── commit: refactor: update imports across codebase
    │
    └── Pull Request (architectural review)
```

**When to Create Refactor Branch:**
- Code structure improvements
- Mapper/utility consolidation
- Architecture improvements
- Code duplication reduction

### Release Branches

**Naming Convention:** `release/[version]`

**Examples:**
```
release/1.0.0
release/1.1.0
release/2.0.0-beta
```

**Branch Lifecycle:**
```
main (development branch)
│
└── release/1.0.0 (created at release time)
    │
    ├── commit: chore(release): bump version to 1.0.0
    ├── commit: docs(release): update changelog
    │
    └── Tagged as v1.0.0
        │
        └── Merged back to main
```

**When to Create Release Branch:**
- Preparing version release
- Feature freeze before release
- Stabilization and final testing
- Version bumping

## Conventional Commits

All commits must follow the Conventional Commits specification for clear, semantic commit history.

### Commit Format

```
type(scope): subject

body (optional)

footer (optional)
```

### Commit Types

**feat:** New feature
```
feat(auth): add two-factor authentication

Add support for 2FA using Time-based One-Time Password (TOTP).
Integrates with standard authenticator apps.
```

**fix:** Bug fix
```
fix(profile): prevent crash when loading missing image

Added null check before accessing profile image URL.
Resolves issue where missing image causes ViewController crash.
```

**refactor:** Code restructuring without behavioral change
```
refactor(mappers): extract common mapping logic

Consolidate UserMapper and AccountMapper into shared UserMapper.
Reduces code duplication by 30 lines.
```

**test:** Adding or updating tests
```
test(auth): add login state transition tests

Add unit tests for LoginViewModel state transitions.
Covers success, error, and loading states.
Coverage: 85% → 92%
```

**chore:** Build, dependency updates, configuration
```
chore(deps): update Domain layer imports

Update import statements across Presentation layer
to use new Domain module paths.
```

**docs:** Documentation changes
```
docs(architecture): update MVVM-C pattern guide

Clarify coordinator navigation patterns with examples.
Add deep linking section.
```

**style:** Formatting, semicolons, missing semicolons, etc.
```
style(formatting): run SwiftFormat on User module

No behavioral changes, formatting consistency only.
```

### Scope

The scope specifies which module/feature the change affects:

**Common Scopes:**
```
auth        # Authentication feature
profile     # User profile feature
product     # Product catalog feature
mappers     # Data mappers
repositories  # Repository implementations
navigation  # Coordinator navigation
api         # API client
persistence # CoreData/storage
```

### Subject Line

- Imperative mood ("add" not "adds" or "added")
- Don't capitalize first letter
- No period at end
- Maximum 50 characters
- Clear and descriptive

### Body (Optional)

- Explain WHAT and WHY, not HOW
- Wrap at 72 characters
- Separate from subject with blank line
- Reference issue numbers: "Resolves #123"

### Footer (Optional)

Breaking changes:
```
feat(auth)!: change authentication token storage format

BREAKING CHANGE: Token storage format changed from UserDefaults to Keychain
Requires users to re-authenticate after update.
```

## Commit Examples

**Example 1: New Feature**
```
feat(home): implement product list screen

Create HomeView with SwiftUI list displaying products.
Implement HomeViewModel with FetchProductsUseCase.
Add pagination support with load more button.

- ProductListView with product cards
- HomeViewModel with async product loading
- ProductCoordinator for navigation
- Unit tests for ViewModel state transitions

Resolves #ios-456
```

**Example 2: Bug Fix**
```
fix(profile): prevent crash on empty user data

Added defensive null checks before accessing optional properties.
Handle missing profile image URL gracefully.

- Check for nil email before display
- Provide default image if URL missing
- Add error state for incomplete user data

Fixes crash reported in production (v1.2.1)
```

**Example 3: Test Addition**
```
test(auth): add comprehensive login flow tests

Add unit tests for LoginViewModel covering all state transitions.
Add UI tests for authentication flow end-to-end.

Coverage improvements:
- LoginViewModel: 45% → 92%
- AuthRepository: 78% → 95%

Resolves code review feedback on test coverage.
```

**Example 4: Refactoring**
```
refactor(mappers): consolidate user mapping logic

Extract common mapping patterns from UserMapper and ProfileMapper.
Create UserTransformationHelper for reusable conversions.

Changes:
- New: UserTransformationHelper with shared logic
- Updated: UserMapper and ProfileMapper use shared logic
- Removed: 45 lines of duplicated code

No behavioral changes, refactoring only.
```

## Pull Request Workflow

### Creating a Pull Request

The Git Agent automatically creates PRs after code quality gates pass.

**PR Creation Trigger:**
```
Task assigned → Code generation → Test pass → Lint pass → Build pass → PR created
```

**Example Generated PR:**

```markdown
# Title
feat(auth): implement login screen with validation

## Summary
Implement complete login authentication flow with email/password validation.
Add LoginView with SwiftUI components and LoginViewModel for state management.
Integrate with LoginUseCase for authentication business logic.

## Changes
- Presentation/Features/Auth/Screens/LoginView.swift (new)
- Presentation/Features/Auth/ViewModels/LoginViewModel.swift (new)
- Domain/UseCases/LoginUseCase.swift (new)
- Data/Repositories/AuthRepositoryImpl.swift (updated)
- Tests: 12 new unit tests, 3 UI tests (new)

## Testing
- [x] Unit tests pass (12/12)
- [x] UI tests pass (3/3)
- [x] Linting passed
- [x] Build successful
- [x] Manual testing completed

## Screenshots
[Screenshot of login screen]

🤖 Generated with ios-full-pipeline
```

### PR Template

**File: .github/pull_request_template.md**

```markdown
## Summary
[What this PR does in 1-2 sentences]

## Related Issues
Resolves #[issue number]
Related to #[issue number]

## Changes
- [File-by-file or feature-by-feature changes]
- Describe each change clearly
- Include new files and modifications

## Architecture
- [ ] Follows MVVM-C pattern
- [ ] Respects layer boundaries
- [ ] Uses dependency injection
- [ ] No forced unwraps

## Testing
- [x] Unit tests pass
- [x] UI tests pass
- [x] Manual testing completed
- [x] Added tests for new code

## Code Quality
- [x] SwiftLint passed
- [x] SwiftFormat applied
- [x] No compiler warnings
- [x] Code reviewed by [reviewer]

## Screenshots (if UI changes)
[Attach screenshots of before/after]

## Deployment Notes
- Breaking changes: [describe if any]
- Database migrations: [if applicable]
- Config changes: [if applicable]
```

## Manual Review Gate

**Default Configuration:**
```yaml
git_auto_merge: false
require_pr_review: true
```

### Review Process

1. **Developer Opens PR**
   - Git Agent creates PR with auto-generated description
   - All quality gates must pass (Test, Lint, Build)
   - PR shows commit history and file changes

2. **Code Review**
   - Reviewer examines:
     - Code changes and logic
     - Test coverage
     - Architecture adherence
     - Naming and style consistency
   - Reviewer can:
     - Approve PR
     - Request changes
     - Add comments and suggestions

3. **Review Feedback Loop**
   ```
   PR Created
   │
   ├─ Reviewer requests changes
   │  ├─ Git → Coordinator
   │  ├─ Coordinator → Code Gen (specific layer)
   │  ├─ Code Gen fixes issues
   │  ├─ Test ‖ Lint ‖ Build executed
   │  ├─ New commits pushed to branch
   │  └─ Reviewer notified of changes
   │
   ├─ Reviewer approves
   │  └─ Developer merges manually
   │
   └─ Merge to main
   ```

4. **Manual Merge**
   - Developer clicks "Merge" button on GitHub
   - Branch deletion offered after merge
   - Automatic branch cleanup (optional)

### Reviewer Checklist

**Code Logic:**
- [ ] Code logic is correct and handles edge cases
- [ ] No unnecessary complexity
- [ ] Appropriate error handling
- [ ] No hardcoded values or magic numbers

**Architecture:**
- [ ] Follows MVVM-C pattern
- [ ] Respects layer boundaries
- [ ] No circular dependencies
- [ ] Appropriate use of protocols

**Testing:**
- [ ] Tests cover new functionality
- [ ] Tests have meaningful assertions
- [ ] Edge cases tested
- [ ] Mock objects properly configured

**Code Quality:**
- [ ] Naming follows Swift conventions
- [ ] No code duplication
- [ ] Appropriate use of language features
- [ ] Documentation/comments where needed

**Performance:**
- [ ] No obvious performance issues
- [ ] Appropriate data structures
- [ ] No memory leaks (autoreleasepool, strong ref cycles)
- [ ] Efficient algorithms

## Sprint Batch Git Workflow

When multiple tasks are assigned in a sprint batch, Git Agent creates parallel PRs.

**Parallel PR Creation:**

```
Sprint Batch Start (5 tasks)
│
├─ Task 1: feature/ios-100-auth → PR-1001 (independent)
├─ Task 2: feature/ios-101-profile → PR-1002 (independent)
├─ Task 3: feature/ios-102-settings → PR-1003 (depends on Task 1)
├─ Task 4: fix/ios-103-crash → PR-1004 (independent)
└─ Task 5: refactor/ios-104-mappers → PR-1005 (independent)

All PRs created in parallel (Tasks 1, 2, 4, 5)
PR-1003 waits for PR-1001 merge
```

**Merge Order for Dependent Tasks:**

```
Dependency Graph:
Task 1 (auth) → Task 3 (profile)
Task 2, 4, 5 (independent)

Merge Sequence:
1. Merge PR-1001 (auth) → rebase PR-1003
2. Merge PR-1002 (profile)
3. Merge PR-1003 (settings, updated from main)
4. Merge PR-1004 (fix)
5. Merge PR-1005 (refactor)

All merged to main in order
```

## Rebasing and Conflict Resolution

**When Rebasing Needed:**
- Branch is behind main by multiple commits
- Base branch (main) receives urgent updates
- Dependencies resolved by upstream merge

**Rebase Command:**
```bash
git fetch origin
git rebase origin/main
git push origin [branch-name] --force-with-lease
```

**Conflict Resolution:**
```bash
# Identify conflicts
git status

# Edit conflicted files, resolve markers
# <<<<<<<
# Current branch changes
# =======
# Main branch changes
# >>>>>>>

# Mark resolved
git add [resolved-file]

# Continue rebase
git rebase --continue

# Force push (with lease for safety)
git push origin [branch-name] --force-with-lease
```

## Commit History Best Practices

**Good Commit History:**
```
feat(auth): add login screen
feat(auth): implement login use case
test(auth): add login flow tests
fix(auth): handle invalid credentials edge case
refactor(auth): extract validation logic
```

**Avoid:**
- WIP commits left in PR history
- Commit messages like "fix", "update", "change"
- Excessive commits for single feature (squash into logical commits)
- Unrelated changes in single commit (split into multiple)

**Cleaning Up Commit History:**

Interactive rebase to squash related commits:
```bash
git rebase -i HEAD~5
# Change 'pick' to 'squash' for commits to combine
# Update commit message in editor
```

## Version Tagging

Tags mark releases and versions in the repository.

**Tag Naming Convention:** `v[major].[minor].[patch]`

**Examples:**
```
v1.0.0      # Initial release
v1.0.1      # Patch release
v1.1.0      # Minor release
v2.0.0      # Major release
v1.0.0-beta # Beta version
```

**Creating Tags:**
```bash
git tag v1.0.0 -a -m "Release version 1.0.0"
git push origin v1.0.0
```

**Tag Workflow:**
```
release/1.0.0 branch
│
├─ Stabilization and final testing
├─ Version bumped in code
├─ Changelog updated
│
└─ Tag created: v1.0.0
   │
   └─ Merged to main
      └─ Release builds created
```

## GitHub Configuration

**Protection Rules for Main Branch:**

```yaml
# Settings > Branches > Branch protection rules

main:
  require_pull_request_reviews: true
  dismiss_stale_pull_request_approvals: true
  require_code_owner_reviews: false
  require_status_checks_to_pass: true
  status_checks:
    - Test Suite (iOS)
    - Lint Check (Swift)
    - Build (iOS Release)
  require_branches_to_be_up_to_date_before_merging: true
  require_conversation_resolution_before_merging: true
  allow_force_pushes: false
  allow_deletions: false
```

**Required Status Checks:**
- All tests pass
- Lint checks pass
- Build succeeds
- Code review approval

**Enforcement:**
- Enforce for administrators: yes (no bypass)
- Allow force pushes: no
- Dismiss stale reviews: yes (on new commits)

## Git Workflow Summary

```
Development Process:
1. Create feature branch from main
2. Code generation and implementation
3. Commit with conventional commits
4. Test → Lint → Build (quality gates)
5. Create PR (Git Agent)
6. Code review (manual)
7. Address feedback (fix and re-test)
8. Merge to main (developer action)
9. Delete feature branch
10. Tag release (if applicable)

Timeline:
Initial code → Quality gates pass: ~15-30 min
Code review: ~24 hours (typical)
Merge to main: ~1 day after approval
Release (optional): ~2-3 days after merge
```
