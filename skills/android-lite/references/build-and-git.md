# Build & Git Workflow for Android Lite

## Build Phase

After Phase 4 (Integration) completes, the Coder runs the build to compile the Android app.

### Build Command

```bash
./gradlew assembleDebug
```

This command:
1. Compiles Kotlin source code to bytecode
2. Runs code generation tools (Hilt APT, Room, etc.)
3. Packages resources, manifests, and compiled code
4. Creates an APK file at `app/build/outputs/apk/debug/app-debug.apk`

### Build Process

**Step 1: Dependency Resolution**
```bash
./gradlew assemble --offline (if dependencies are cached)
```

**Step 2: Code Compilation**
- Kotlin compiler: source → bytecode
- Java compiler: Java → bytecode
- Annotation processors: Hilt, Room, Kotlin Serialization

**Step 3: Resource Processing**
- Compile resources
- Merge manifests
- Process vector drawables

**Step 4: APK Packaging**
- Combine bytecode, resources, assets
- Sign with debug key (automatically)
- Align resources (zipalign)
- Output: `app/build/outputs/apk/debug/app-debug.apk`

### Success Output

```
BUILD SUCCESSFUL in 45s
84 actionable tasks: 1 executed, 83 up-to-date
```

### Failure Handling

If the build fails, the error message indicates the root cause:

#### Example 1: Hilt Wiring Error
```
error: [Dagger/MissingBinding] java.lang.String cannot be provided without an @Provides-annotated method.
   public abstract RecipeService recipeService();
              ~~~~~~~~~~~~~~~~~
   It is injected at
       NetworkModule.provideRecipeService(...)
```

**Fix**: Add a @Provides method for String or correct the dependency.

#### Example 2: Type Mismatch
```
error: type mismatch: inferred type is RecipeRepositoryImpl but RecipeRepository was expected
   @Binds
   fun bindRecipeRepository(impl: RecipeRepositoryImpl): RecipeRepository
                                  ~~~~~~~~~~~~~~~~~
```

**Fix**: Ensure the implementation class matches the interface type.

#### Example 3: Duplicate Bindings
```
error: [Dagger/DuplicateBindings] RecipeRepository is bound multiple times
```

**Fix**: Remove duplicate @Binds or @Provides methods.

### Build Failure Recovery

When `./gradlew assembleDebug` fails:

1. **Log the error** in detail
2. **Route to Coder Integration** phase
3. **Identify the issue** (Hilt, type system, resource, etc.)
4. **Fix in the appropriate layer**:
   - Hilt wiring → di/ modules
   - Type errors → domain or data layer
   - Resource issues → res/ directory
   - Manifest issues → AndroidManifest.xml
5. **Retry**: `./gradlew clean && ./gradlew assembleDebug`
6. **Loop up to 2x** before halting for manual intervention

### Optional: Release Build Signing

For production release builds, configure signing:

```kotlin
// build.gradle.kts
android {
    signingConfigs {
        create("release") {
            storeFile = file("path/to/keystore.jks")
            storePassword = System.getenv("STORE_PASSWORD")
            keyAlias = System.getenv("KEY_ALIAS")
            keyPassword = System.getenv("KEY_PASSWORD")
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            signingConfig = signingConfigs.getByName("release")
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }
}
```

Build release APK:
```bash
./gradlew assembleRelease
```

---

## Git Workflow

After successful build, the Coder creates a Git commit and pull request.

### Branch Creation

Create a feature branch from main:

```bash
git checkout -b feature/[feature-name]
```

Examples:
- `feature/recipe-search`
- `feature/user-authentication`
- `feature/offline-caching`

**Branch naming convention**: `feature/`, `bugfix/`, `refactor/` prefixes

### Conventional Commits

All commits follow the conventional commit format:

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types**:
- `feat:` — New feature
- `fix:` — Bug fix
- `refactor:` — Code structure change (no feature change)
- `test:` — Add or modify tests
- `docs:` — Documentation changes
- `style:` — Code style (formatting, semicolons, etc.)
- `perf:` — Performance improvement
- `ci:` — CI/CD configuration changes

**Examples**:

```
feat(recipe-list): add search functionality to recipe list

- Implement search input field with Material3 styling
- Filter recipes by name, cuisine, ingredients
- Show empty state when no results found

Closes #42
```

```
fix(hilt): resolve missing RecipeService binding

- Add @Provides method for RecipeService in NetworkModule
- Configure Retrofit with correct baseUrl and converter factory

Fixes #38
```

```
refactor(domain): simplify recipe entity validation

- Move complex validation to separate Validator class
- Remove redundant checks in init block
```

### Staging & Committing

Stage specific files (avoid git add -A):

```bash
git add app/src/main/kotlin/com/example/feature/recipe/domain/
git add app/src/main/kotlin/com/example/feature/recipe/data/
git add app/src/main/kotlin/com/example/feature/recipe/presentation/
git add app/src/main/kotlin/com/example/di/
git add app/build.gradle.kts
```

Commit with message:

```bash
git commit -m "feat(recipe): implement recipe list with search

- Add RecipeListScreen with Material3 components
- Implement RecipeListViewModel with Hilt injection
- Create use case and repository layers with domain isolation
- Configure Retrofit and Room for data persistence

Closes #42"
```

### Pull Request Creation

Create a PR with title and description:

```bash
gh pr create --title "Add recipe search feature" --body "
## Summary
- Implement recipe search with filtering
- Add Material Design 3 components
- Configure Hilt dependency injection

## Changes
- RecipeListScreen and RecipeDetailScreen
- GetRecipesUseCase and SearchRecipesUseCase
- RecipeRepository with Retrofit + Room integration
- Hilt modules for DI setup

## Testing
- Unit tests for use cases and repository
- ViewModel state transitions tested
- Compose previews for screens

## Checklist
- [x] Code follows Kotlin standards
- [x] ktlint and detekt pass
- [x] Unit tests pass
- [x] APK builds successfully
- [x] No layer boundary violations
"
```

**PR title**: Short (< 70 chars), descriptive
**PR body**: Include summary, changes, testing approach, checklist

### Human Review Gate #2

The PR is now awaiting human review:

1. **Reviewer checks**:
   - Diff review: code quality, logic correctness
   - Commit history: conventional format, logical grouping
   - File structure: layer organization, naming
   - Architecture: layer boundaries, DI setup
   - Tests: coverage, adequacy
   - Edge cases: error handling, validation

2. **Possible actions**:
   - **Approve + Merge**: PR is merged to main
   - **Request changes**: Reviewer leaves comments; Coder addresses them (see feedback loop)
   - **Reject**: PR is closed; feature is discarded or rethought

### PR Feedback Loop

If the reviewer requests changes:

1. **Read feedback comments** on the PR
2. **Identify affected code** and phase
3. **Make targeted fixes**:
   - Logic fixes → Domain or Data phase
   - UI fixes → Presentation phase
   - Architecture fixes → Integration phase
4. **Run Build** to verify: `./gradlew assembleDebug`
5. **Commit changes**:
   ```bash
   git add .
   git commit -m "fix(recipe): address PR feedback on search validation"
   ```
6. **Push to branch**:
   ```bash
   git push origin feature/recipe-search
   ```
7. **Await next review** (or auto-dismiss old comments if fixes are obvious)

### Force Push (After Approval)

If clean history is needed (e.g., squash commits), force-push with caution:

```bash
git rebase -i HEAD~5  # Interactive rebase of last 5 commits
git push --force-with-lease origin feature/recipe-search
```

**Caution**: Only force-push if you're the sole contributor to the branch.

### Merge to Main

**Important**: The Lite Pipeline DOES NOT automatically merge to main. Only the human reviewer can merge.

Manual merge options:

```bash
# 1. Squash merge (all commits squashed into one)
git merge --squash feature/recipe-search

# 2. Regular merge (preserves commit history)
git merge --no-ff feature/recipe-search

# 3. Rebase merge (linear history)
git rebase feature/recipe-search
```

After merge:

```bash
git push origin main
git branch -d feature/recipe-search  # Delete local branch
git push origin --delete feature/recipe-search  # Delete remote branch
```

---

## Git Workflow Diagram

```
git checkout -b feature/[name]
           ↓
   Coder: Phase 1-4
           ↓
    Build: ./gradlew assembleDebug
           ↓
      Success? No → Fix → Build again
           ↓ Yes
  Git: add, commit, push
           ↓
gh pr create --title "..." --body "..."
           ↓
⏸ Review Gate #2: Human Review
           ├─ Approve & Merge → git merge feature/[name]
           ├─ Request Changes → Coder fixes → Build → Git commit/push
           └─ Reject → Close PR
```

---

## Troubleshooting

### Issue: Dirty Working Directory
```
fatal: your local changes to the following files would be overwritten by checkout
```

**Fix**:
```bash
git status  # See what changed
git add .   # Stage all changes
git commit -m "save work in progress"
```

### Issue: Merge Conflict
```
CONFLICT (content merge): Merge conflict in app/src/main/kotlin/...
```

**Fix**:
```bash
git merge --abort  # Or resolve conflicts
# Edit conflicted files, remove <<< >>> markers
git add [resolved-files]
git merge --continue
```

### Issue: Rebase Failed
```
error: could not apply [commit-hash]...
```

**Fix**:
```bash
git rebase --abort
# Or resolve conflicts and continue
git rebase --continue
```

### Issue: Force Push Not Allowed
```
remote: fatal: You've updated the history of this repository
```

**Fix**: Use `--force-with-lease` instead of `--force`:
```bash
git push --force-with-lease origin feature/[name]
```

---

## Summary

| Phase | Command | Output |
|-------|---------|--------|
| Build | `./gradlew assembleDebug` | `app/build/outputs/apk/debug/app-debug.apk` |
| Branch | `git checkout -b feature/[name]` | Feature branch created |
| Commit | `git commit -m "feat(...): ..."` | Conventional commit |
| Push | `git push origin feature/[name]` | Branch pushed to remote |
| PR | `gh pr create --title "..." --body "..."` | PR created on GitHub |
| Review | Human review & merge | Main branch updated |

