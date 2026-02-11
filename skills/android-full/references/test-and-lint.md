# Test and Lint

## Overview

Testing and Linting happen **in parallel** after Code Gen Phase 3 (Integration). They are performed by two specialized agents:
- **Test Agent:** Generates and runs JUnit5 + MockK + Espresso tests
- **Lint Agent:** Runs ktlint (formatting) + detekt (static analysis)

---

## Test Agent

### Testing Framework

- **Unit Tests:** JUnit 5 (Jupiter)
- **Mocking:** MockK (Kotlin-native mocking)
- **UI Tests (Espresso):** Android Espresso + Compose Testing
- **Test Execution:** `./gradlew test` (unit), `./gradlew connectedAndroidTest` (instrumented)

### Layer-Aware Testing Strategy

Tests are structured differently per architectural layer to maximize value and minimize overhead.

#### Domain Layer Tests (80%+ Coverage Target)

**Goal:** Pure Kotlin logic, zero Android dependencies.

**Tools:** JUnit5 + MockK (no Android mocks)

**Test Structure:**
```kotlin
class GetUserUseCaseTest {
    private val mockRepository = mockk<UserRepository>()
    private val useCase = GetUserUseCase(mockRepository)

    @Test
    fun `invoke returns user on success`() = runTest {
        // Arrange
        val user = User("1", "john@example.com", "John")
        coEvery { mockRepository.getUser("1") } returns Result.success(user)

        // Act
        val result = useCase("1")

        // Assert
        assertTrue(result.isSuccess)
        assertEquals(user, result.getOrNull())
    }

    @Test
    fun `invoke returns failure on error`() = runTest {
        // Arrange
        coEvery { mockRepository.getUser("1") } returns Result.failure(Exception("Network error"))

        // Act
        val result = useCase("1")

        // Assert
        assertTrue(result.isFailure)
    }
}
```

**Why High Coverage:**
- Domain is pure logic, no framework dependencies
- Small, focused, low-cost tests
- Highest ROI on test investment

**Common Domain Tests:**
- Use Case execution with mocked repositories
- Exception handling
- Multiple branch paths (happy path, error cases)

---

#### Data Layer Tests (70%+ Coverage Target)

**Goal:** Verify data fetching, transformation, and persistence.

**Tools:** JUnit5 + MockK + Mockito (for Retrofit/Room mocking)

**Test Structure - Repository:**
```kotlin
class UserRepositoryImplTest {
    private val mockUserService = mockk<UserService>()
    private val mockUserDao = mockk<UserDao>()
    private val repository = UserRepositoryImpl(mockUserService, mockUserDao)

    @Test
    fun `getUser fetches from network and caches`() = runTest {
        // Arrange
        val userDto = UserDto("1", "john@example.com", "John")
        coEvery { mockUserService.getUser("1") } returns userDto
        coEvery { mockUserDao.insert(any()) } just runs

        // Act
        val result = repository.getUser("1")

        // Assert
        assertTrue(result.isSuccess)
        coVerify { mockUserDao.insert(any()) }  // Verify caching
    }

    @Test
    fun `getUser returns cached data on network error`() = runTest {
        // Arrange
        val cachedEntity = UserEntity("1", "john@example.com", "John", System.currentTimeMillis())
        coEvery { mockUserService.getUser("1") } throws Exception("Network error")
        coEvery { mockUserDao.getUser("1") } returns cachedEntity

        // Act
        // (Note: implementation-specific; may require fallback logic)

        // Assert
        coVerify { mockUserService.getUser("1") }  // Attempted network fetch
    }
}
```

**Test Structure - DTO Mapper:**
```kotlin
class UserMapperTest {
    @Test
    fun `toDomain maps UserDto to User`() {
        // Arrange
        val userDto = UserDto("1", "john@example.com", "John")

        // Act
        val user = userDto.toDomain()

        // Assert
        assertEquals("1", user.id)
        assertEquals("john@example.com", user.email)
        assertEquals("John", user.name)
    }
}
```

**Test Structure - Retrofit Service (Mock Server):**
```kotlin
@RunWith(AndroidJUnit4::class)
class UserServiceTest {
    @get:Rule
    val mockServerRule = MockWebServerRule()

    @Test
    fun `getUser returns mapped response`() = runTest {
        // Arrange
        mockServerRule.mockServer.enqueue(
            MockResponse().setBody("""
                {
                    "user_id": "1",
                    "email_address": "john@example.com",
                    "name": "John"
                }
            """).setResponseCode(200)
        )
        val service = mockServerRule.createService(UserService::class.java)

        // Act
        val response = service.getUser("1")

        // Assert
        assertEquals("1", response.userId)
    }
}
```

**Common Data Layer Tests:**
- Repository fetching + caching logic
- DTO serialization/deserialization
- Retrofit API mocking (via MockWebServer)
- Room DAO queries
- Mapper transformations

---

#### Presentation Layer Tests (60%+ Coverage Target)

**Goal:** Verify ViewModel state management and UI rendering.

**Tools:** JUnit5 + MockK + Compose Testing + Espresso

**Test Structure - ViewModel:**
```kotlin
class LoginViewModelTest {
    private val mockLoginUseCase = mockk<LoginUseCase>()
    private lateinit var viewModel: LoginViewModel

    @get:Rule
    val instantExecutorRule = InstantTaskExecutorRule()

    @Before
    fun setup() {
        viewModel = LoginViewModel(mockLoginUseCase)
    }

    @Test
    fun `login emits Loading then Success`() = runTest {
        // Arrange
        val token = AuthToken("jwt...", "refresh...", 3600)
        coEvery { mockLoginUseCase("john@example.com", "password") } returns Result.success(token)

        // Act
        viewModel.login("john@example.com", "password")
        advanceUntilIdle()  // Wait for viewModelScope.launch to complete

        // Assert
        val state = viewModel.uiState.value
        assertTrue(state is LoginUiState.Success)
        assertEquals(token, (state as LoginUiState.Success).token)
    }

    @Test
    fun `login emits Error on failure`() = runTest {
        // Arrange
        coEvery { mockLoginUseCase("john@example.com", "wrong") } returns Result.failure(Exception("Invalid credentials"))

        // Act
        viewModel.login("john@example.com", "wrong")
        advanceUntilIdle()

        // Assert
        val state = viewModel.uiState.value
        assertTrue(state is LoginUiState.Error)
    }
}
```

**Test Structure - Compose Screen (UI Test):**
```kotlin
@RunWith(AndroidJUnit4::class)
class LoginScreenTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    private val mockViewModel = mockk<LoginViewModel> {
        every { uiState } returns MutableStateFlow(LoginUiState.Idle)
    }

    @Test
    fun loginScreen_showsFormOnIdle() {
        // Arrange
        composeTestRule.setContent {
            AppTheme {
                LoginScreen(viewModel = mockViewModel)
            }
        }

        // Assert
        composeTestRule.onNodeWithText("Email").assertIsDisplayed()
        composeTestRule.onNodeWithText("Password").assertIsDisplayed()
        composeTestRule.onNodeWithText("Login").assertIsDisplayed()
    }

    @Test
    fun loginScreen_showsLoadingOnLoading() {
        // Arrange
        every { mockViewModel.uiState } returns MutableStateFlow(LoginUiState.Loading)
        composeTestRule.setContent {
            AppTheme {
                LoginScreen(viewModel = mockViewModel)
            }
        }

        // Assert
        composeTestRule.onNodeWithContentDescription("Loading").assertIsDisplayed()
    }

    @Test
    fun loginScreen_callsViewModelLoginOnButtonClick() {
        // Arrange
        composeTestRule.setContent {
            AppTheme {
                LoginScreen(viewModel = mockViewModel)
            }
        }

        // Act
        composeTestRule.onNodeWithText("Email").performTextInput("john@example.com")
        composeTestRule.onNodeWithText("Password").performTextInput("password")
        composeTestRule.onNodeWithText("Login").performClick()

        // Assert
        verify { mockViewModel.login("john@example.com", "password") }
    }
}
```

**Common Presentation Tests:**
- ViewModel state emissions (Idle, Loading, Success, Error)
- UI element visibility based on state
- User interactions (clicks, text input)
- Navigation callbacks
- Error message display

---

### Test Execution

```bash
# Unit tests (domain + data)
./gradlew test

# Instrumented tests (UI + integration)
./gradlew connectedAndroidTest

# All tests
./gradlew test connectedAndroidTest

# Test with coverage report
./gradlew testDebugUnitTest --coverage
```

### Test Report

After all tests pass, Test Agent generates a report:

```markdown
# Test Execution Report

## Domain Tests
- GetUserUseCaseTest: 2 tests, 2 passed
- LoginUseCaseTest: 3 tests, 3 passed
- WatchUserUseCaseTest: 2 tests, 2 passed

**Domain Coverage: 82% (target 80%+)**

## Data Tests
- UserRepositoryImplTest: 4 tests, 4 passed
- AuthRepositoryImplTest: 3 tests, 3 passed
- UserMapperTest: 3 tests, 3 passed
- UserServiceTest (MockWebServer): 2 tests, 2 passed
- UserDaoTest: 4 tests, 4 passed

**Data Coverage: 72% (target 70%+)**

## Presentation Tests
- LoginViewModelTest: 3 tests, 3 passed
- LoginScreenTest: 4 tests, 4 passed
- HomeViewModelTest: 2 tests, 2 passed
- HomeScreenTest: 3 tests, 3 passed

**Presentation Coverage: 65% (target 60%+)**

## Summary
✓ Total: 40 tests, 40 passed, 0 failed
✓ Coverage: 73% overall
✓ READY FOR BUILD
```

---

## Lint Agent

### Linting Tools

1. **ktlint** (Code Formatting)
   - Enforces Kotlin style guide
   - Auto-fixable: `./gradlew ktlintFormat`
   - Zero configuration (standard Kotlin rules)

2. **detekt** (Static Analysis)
   - Detects code smells, complexity, naming violations, architecture violations
   - Non-auto-fixable (requires manual review)
   - Custom rules for this pipeline (e.g., no Android imports in domain/)

### Lint Execution Pipeline

#### Step 1: ktlint Format

```bash
./gradlew ktlintFormat
```

**Auto-fixes:**
- Indentation (4 spaces)
- Line length (max 120 chars)
- Import ordering (alphabetical)
- Trailing commas
- Spacing (around operators, colons, etc.)

**Example Auto-Fix:**
```kotlin
// Before
fun    login(email:String,password:String) {
    return loginUseCase.invoke(email,password)
}

// After (ktlint auto-fixed)
fun login(email: String, password: String) {
    return loginUseCase.invoke(email, password)
}
```

#### Step 2: detekt Analysis

```bash
./gradlew detekt
```

**Rules Checked (Sample):**

| Rule | Severity | Example |
|------|----------|---------|
| **Complexity (McCabe > 15)** | MEDIUM | Nested conditions → suggest refactor |
| **Function Naming** | LOW | `get_user()` → should be `getUser()` |
| **Class Naming** | LOW | `loginViewmodel` → should be `LoginViewModel` |
| **Magic Numbers** | MEDIUM | `30000` timeout → create constant |
| **Long Parameter List** | MEDIUM | 6+ parameters → create data class |
| **Android Imports in Domain** | HIGH | `import android.*` in domain/ → FAIL |
| **Unused Variables** | LOW | `val x = getSomething()` (unused) → remove |
| **Unused Imports** | LOW | Auto-detect by ktlint |

**Custom Pipeline Rules:**

```yaml
# detekt.yml (project-specific configuration)
plugins:
  - "io.gitlab.arturbosch.detekt"

detekt:
  # Custom rule: no Android imports in domain/
  AndroidImportInDomain:
    active: true
    severity: error
    excludes: ['**/Test*.kt']
    patterns:
      - 'src/main/kotlin/domain/.*\.kt'

  # Standard rules
  Complexity:
    enabled: true
    ComplexityThreshold: 15

  Naming:
    enabled: true
    FunctionNamingRules:
      functionPattern: '^([a-z][a-zA-Z0-9]*)|(`.*`)$'
```

#### Step 3: Layer Boundary Check

After detekt, Lint Agent manually verifies layer boundaries:

```
✓ Domain layer (domain/):
  - Scan all imports
  - Forbidden: android.*, androidx.* (except specific whitelist)
  - Result: 0 violations

✓ Data layer (data/):
  - Forbidden: presentation.* imports
  - Result: 0 violations

✓ Presentation layer (presentation/):
  - Forbidden: data.* imports (direct use of repositories)
  - Allowed: domain.* imports only
  - Result: 0 violations
```

#### Step 4: Auto-Fix & Re-lint

If violations found:

```bash
# For complexity, naming, etc. that can't auto-fix:
# Manual review required, highlight lines, suggest refactor

# Re-run ktlint if fixes applied:
./gradlew ktlintFormat
```

#### Step 5: Lint Report

```markdown
# Lint Report

## ktlint (Code Formatting)
✓ Ran: ./gradlew ktlintFormat
✓ All files formatted
✓ No violations

## detekt (Static Analysis)
- Total violations: 3
- Errors (HIGH): 0
- Warnings (MEDIUM): 2
- Info (LOW): 1

### Violations:
1. **ComplexityViolation** (MEDIUM)
   - File: LoginViewModel.kt:42
   - Message: "Function 'login' has too many branches (complexity: 16)"
   - Suggestion: Extract nested conditions into helper function

2. **MagicNumberViolation** (MEDIUM)
   - File: NetworkModule.kt:15
   - Message: "Magic number 30000 (timeout in ms)"
   - Suggestion: Create constant: `const val TIMEOUT_MS = 30000`

3. **UnusedImportViolation** (LOW)
   - File: UserScreen.kt:3
   - Message: "Unused import: kotlin.coroutines.coroutineScope"
   - Suggestion: Remove import

## Layer Boundary Check
✓ Domain: 0 Android imports
✓ Data: 0 presentation imports
✓ Presentation: 0 direct data imports

## Summary
✓ Auto-fixes applied: ktlint formatting
⚠ Manual fixes needed: 3 violations (2 MEDIUM, 1 LOW)
  - Estimated time: 10 minutes to address

## Next: Pause for manual fixes, then re-lint
```

### Handling Lint Failures

**If Violations Found:**

```
Lint Agent identifies:
- LoginViewModel.kt:42: ComplexityViolation (function > 15 branches)

Routes to: Pres Lead (who wrote LoginViewModel)

Pres Lead action:
1. Review violation: confirm too many branches
2. Refactor: extract helper function
3. Re-run detekt to verify
4. Commit fix

Lint Agent re-checks: ✓ Violation resolved
```

**If Layer Violation Found:**

```
Lint Agent identifies:
- AuthRepositoryImpl.kt:5: "import android.util.Log"

Routes to: Integration Lead (layer boundary check)

Integration Lead action:
1. Review: confirm Android import in data/ (actually OK here)
   OR confirm violation: should be in data/, so auto-import is valid
2. If true violation: route back to Data Lead to move code
3. Otherwise: suppress rule with comment

Lint Agent re-checks: ✓ Verified
```

### Lint Configuration (build.gradle.kts)

```kotlin
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.1"
}

detekt {
    toolVersion = "1.23.1"
    config = files("config/detekt.yml")
    buildUponDefaultConfig = true
    autoCorrect = false  // Manual fixes for arch rules

    reports {
        html.enabled = true
        md.enabled = true
        txt.enabled = true
        sarif.enabled = true
    }
}

kotlin {
    compilerOptions {
        // Enforce nullability warnings
        allWarningsAsErrors = false
    }
}
```

### Lint Best Practices

1. **Run Early:** Lint during Code Gen Phase 3, not just before Build
2. **Layer Enforcement:** Custom detekt rule for domain/ purity
3. **Team Standards:** Share detekt.yml config across team
4. **Ignoring Rules:** Document why with `@Suppress` comment, not blindly ignore
   ```kotlin
   @Suppress("ComplexMethod")  // Necessary for complex state machine
   fun login(...) { ... }
   ```

---

## Test ‖ Lint Parallel Execution

```
Code Gen Phase 3 complete (Integration)
  ↓
Test Agent ‖ Lint Agent (start simultaneously)
  ├─ Test Agent:
  │  ├─ Generate test code (domain, data, presentation)
  │  ├─ Run tests: ./gradlew test
  │  └─ Generate coverage report
  │
  └─ Lint Agent:
     ├─ Run ktlint: ./gradlew ktlintFormat
     ├─ Run detekt: ./gradlew detekt
     ├─ Layer boundary validation
     └─ Generate lint report
  ↓
Both complete (test passing, lint clean)
  ↓
Build Agent can proceed
```

**Time Saved:** Running tests and linting in parallel saves ~30% execution time vs. sequential.

---

## Feedback Routes

### Test Failure

```
Test fails: LoginViewModelTest::testLoginEmitsSuccessState

↓ Identify root cause

Test type = Presentation (ViewModel)
  ↓
Route to: Pres Lead

Pres Lead:
1. Fix ViewModel implementation
2. Update/fix test if needed
3. Run: ./gradlew test
4. Verify: test passes
5. Return to Lint Agent for parallel checks
```

### Lint Violation

```
Lint violation: Android import in Domain

↓ Identify severity

Severity = CRITICAL (architecture rule)
  ↓
Route to: Integration (or violating layer lead)

Integration/Lead:
1. Move code to correct layer or remove Android call
2. Run: ./gradlew detekt
3. Verify: violation resolved
4. Return to Build Agent
```

---

## Integration with Build

After Test ‖ Lint complete:

```
Both agents report:
✓ Test Agent: All tests pass (73% coverage)
✓ Lint Agent: Zero violations

  ↓
Coordinator signals Build Agent:
"Ready to build. All checks passed."

  ↓
Build Agent: ./gradlew assembleDebug
```

If either Test or Lint fails:
- Do NOT proceed to Build
- Route to appropriate lead for fixes
- Re-run test/lint checks
- Then proceed to Build only when all pass
