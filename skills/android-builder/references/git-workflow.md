# Git Workflow

## Overview

The **Git Agent** orchestrates version control: committing changes, pushing branches, creating pull requests, and preparing for human review/merge. This workflow is platform-agnostic and consistent across all mobile pipeline variants (iOS, Android, Kotlin Multiplatform).

---

## Git Workflow Phases

### Phase 1: Feature Branch Preparation

**Goal:** Create a feature branch if one doesn't exist, prepare for commits.

**Steps:**

1. **Check Current Branch**
   ```bash
   git status
   git rev-parse --abbrev-ref HEAD
   ```
   Expected output: feature branch name (e.g., `feature/user-auth` or `feat/auth`)

2. **Determine Branch Name**
   - If new feature: Create branch `feature/<feature-name>`
   - If bug fix: Create branch `bugfix/<issue-number>-<description>`
   - If refactor: Create branch `refactor/<scope>`

   **Branch Naming Convention:**
   ```
   feature/<feature-description>        # New feature
   bugfix/<issue-number>-<description>  # Bug fix
   refactor/<scope>                     # Refactoring
   chore/<scope>                        # Non-functional changes
   hotfix/<issue-number>                # Production hotfix
   ```

3. **Create Branch (if needed)**
   ```bash
   git checkout -b feature/user-auth
   ```

4. **Ensure Branch Tracks Remote**
   ```bash
   git branch -u origin/feature/user-auth
   # Or on first push: git push -u origin feature/user-auth
   ```

---

### Phase 2: Stage Changes

**Goal:** Add all generated code, tests, lint results to staging area.

**Steps:**

1. **Check Working Directory**
   ```bash
   git status
   ```
   Output shows untracked files, modifications.

2. **Stage Specific Files (Preferred)**
   ```bash
   # Stage generated code
   git add src/main/kotlin/com/example/myapp/domain/
   git add src/main/kotlin/com/example/myapp/data/
   git add src/main/kotlin/com/example/myapp/presentation/
   git add src/test/kotlin/com/example/myapp/

   # Stage build config updates (if changed)
   git add build.gradle.kts

   # Stage Hilt modules
   git add src/main/kotlin/com/example/myapp/di/
   ```

   **Files to Stage:**
   - ✓ Kotlin source code (domain/, data/, presentation/, tests/)
   - ✓ build.gradle.kts (if dependencies added)
   - ✓ AndroidManifest.xml (if activities added)
   - ✓ Documentation (if design decisions documented)

   **Files to NOT Stage:**
   - ✗ `build/` (compiled artifacts)
   - ✗ `.gradle/` (gradle cache)
   - ✗ `*.apk` (APK artifacts) — optional, usually in .gitignore
   - ✗ `.env` files (secrets)
   - ✗ IDE config files (unless team standard)

3. **Verify Staging**
   ```bash
   git status
   git diff --cached  # Review staged changes
   ```

---

### Phase 3: Commit Changes

**Goal:** Create descriptive commit with clear message.

**Commit Message Format:**

Use conventional commits format for clarity and automation:

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**
- `feat:` New feature
- `fix:` Bug fix
- `refactor:` Code reorganization (no behavior change)
- `test:` Test additions/updates
- `docs:` Documentation
- `chore:` Build, dependencies, tooling
- `perf:` Performance improvements

**Example Commit Messages:**

```
feat(auth): add JWT login with offline caching

- Implemented LoginUseCase in domain layer (pure Kotlin)
- Added AuthRepository interface + AuthRepositoryImpl with Retrofit/Room
- Created LoginViewModel with UiState sealed class for state management
- Added LoginScreen and SplashScreen with Material Design 3
- Implemented Hilt DI (@HiltViewModel, @Binds, @Provides)
- Navigation: type-safe routes with NavHost
- Test coverage: 75% (domain, data, presentation)
- ktlint + detekt: zero violations

Closes #42
```

```
fix(login): fix StateFlow not emitting on screen rotation

Previously, LoginScreen.collectAsStateWithLifecycle() was not properly
handling lifecycle changes on rotation, causing state loss.

Changed to use collectAsStateWithLifecycle() which respects lifecycle
events and prevents memory leaks.

Test: LoginScreenTest verified state persists on rotation.

Fixes #156
```

```
test(user): add test coverage for GetUserUseCase

Added comprehensive unit tests for GetUserUseCase:
- Test happy path (user found)
- Test error scenarios (network error, invalid ID)
- Test repository mocking with MockK

Coverage increased: 65% → 82% for domain layer.
```

**Commit Message Template (Git Hook):**

Create `.git/hooks/commit-msg` for consistency:

```bash
#!/bin/bash

# Enforce conventional commit format
if ! grep -qE "^(feat|fix|refactor|test|docs|chore|perf)(\(.+\))?:" "$1"; then
    echo "❌ Commit message must start with: feat/fix/refactor/test/docs/chore/perf"
    exit 1
fi

echo "✓ Commit message format valid"
```

**Create Commit:**

```bash
git commit -m "feat(auth): add JWT login with offline caching

- Implemented LoginUseCase in domain layer (pure Kotlin)
- Added AuthRepository interface + AuthRepositoryImpl with Retrofit/Room
- Created LoginViewModel with UiState sealed class
- Added LoginScreen and SplashScreen with Material Design 3
- Implemented Hilt DI modules
- Navigation: type-safe routes with NavHost
- Test coverage: 75% (domain, data, presentation)
- ktlint + detekt: zero violations

Closes #42"
```

---

### Phase 4: Push to Remote

**Goal:** Push commits to remote branch, ensuring upstream tracking.

**Steps:**

1. **Initial Push (Create Remote Branch)**
   ```bash
   git push -u origin feature/user-auth
   ```
   The `-u` flag sets upstream tracking: subsequent pushes can use `git push` (no args).

2. **Subsequent Pushes**
   ```bash
   git push
   ```
   (If upstream already set)

3. **Force Push (Only if Rewriting History)**
   ```bash
   # DANGEROUS: only do this on feature branches, never main/develop
   git push --force-with-lease origin feature/user-auth
   ```
   Use `--force-with-lease` instead of `--force` to avoid overwriting others' work.

4. **Verify Push**
   ```bash
   git log -1 --oneline origin/feature/user-auth
   ```
   Expected: shows your latest commit on remote.

---

### Phase 5: Create/Update Pull Request

**Goal:** Open PR on GitHub/GitLab for human review.

**Using GitHub CLI (`gh`):**

```bash
gh pr create \
  --title "feat(auth): add JWT login with offline caching" \
  --body "## Summary
- Added JWT-based authentication with offline user caching
- Implemented Clean Architecture (Domain, Data, Presentation layers)
- Hilt dependency injection with proper module bindings
- Material Design 3 UI with Jetpack Compose

## Changes
- Domain: LoginUseCase, AuthRepository interface
- Data: AuthRepositoryImpl with Retrofit/Room, DTOs with @Serializable
- Presentation: LoginViewModel with UiState, LoginScreen + SplashScreen
- DI: Hilt modules (@Module, @Binds, @Provides, @Singleton)
- Navigation: type-safe routes with NavHost

## Test Plan
- [x] Domain: LoginUseCase tests (MockK)
- [x] Data: AuthRepository tests (Retrofit/Room mocks)
- [x] Presentation: LoginViewModel + UI tests (Compose Testing)
- [x] Coverage: 75% overall (target 60%+)

## Checklist
- [x] Layer boundaries verified (zero Android in domain/)
- [x] Hilt bindings complete (@Binds for repos)
- [x] Navigation routes in NavHost
- [x] Material Design 3 applied
- [x] All tests passing
- [x] ktlint formatted, detekt clean

Closes #42" \
  --draft false
```

**Using Manual GitHub Web UI:**

1. Visit https://github.com/YourOrg/YourRepo/
2. Click "New pull request"
3. Select base: `main` (or `develop`), compare: `feature/user-auth`
4. Fill PR title, body (from commit message + details)
5. Click "Create pull request"

**Using GitLab CLI (`glab`):**

```bash
glab mr create \
  --title "feat(auth): add JWT login with offline caching" \
  --description "[content same as GitHub PR body]"
```

**PR Template (Example):**

```markdown
## Summary
Brief description of what this PR accomplishes.

## Motivation
Why was this change needed?

## Changes
- Bullet point 1
- Bullet point 2
- Bullet point 3

## Test Plan
- [ ] Unit tests passing
- [ ] UI tests passing
- [ ] Manual testing on device
- [ ] Coverage > 70%

## Checklist
- [ ] Code follows style guidelines (ktlint, detekt)
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] No breaking changes

## Related Issues
Closes #42
```

---

### Phase 6: PR Status Management

**Goal:** Monitor PR status, address feedback, iterate until approved.

**Steps:**

1. **Monitor Checks**
   - GitHub/GitLab runs automated checks (CI/CD):
     - Build (./gradlew build)
     - Tests (./gradlew test)
     - Lint (ktlint, detekt)
   - Wait for ✓ All checks pass

2. **If Checks Fail**
   - Git Agent routes back to relevant team member
   - Fix issue locally
   - Commit + push new fix
   - Checks re-run automatically

3. **Handle Review Feedback**
   - Maintainer comments on PR
   - Git Agent (or human) reviews comments
   - Make requested changes locally
   - Commit + push
   - Comment on PR: "Ready for re-review"

4. **Example Feedback Loop**
   ```
   Maintainer: "No error handling in login. What if network fails?"
   ↓
   Git Agent (or Pres Lead): Review feedback
   ↓
   Add error handling: map network exceptions to UiState.Error
   ↓
   Commit: "fix(auth): add error handling for network failures"
   ↓
   Push new commit
   ↓
   Comment on PR: "✓ Error handling added, re-review at your convenience"
   ↓
   Maintainer approves
   ```

5. **Approval Signals**
   ```
   GitHub: Shows ✓ Approved (X reviewers approved)
   GitLab: Shows Approved status

   Next: Maintainer can merge
   ```

---

### Phase 7: Merge to Main

**Goal:** Merge approved PR into main branch.

**By Maintainer (Human):**

```bash
# Via GitHub UI: Click "Merge pull request"
# OR via CLI:
gh pr merge <PR_NUMBER> --squash
```

**Merge Strategies:**
- **Squash:** Combine all PR commits into one (cleaner history)
- **Rebase:** Reapply PR commits on top of main (linear history)
- **Merge commit:** Keep all commits, create merge commit (full history)

**Recommended:** Squash for feature branches, Rebase for release branches.

**Post-Merge:**
1. Branch is automatically deleted (if option enabled)
2. Pipeline Memory updates with learned patterns
3. PR closed, linked issues auto-close (if Closes #XX in message)

---

## Git Workflow Diagram

```
Feature Work (local)
  ↓
Code Gen Team generates code
  ↓
Test ‖ Lint ‖ Build (all pass)
  ↓
Git Phase 1: Feature branch (feature/user-auth)
  ↓
Git Phase 2: Stage all code changes
  ↓
Git Phase 3: Commit with descriptive message
  ↓
Git Phase 4: Push to remote
  ↓
Git Phase 5: Create PR with detailed description
  ↓
GitHub/GitLab: Run automated checks (build, test, lint)
  ↓
  ├─ ✓ All checks pass
  │  ↓
  │  Human maintainer reviews
  │  ↓
  │  ├─ ✓ Approves → Merge to main
  │  │
  │  └─ ✗ Requests changes
  │     ↓
  │     Git Agent (or team) fixes + push new commit
  │     ↓
  │     Checks re-run
  │     ↓
  │     [loop until approved]
  │
  └─ ✗ Checks fail
     ↓
     Route back to relevant agent/team member
     ↓
     Fix issue + push
     ↓
     [checks re-run]
  ↓
Merge to main (human does this)
  ↓
Branch deleted
  ↓
Pipeline Memory updates
```

---

## Best Practices

1. **Small, Focused Commits:** One feature per PR (easier review)
2. **Descriptive Messages:** Explain "why", not just "what"
3. **Link Issues:** Use "Closes #42" to auto-close related issues
4. **Atomic Commits:** Each commit should be independently compilable
5. **No Merge Conflicts:** Keep branch up-to-date with main if long-running
6. **Squash Before Merge:** Option to squash to keep history clean
7. **Semantic Versioning:** Tag releases properly (v1.0.0, v1.0.1, etc.)

---

## Troubleshooting

### Problem: Merge Conflict

```
git pull origin main
# Auto-merge failed: conflict in src/.../LoginViewModel.kt

→ Manual resolution required
```

**Fix:**
1. Open conflicted file
2. Identify conflict markers (<<<<<<, ======, >>>>>>)
3. Resolve by choosing correct version (or combining)
4. Stage file: `git add LoginViewModel.kt`
5. Commit: `git commit -m "resolve: merge conflict in LoginViewModel"`
6. Push: `git push`

### Problem: Accidental Force Push

```
git push --force origin feature/user-auth
# WARNING: This overwrote teammate's commit!
```

**Prevention:**
- Use `git push --force-with-lease` instead
- This checks remote hasn't changed before forcing

### Problem: PR from Wrong Branch

```
Opened PR: feature/bugfix → feature/user-auth
(Should have been feature/bugfix → main)
```

**Fix:**
1. Close incorrect PR
2. Re-open correct PR with proper base/compare branches

---

## Integration with CI/CD

After `git push`, CI/CD pipeline runs:

```
Code pushed to feature/user-auth
  ↓
GitHub Actions / GitLab CI triggered
  ↓
Build: ./gradlew build
Test: ./gradlew test
Lint: ./gradlew ktlintFormat && ./gradlew detekt
  ↓
  ├─ ✓ All pass → Status: PASSING
  │
  └─ ✗ Any fail → Status: FAILING
     ↓
     PR shows ✗ Required checks failed
     ↓
     Block merge until fixed
```

---

## Summary

Git Workflow ensures:
- ✓ Clear commit history (conventional commits)
- ✓ Tracked changes (branch per feature)
- ✓ Code review (PR with CI/CD checks)
- ✓ Collaboration (feedback → iterate → merge)
- ✓ Traceability (link to issues)
- ✓ Quality gates (all checks must pass)

**Result:** Clean, traceable, reviewed codebase ready for human maintainer approval and merge.
