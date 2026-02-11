# Product Agent

## Overview

The **Product Agent** gathers requirements, validates against Material Design 3 best practices and existing architecture, and produces a detailed specification that guides the Code Generation team.

## Workflow

### Step 1: Initial Clarification

Ask the user targeted questions to understand scope:

**For Feature Requests:**
```
User: "Add user authentication with JWT + offline caching"

Product Agent Questions:
1. Scope: Single login screen or include profile, logout, token refresh?
   User: "Just login + auto-refresh on app startup, logout on button"

2. Persistence strategy:
   - Encrypted SharedPreferences for JWT token? Or Room?
   - User profile cached in Room for offline access?
   User: "JWT in SharedPreferences (encrypted), profile in Room"

3. Error scenarios:
   - Invalid credentials (wrong password)?
   - Network timeout?
   - Expired token (auto-refresh)?
   - Server 500 error?
   User: "Handle all 4 gracefully, show toasts for user-facing errors"

4. Navigation:
   - New screens: LoginScreen + SplashScreen?
   - Or modify existing login UI?
   User: "New LoginScreen with Material Design 3, SplashScreen while checking token"

5. API Contract:
   - Endpoint: POST /auth/login?
   - Request: { email, password }?
   - Response: { token, refreshToken, expiresIn }?
   User: "Yes, plus user profile in response"

6. Material Design 3 alignment:
   - Color scheme (Material You dynamic colors)?
   - Typography (Headline, Body, Label scales)?
   - Components (OutlinedTextField, FilledButton)?
   User: "Material You if available, fall back to app theme"
```

**For Bug Fixes:**
```
User: "Fix login crash (NullPointerException)"

Product Agent Questions:
1. When does crash occur?
   User: "After entering credentials, clicking login button"

2. Stack trace (if available)?
   User: [Pastes stack trace → shows ViewModel.login() line 42]

3. Reproduced in:
   - Specific device/API level?
   - Online or offline?
   User: "Android 13+, with network"

4. Expected behavior?
   User: "Show loading spinner, then navigate to home screen or show error"
```

### Step 2: Validate Against Architecture & Material Design

**Architecture Validation:**
- Is this feature compatible with existing Clean Architecture?
- Does it require new layers (domain/data/presentation)?
- Can it reuse existing repositories?

**Material Design 3 Validation:**
- Are proposed screens/components following Material Design 3 guidelines?
- Color contrast passing WCAG AA?
- Touch targets ≥ 48dp?
- Typography using correct Material scales?

Example:
```
✓ LoginScreen proposal:
  - FilledButton (login) ✓ Material Design 3
  - OutlinedTextField (email/password) ✓ Material Design 3
  - Error message in red (#B3261E) ✓ Material M3 error color
  - Touch targets: buttons 48dp × 48dp ✓

✓ Architecture:
  - AuthRepository (domain/data) ✓ New
  - LoginUseCase (domain) ✓ New
  - LoginViewModel (presentation) ✓ New
  - No conflicts with existing UserRepository
```

### Step 3: Reference Pipeline Memory

Load patterns from prior runs to ensure consistency:

```
Pipeline Memory Findings:
✓ DTOs use @Serializable + @SerialName: "Applied to AuthDto"
✓ Error handling: Result<T> sealed class in domain: "LoginUseCase: suspend fun execute(...): Result<AuthToken>"
✓ Navigation: type-safe routes (sealed class): "object Login : Route(), object Authenticated : Route()"
✓ ViewModels: @HiltViewModel + MutableStateFlow + viewModelScope.launch: "LoginViewModel will follow pattern"
```

### Step 4: Create Detailed Specification

Produce a specification document (Markdown) with:

```markdown
# Feature Specification: User Authentication with JWT + Offline Caching

## Overview
Add JWT-based authentication to MyApp with offline user profile caching via Room.

## Use Cases

### Happy Path: Login with Valid Credentials
```
1. User navigates to LoginScreen
2. Enters email + password
3. Clicks "Login" button
4. ViewModel.login(email, password) called
   - Show Loading spinner
   - LoginUseCase.execute(email, password) called
5. Retrofit call: POST /auth/login { email, password }
6. Success response: { token, refreshToken, expiresIn, user: User }
7. Token saved to SharedPreferences (encrypted)
8. User profile saved to Room UserEntity
9. Navigate to HomeScreen
10. Success toast: "Logged in as John"
```

### Error Path: Invalid Credentials
```
1. User enters wrong password
2. API returns 401 { message: "Invalid credentials" }
3. ViewModel catches result.isFailure
4. UiState changes to LoginUiState.Error("Invalid credentials")
5. Screen shows error message in red
6. User can retry
```

### Error Path: Network Timeout
```
1. Network connection lost
2. Retrofit throws TimeoutException (30s timeout)
3. ViewModel catches, maps to AppException
4. UiState: LoginUiState.Error("Network error, check connection")
5. Offline mode: User can't login (no cached token bypass for first login)
```

### App Startup: Auto-Refresh Token
```
1. App launches (SplashScreen)
2. ViewModel checks SharedPreferences for stored token
3. If token exists + not expired:
   - Auto-navigate to HomeScreen
   - Background: refresh token via API
4. If no token or expired:
   - Navigate to LoginScreen
```

### Logout
```
1. User clicks logout button (on HomeScreen or Profile)
2. ViewModel.logout() called
3. Delete token from SharedPreferences
4. Delete user profile from Room
5. Navigate to LoginScreen
```

## Data Models

### Domain Layer (domain/entity/AuthToken.kt)
```kotlin
data class AuthToken(
    val token: String,
    val refreshToken: String,
    val expiresIn: Long
)

data class User(
    val id: String,
    val email: String,
    val name: String
)
```

### Data Layer DTOs (data/dto/AuthDto.kt)
```kotlin
@Serializable
data class LoginRequestDto(
    val email: String,
    val password: String
)

@Serializable
data class LoginResponseDto(
    @SerialName("token")
    val token: String,
    @SerialName("refresh_token")
    val refreshToken: String,
    @SerialName("expires_in")
    val expiresIn: Long,
    val user: UserDto
)

@Serializable
data class UserDto(
    val id: String,
    val email: String,
    val name: String
)
```

### Room Persistence (data/local/UserEntity.kt)
```kotlin
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val email: String,
    val name: String,
    val cachedAt: Long
)
```

## Repositories & Use Cases

### AuthRepository (domain/repository/AuthRepository.kt)
Interface (domain):
```kotlin
interface AuthRepository {
    suspend fun login(email: String, password: String): Result<AuthToken>
    suspend fun logout(): Result<Unit>
    suspend fun refreshToken(refreshToken: String): Result<AuthToken>
    suspend fun getCachedUser(): Result<User>
}
```

Implementation (data/repository/AuthRepositoryImpl.kt):
```kotlin
@Singleton
class AuthRepositoryImpl @Inject constructor(
    private val authService: AuthService,
    private val userDao: UserDao,
    private val tokenCache: TokenCache // SharedPreferences wrapper
) : AuthRepository {
    override suspend fun login(...): Result<AuthToken> =
        runCatching {
            val request = LoginRequestDto(email, password)
            val response = authService.login(request)
            val token = AuthToken(response.token, response.refreshToken, response.expiresIn)
            tokenCache.save(token)
            userDao.insert(response.user.toDomain().toEntity())
            token
        }
}
```

### LoginUseCase (domain/usecase/LoginUseCase.kt)
```kotlin
class LoginUseCase @Inject constructor(
    private val authRepository: AuthRepository
) {
    suspend operator fun invoke(email: String, password: String): Result<AuthToken> =
        authRepository.login(email, password)
}
```

## Presentation Layer

### UiState (presentation/feature/LoginUiState.kt)
```kotlin
sealed class LoginUiState {
    object Idle : LoginUiState()
    object Loading : LoginUiState()
    data class Success(val authToken: AuthToken) : LoginUiState()
    data class Error(val message: String) : LoginUiState()
}
```

### ViewModel (presentation/feature/LoginViewModel.kt)
```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val loginUseCase: LoginUseCase
) : ViewModel() {
    private val _uiState = MutableStateFlow<LoginUiState>(LoginUiState.Idle)
    val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()

    fun login(email: String, password: String) {
        viewModelScope.launch {
            _uiState.value = LoginUiState.Loading
            val result = loginUseCase(email, password)
            _uiState.value = when {
                result.isSuccess -> LoginUiState.Success(result.getOrNull()!!)
                else -> LoginUiState.Error(result.exceptionOrNull()?.message ?: "Unknown error")
            }
        }
    }
}
```

### Compose Screen (presentation/feature/LoginScreen.kt)
```
[ASCII mockup of LoginScreen]

┌────────────────────────────────────────┐
│                                        │
│           MyApp Login                  │
│                                        │
│  ┌──────────────────────────────────┐ │
│  │ Email                            │ │
│  │ ___________________________       │ │
│  │                                  │ │
│  │ Password                         │ │
│  │ ___________________________       │ │
│  │                                  │ │
│  │  ┌────────────────────────────┐ │ │
│  │  │       LOGIN BUTTON         │ │ │
│  │  └────────────────────────────┘ │ │
│  │                                  │ │
│  │  [Error message in red]          │ │
│  └──────────────────────────────────┘ │
│                                        │
└────────────────────────────────────────┘

Components:
- OutlinedTextField (email, password) ✓ Material Design 3
- FilledButton (Login) ✓ Material Design 3
- Text (error) ✓ Material Design 3 error color
- CircularProgressIndicator (loading) ✓ Material Design 3
```

## Navigation

Type-safe routes:
```kotlin
sealed class AuthRoute {
    object Login : AuthRoute()
    object Authenticated : AuthRoute()
}

// NavHost in MainActivity:
NavHost(navController, startDestination = AuthRoute.Login) {
    composable<AuthRoute.Login> {
        LoginScreen(
            onLoginSuccess = {
                navController.navigate(AuthRoute.Authenticated) {
                    popUpTo(AuthRoute.Login) { inclusive = true }
                }
            }
        )
    }
    composable<AuthRoute.Authenticated> { HomeScreen() }
}
```

## Hilt Dependency Injection

### AuthModule (di/AuthModule.kt)
```kotlin
@Module
@InstallIn(SingletonComponent::class)
interface AuthModule {
    @Binds
    @Singleton
    fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository
}

@Module
@InstallIn(SingletonComponent::class)
object AuthProvidersModule {
    @Provides
    @Singleton
    fun provideAuthService(retrofit: Retrofit): AuthService =
        retrofit.create(AuthService::class.java)

    @Provides
    @Singleton
    fun provideTokenCache(@ApplicationContext context: Context): TokenCache =
        SharedPrefsTokenCache(context)
}
```

## Testing Strategy

### Domain Tests
```kotlin
class LoginUseCaseTest {
    private lateinit var useCase: LoginUseCase
    private val mockRepository = mockk<AuthRepository>()

    @Test
    fun `invoke calls repository and returns result`() = runTest {
        val token = AuthToken("jwt...", "refresh...", 3600)
        coEvery { mockRepository.login("john@example.com", "password") } returns Result.success(token)

        val result = useCase("john@example.com", "password")

        assertTrue(result.isSuccess)
        assertEquals(token, result.getOrNull())
    }
}
```

### Data Tests (Retrofit)
```kotlin
@Test
fun `login returns mapped token on 200 response`() = runTest {
    // Mock: Retrofit returns LoginResponseDto
    // Verify: token saved to SharedPreferences + user saved to Room
}
```

### Presentation Tests (ViewModel + UI)
```kotlin
class LoginViewModelTest {
    @Test
    fun `login emits Loading then Success`() = runTest {
        // Arrange: mock LoginUseCase
        // Act: viewModel.login(...)
        // Assert: uiState changes Loading → Success
    }
}

// Compose test
@Test
fun `LoginScreen shows error on failure`() {
    // Arrange: mock ViewModel with Error state
    // Act: render LoginScreen
    // Assert: error text is displayed
}
```

## Material Design 3 Compliance

✓ Color scheme: Uses app theme (Material You if available)
✓ Typography: Material Design 3 scales (Headline, Body, Label)
✓ Components: OutlinedTextField, FilledButton, CircularProgressIndicator
✓ Spacing: 16dp horizontal padding, 8dp vertical spacing
✓ Touch targets: All interactive elements ≥ 48dp
✓ Accessibility: contentDescription on all buttons + icons
✓ Dark mode: Supports system dark theme (Material Design 3)

## Error Handling

- **NetworkException** (timeout, DNS fail): User-friendly "Check your connection"
- **InvalidCredentials** (401): "Email or password incorrect"
- **ServerError** (5xx): "Server error, please try again later"
- **Generic** (unknown): "Something went wrong, contact support"

All errors shown as Material Design 3 styled error message in red (#B3261E).

## Acceptance Criteria

- [ ] LoginScreen renders with Material Design 3 components
- [ ] Successful login stores JWT token + user profile
- [ ] Failed login shows error message (invalid credentials)
- [ ] Network error handled gracefully (timeout)
- [ ] Auto-refresh on app startup if token exists
- [ ] Logout clears token + user data
- [ ] Domain: zero Android imports (pure Kotlin)
- [ ] Data: Retrofit service + Room DAO + mapper
- [ ] Presentation: ViewModel + UiState + Compose screen
- [ ] Navigation: type-safe routes, single activity
- [ ] Hilt: all repositories bound (@Binds or @Provides)
- [ ] Tests: 75%+ coverage (domain/data/presentation)
- [ ] ktlint + detekt: zero violations
- [ ] Build: ./gradlew build succeeds
```

### Step 5: Create Material Design 3 Mockup

Create a detailed Compose component mockup (ASCII or reference Material Design 3 patterns):

```
[Material Design 3 LoginScreen Mockup]

Color Palette:
- Primary: Color(0xFF6750A4) (Material M3 Purple)
- Error: Color(0xFFB3261E) (Material M3 Red)
- Surface: Color(0xFFFFFBFE)
- On Surface: Color(0xFF1C1B1F)

Typography:
- Headline: HeadlineSmall (size 24sp, weight Medium)
- Body: BodyMedium (size 14sp, weight Regular)
- Label: LabelMedium (size 12sp, weight Medium)

Components:
- OutlinedTextField: email input, outlined border
- OutlinedTextField: password input, outlined border, password visualTransformation
- FilledButton: Login action, background primary color
- Text: error message, color error
- CircularProgressIndicator: loading spinner, color primary
- Surface container for form

Spacing: Material Design 3 16dp/8dp grid
Dark mode: Follows system theme
Accessibility: All touch targets ≥ 48dp, contentDescription on buttons
```

### Step 6: Return Spec to Scaffold/Code Gen

Output: **Detailed Specification** (Markdown) + **Material Design 3 Mockup** (Compose reference or ASCII).

This spec is passed to:
1. **Scaffold**: Create folder structure + Hilt module skeleton
2. **Architect**: Design module boundaries + error handling strategy
3. **Code Gen Team**: Domain Lead, Data Lead, Pres Lead reference this spec

## Product Agent Responsibilities

- ✓ Clarify user intent (feature/bug/refactor/design/sprint/dependency/PR/release/test)
- ✓ Validate scope against existing architecture
- ✓ Reference Material Design 3 guidelines
- ✓ Load + apply Pipeline Memory patterns
- ✓ Create detailed specification (use cases, data models, API contracts, navigation, error scenarios)
- ✓ Create Material Design 3 mockup (Compose reference)
- ✓ Define testing strategy per layer
- ✓ Hand off to Scaffold (if new feature) or Code Gen team (if enhancement)

## Troubleshooting

### Problem: User Request Too Vague
**Example:** "Add a feature"
**Solution:** Ask specific clarifying questions (from Step 1)

### Problem: Spec Conflicts with Existing Architecture
**Example:** "Add feature that requires breaking existing repository pattern"
**Solution:** Flag conflict, suggest refactor scope, consult with Architect

### Problem: User Wants Feature Outside Material Design 3 Guidelines
**Example:** "Custom button that doesn't follow Material Design 3"
**Solution:** Warn about accessibility/consistency risks, suggest Material Design 3 alternative
