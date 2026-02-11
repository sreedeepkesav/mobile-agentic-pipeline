# Feedback Loops for Android Lite Pipeline

The Android Lite Pipeline includes 5 self-healing feedback loops that automatically detect and fix common issues without human intervention.

---

## Loop 1: Build Failure Recovery

**Trigger**: `./gradlew assembleDebug` fails due to compilation, Hilt wiring, or type errors

**Goal**: Automatically fix root cause and retry build (up to 2 attempts)

### Scenario 1: Missing Hilt @Provides

**Error**:
```
error: [Dagger/MissingBinding] java.lang.String cannot be provided without an @Provides-annotated method.
   public abstract RecipeService recipeService();
```

**Root Cause**: NetworkModule does not provide String as a base URL, or RecipeService is not properly provided.

**Auto-Fix**:
1. AI detects Hilt compilation failure
2. Examines NetworkModule for missing @Provides
3. Adds or corrects the @Provides method:
   ```kotlin
   @Provides
   @Singleton
   fun provideRecipeService(retrofit: Retrofit): RecipeService =
       retrofit.create(RecipeService::class.java)
   ```
4. Retries: `./gradlew clean && ./gradlew assembleDebug`

**Success Output**:
```
BUILD SUCCESSFUL in 45s
```

### Scenario 2: Type Mismatch in ViewModel Injection

**Error**:
```
error: type mismatch: inferred type is GetRecipesUseCaseImpl but GetRecipesUseCase was expected
```

**Root Cause**: Repository or use case implementation doesn't implement the interface correctly.

**Auto-Fix**:
1. AI detects type mismatch in Integration phase
2. Traces back to affected class
3. Corrects the class declaration:
   ```kotlin
   class GetRecipesUseCase @Inject constructor(
       private val recipeRepository: RecipeRepository
   ) {  // ← Ensure interface implementation is clear
       suspend operator fun invoke(): Result<List<Recipe>> = ...
   }
   ```
4. Retries build

### Scenario 3: Duplicate Hilt Bindings

**Error**:
```
error: [Dagger/DuplicateBindings] RecipeRepository is bound multiple times:
   - RecipeRepositoryImpl (with @Binds)
   - RecipeRepositoryImpl (with @Provides)
```

**Root Cause**: Both @Binds and @Provides exist for same interface.

**Auto-Fix**:
1. AI detects duplicate binding error
2. Examines RepositoryModule and other modules
3. Removes the duplicate (keeps @Binds as preferred):
   ```kotlin
   // Remove @Provides version, keep @Binds
   @Module
   interface RepositoryModule {
       @Binds
       @Singleton
       fun bindRecipeRepository(impl: RecipeRepositoryImpl): RecipeRepository
   }
   ```
4. Retries build

### Failure Escalation

If build fails after 2 retry attempts:
- **Log failure details** with timestamps
- **Halt pipeline** and request manual intervention
- **Output**: Comprehensive error log + suggestion for fix

---

## Loop 2: Lint Violation Auto-Fix

**Trigger**: `./gradlew ktlint` or `./gradlew detekt` finds violations after Phase 4

**Goal**: Auto-fix style violations without manual intervention

### Scenario: ktlint Formatting Issues

**Error**:
```
ktlint: line too long (121 > 120) [max-line-length]
   val recipeListViewModelFactory = remember { RecipeListViewModelFactory(getRecipesUseCase, searchRecipesUseCase) }
```

**Auto-Fix**:
1. AI runs `./gradlew ktlintFormat` to auto-fix
   ```bash
   ./gradlew ktlintFormat
   ```
   Output:
   ```
   val recipeListViewModelFactory = remember {
       RecipeListViewModelFactory(
           getRecipesUseCase,
           searchRecipesUseCase
       )
   }
   ```
2. Re-run linter to verify:
   ```bash
   ./gradlew ktlint
   ```
3. If passes: continue to Build phase
4. If fails: generate report and request manual review

### Common Auto-Fixable Issues

| Issue | Auto-Fix |
|-------|----------|
| Trailing whitespace | Remove trailing spaces |
| Unused imports | Delete import statements |
| Inconsistent indentation | Reformat with 4 spaces |
| Missing spaces around operators | Add spaces: `a+b` → `a + b` |
| Wrong quote style | Convert single to double (or vice versa) |

### Non-Auto-Fixable Violations

If `ktlintFormat` can't auto-fix, AI manually corrects:

**Example**: Unused variable
```kotlin
// ✗ Bad: detekt finds unused variable
val unused = viewModel.uiState.collectAsState()

// ✓ Fixed: either use it or remove
val uiState by viewModel.uiState.collectAsState()
```

---

## Loop 3: Layer Boundary Violation

**Trigger**: Architect or Integration phase detects cross-layer imports or illegal dependencies

**Goal**: Isolate violation, re-architect if needed, re-implement affected layer

### Scenario 1: Domain Imports Android

**Violation**:
```kotlin
// ✗ Bad: domain/entity/Recipe.kt
import android.content.Context  // ← Domain has Android import!
```

**Detection**:
1. Phase 4 (Integration) runs layer audit
2. Scans domain/ for Android imports
3. Finds `import android.content.Context`

**Recovery**:
1. Rolls back Phase 2 (Domain)
2. Re-runs Phase 1 (Architect) to clarify domain responsibilities
3. Re-implements Phase 2 without Android imports
4. Re-runs Phase 4 (Integration) audit
5. Retries Build

### Scenario 2: Presentation References Data DTOs

**Violation**:
```kotlin
// ✗ Bad: presentation/screen/RecipeListScreen.kt
import com.example.feature.recipe.data.remote.dto.RecipeDto  // ← Should use Recipe!

@Composable
fun RecipeListScreen() {
    val recipes: List<RecipeDto> = viewModel.getRecipes()  // ← Wrong!
}
```

**Detection**:
1. Phase 4 scans presentation/ for data/ imports
2. Finds `import *.data.remote.dto.RecipeDto`

**Recovery**:
1. AI corrects the import and type:
   ```kotlin
   // ✓ Fixed: presentation/screen/RecipeListScreen.kt
   import com.example.feature.recipe.domain.entity.Recipe

   @Composable
   fun RecipeListScreen(viewModel: RecipeListViewModel = hiltViewModel()) {
       val uiState by viewModel.uiState.collectAsState()
       val recipes: List<Recipe> = (uiState as? RecipeListUiState.Success)?.recipes ?: emptyList()
   }
   ```
2. Retries Build

### Scenario 3: Data References Presentation

**Violation**:
```kotlin
// ✗ Bad: data/repository/RecipeRepositoryImpl.kt
import com.example.feature.recipe.presentation.viewmodel.RecipeListViewModel  // ← No!
```

**Recovery**:
1. Remove the illegal import
2. Ensure data layer only depends on domain
3. Retries Build

---

## Loop 4: Spec Ambiguity During Coding

**Trigger**: Coder encounters unclear acceptance criteria or missing requirement during Phase 2-4

**Goal**: Pause development, clarify with Product + Design, resume from paused phase

### Scenario: Unclear Edge Case Handling

**Situation**:
- Phase 3 (Presentation Lead) is writing RecipeDetailScreen
- Spec says "Show recipe details or error message"
- But spec doesn't define what happens if recipe is deleted while viewing it

**Process**:
1. Presentation Lead pauses with question: "What to display if recipe is deleted?"
2. AI routes back to Product + Design Agent
3. Product + Design clarifies in edge-cases.md:
   ```markdown
   ## Recipe Deleted After Load
   - Display error message: "Recipe is no longer available"
   - Show button: "Return to list"
   - Do NOT crash the app
   ```
4. Product + Design returns updated spec
5. Presentation Lead resumes from Phase 3 with clarified requirements
6. Implements the fix:
   ```kotlin
   is RecipeDetailUiState.Error -> {
       Column(...) {
           Text("Recipe is no longer available")
           Button(onClick = onBack) { Text("Return to list") }
       }
   }
   ```
7. Phase 4 (Integration) resumes
8. Build succeeds

---

## Loop 5: PR Feedback Comments

**Trigger**: Human reviewer leaves comments on PR requesting changes

**Goal**: AI reads feedback, makes targeted fixes, re-builds, re-commits

### Scenario: Reviewer Comments on Error Handling

**PR Comment**:
```
@@ -45,5 +45,8 @@
     is RecipeListUiState.Error -> {
-        Text((uiState as RecipeListUiState.Error).message)
+        Column(horizontalAlignment = Alignment.CenterHorizontally) {
+            Text((uiState as RecipeListUiState.Error).message)
+            Button(onClick = viewModel::onRetry) { Text("Retry") }
+        }
     }
```

**Reviewer suggests**: "Add retry button to error message"

**Process**:
1. AI reads comment
2. Identifies affected file: `presentation/screen/RecipeListScreen.kt`
3. Locates error handling code
4. Implements suggestion:
   ```kotlin
   is RecipeListUiState.Error -> {
       Column(
           modifier = Modifier
               .fillMaxSize()
               .wrapContentSize(Alignment.Center),
           horizontalAlignment = Alignment.CenterHorizontally,
           verticalArrangement = Arrangement.spacedBy(16.dp)
       ) {
           Text(
               text = (uiState as RecipeListUiState.Error).message,
               style = MaterialTheme.typography.bodyLarge
           )
           Button(onClick = viewModel::onRetry) {
               Text("Retry")
           }
       }
   }
   ```
5. Runs Build to verify: `./gradlew assembleDebug`
   ```
   BUILD SUCCESSFUL
   ```
6. Stages and commits:
   ```bash
   git add app/src/main/kotlin/com/example/feature/recipe/presentation/screen/RecipeListScreen.kt
   git commit -m "fix(recipe): add retry button to error message

   - Improve UX by allowing users to retry failed recipe loads
   - Wrap error message and button in centered column
   - Address PR feedback
   ```
7. Pushes:
   ```bash
   git push origin feature/recipe-search
   ```
8. Awaits next review (auto-resolves comment or requests additional changes)

### Multiple Feedback Comments

If reviewer leaves multiple comments:
1. AI processes each comment independently
2. Groups changes by file
3. Makes all fixes in a single commit or multiple commits (if logically separate)
4. Re-builds once
5. Re-pushes once

**Example**: 3 comments addressing:
- Retry button on error (Presentation)
- Missing unit test for SearchRecipesUseCase (Domain)
- Typo in data model (Domain)

**Actions**:
```bash
# Fix 1: Add retry button
git add presentation/screen/RecipeListScreen.kt

# Fix 2: Add unit test
git add domain/usecase/SearchRecipesUseCaseTest.kt

# Fix 3: Fix typo
git add domain/entity/Recipe.kt

# Single build
./gradlew assembleDebug  # BUILD SUCCESSFUL

# Single commit
git commit -m "fix: address PR feedback

- Add retry button to error message (presentation)
- Add unit test for SearchRecipesUseCase (domain)
- Fix typo in Recipe entity documentation (domain)"

# Single push
git push origin feature/recipe-search
```

---

## Feedback Loop Orchestration

All 5 loops coordinate seamlessly:

```
Phase 1-4: Coder Development
    ↓
Build Phase
    ├─ Failure → Loop 1 (Build Recovery) → retry
    └─ Success ↓
ktlint / detekt
    ├─ Violations → Loop 2 (Lint Auto-Fix) → re-lint → ok
    └─ Clean ↓
Layer Audit
    ├─ Violation → Loop 3 (Layer Boundary) → re-architect & fix → retry
    └─ Clean ↓
Spec Verification
    ├─ Ambiguity → Loop 4 (Spec Clarification) → resume from phase
    └─ Clear ↓
Git & PR
    ↓
Human Review
    ├─ Feedback → Loop 5 (PR Comments) → fix → build → push → await review
    ├─ Approve → Merge to main
    └─ Reject → Close PR
```

---

## Loop Limits & Escalation

Each loop has limits to prevent infinite cycles:

| Loop | Max Attempts | Escalation |
|------|-------------|------------|
| Loop 1: Build Failure | 2 | Halt + request manual fix |
| Loop 2: Lint Violation | Auto-fix + 1 manual review | Halt + detailed report |
| Loop 3: Layer Boundary | 1 re-architect | Halt + architecture review |
| Loop 4: Spec Ambiguity | 1 clarification | Halt + request explicit AC |
| Loop 5: PR Comments | Unlimited | Keep cycling until approved |

---

## Summary

**Loop 1 (Build Recovery)**: Automatically fixes Hilt wiring, type mismatches, duplicate bindings

**Loop 2 (Lint Auto-Fix)**: Auto-formats code with ktlintFormat, removes unused imports

**Loop 3 (Layer Boundary)**: Detects cross-layer violations, re-architects, re-implements

**Loop 4 (Spec Clarification)**: Pauses development for unclear specs, resumes after clarification

**Loop 5 (PR Comments)**: Reads reviewer feedback, makes targeted fixes, re-builds, re-commits

All loops work together to ensure high-quality, standards-compliant code without manual rework.
