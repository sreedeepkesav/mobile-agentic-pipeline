# Android Standards

This document defines the comprehensive standards enforced by the Android Full Pipeline.

## Language & Runtime

### Kotlin
- **Version:** 1.9+ (stable)
- **JVM Target:** 11+
- **Coroutines:** kotlin-coroutines 1.7+
- **Serialization:** kotlinx-serialization 1.5+ (for DTOs)
- **Features:**
  - Data classes for entities and DTOs
  - Sealed classes for state + error handling
  - Extension functions for mappers
  - Suspend functions for async
  - Flow<T> for reactive streams

### JVM/Android Runtime
- **minSdkVersion:** 24+ (Android 7.0)
- **targetSdkVersion:** 34+ (Android 14)
- **compileSdkVersion:** 34+

---

## Jetpack Compose & UI

### Compose Only
- **No XML layouts** — all UI via Jetpack Compose
- **Version:** Latest stable (1.5+)
- **Single Activity architecture** (MainActivity hosts NavHost)

### Material Design 3
- **Color Scheme:** Material You dynamic colors (if API 31+), fallback to app theme
- **Typography:** Material Design 3 text scales
  - `HeadlineSmall`, `HeadlineMedium`, `HeadlineLarge`
  - `BodySmall`, `BodyMedium`, `BodyLarge`
  - `LabelSmall`, `LabelMedium`, `LabelLarge`
- **Components:** Use Material Design 3 component library
  - `OutlinedTextField`, `FilledTextField`
  - `Button`, `FilledButton`, `OutlinedButton`, `TextButton`
  - `Card`, `ElevatedCard`
  - `CircularProgressIndicator`, `LinearProgressIndicator`
  - `TopAppBar`, `NavigationBar`, `NavigationRail`
  - `LazyColumn`, `LazyRow` (no ListView/RecyclerView)
- **Spacing:** Material Design 3 grid (4dp, 8dp, 16dp, 24dp, 32dp)
- **Touch Targets:** Minimum 48dp × 48dp for interactive elements
- **Dark Mode:** System dark theme supported via `isDarkMode` flag or system theme

### Composable Functions
- Every screen is a `@Composable` function
- Arguments passed via function parameters (not Intent extras, though saved state for nav args is OK)
- Return type: `Unit` (side effect: render UI)
- Preview support: `@Preview` for Compose preview tests

**Example:**
```kotlin
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = hiltViewModel(),
    onNavigateToHome: () -> Unit
) {
    // Compose tree
}

@Preview
@Composable
fun LoginScreenPreview() {
    AppTheme {
        LoginScreen(onNavigateToHome = {})
    }
}
```

---

## Architecture (Clean Architecture)

### 3 Layers

#### Domain Layer (Pure Kotlin)
- **Purpose:** Business logic, isolated from Android/framework
- **Constraints:** Zero Android SDK imports
  - ✓ Allowed: `kotlin.*`, `kotlin.coroutines.*`, custom exceptions
  - ✗ Forbidden: `android.*`, `androidx.*`
- **Contents:**
  - **Entities:** Data classes representing core business objects
    ```kotlin
    data class User(
        val id: String,
        val email: String,
        val name: String
    )
    ```
  - **Use Cases:** Single responsibility, take inputs, return outputs
    ```kotlin
    class GetUserUseCase @Inject constructor(
        private val userRepository: UserRepository
    ) {
        suspend operator fun invoke(id: String): Result<User> =
            userRepository.getUser(id)
    }
    ```
  - **Repository Interfaces:** Contracts, no implementation
    ```kotlin
    interface UserRepository {
        suspend fun getUser(id: String): Result<User>
        fun watchUser(id: String): Flow<User>
    }
    ```
  - **Exceptions:** Custom domain exceptions
    ```kotlin
    sealed class DomainException : Exception()
    object InvalidInput : DomainException()
    object NetworkError : DomainException()
    ```

#### Data Layer (Android + Frameworks)
- **Purpose:** Data fetching, caching, persistence
- **Can Import:** `domain.*` freely, `androidx.*` for Room/Retrofit
- **Forbidden:** `presentation.*` imports
- **Contents:**
  - **DTOs (Data Transfer Objects):** JSON-serializable, match API structure
    ```kotlin
    @Serializable
    data class UserDto(
        @SerialName("user_id")
        val userId: String,
        val email: String
    )
    ```
  - **Mappers:** DTO ↔ Domain entity conversion
    ```kotlin
    fun UserDto.toDomain(): User = User(userId, email, name)
    ```
  - **Retrofit Services:** HTTP API contracts
    ```kotlin
    interface UserService {
        @GET("users/{id}")
        suspend fun getUser(@Path("id") id: String): UserDto
    }
    ```
  - **Room DAOs + Entities:** Local database
    ```kotlin
    @Entity(tableName = "users")
    data class UserEntity(
        @PrimaryKey val id: String,
        val email: String
    )

    @Dao
    interface UserDao {
        @Query("SELECT * FROM users WHERE id = :id")
        suspend fun getUser(id: String): UserEntity?
    }
    ```
  - **Repository Implementations:** Combine Retrofit + Room
    ```kotlin
    @Singleton
    class UserRepositoryImpl @Inject constructor(
        private val service: UserService,
        private val dao: UserDao
    ) : UserRepository {
        override suspend fun getUser(id: String): Result<User> =
            runCatching {
                val dto = service.getUser(id)
                dao.insert(dto.toEntity())
                dto.toDomain()
            }
    }
    ```

#### Presentation Layer (UI + ViewModels)
- **Purpose:** UI rendering, user interactions, state management
- **Can Import:** `domain.*` freely, `androidx.compose.*`, `androidx.viewmodel.*`
- **Forbidden:** Direct `data.*` imports (use domain layer interfaces)
- **Contents:**
  - **UiState Sealed Classes:** Screen state representation
    ```kotlin
    sealed class LoginUiState {
        object Idle : LoginUiState()
        object Loading : LoginUiState()
        data class Success(val token: AuthToken) : LoginUiState()
        data class Error(val message: String) : LoginUiState()
    }
    ```
  - **ViewModels:** State holders, lifecycle-aware
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
                    result.isSuccess -> LoginUiState.Success(...)
                    else -> LoginUiState.Error(...)
                }
            }
        }
    }
    ```
  - **Compose Screens:** Render UI, collect state
    ```kotlin
    @Composable
    fun LoginScreen(
        viewModel: LoginViewModel = hiltViewModel()
    ) {
        val uiState by viewModel.uiState.collectAsStateWithLifecycle()

        when (uiState) {
            is LoginUiState.Loading -> CircularProgressIndicator()
            is LoginUiState.Success -> { /* navigate */ }
            is LoginUiState.Error -> Text((uiState as LoginUiState.Error).message)
            is LoginUiState.Idle -> { /* form */ }
        }
    }
    ```
  - **Navigation:** Type-safe routing
    ```kotlin
    sealed class Route {
        object Login : Route()
        object Home : Route()
    }

    NavHost(navController, startDestination = Route.Login) {
        composable<Route.Login> { LoginScreen(...) }
        composable<Route.Home> { HomeScreen(...) }
    }
    ```

### Module Structure (Recommended)
```
src/main/kotlin/com/example/app/
├── di/
│   ├── AppModule.kt
│   ├── AuthModule.kt
│   ├── UserModule.kt
│   └── NetworkModule.kt
├── domain/
│   ├── entity/
│   │   ├── User.kt
│   │   └── AuthToken.kt
│   ├── usecase/
│   │   ├── LoginUseCase.kt
│   │   └── GetUserUseCase.kt
│   ├── repository/
│   │   ├── AuthRepository.kt
│   │   └── UserRepository.kt
│   └── exception/
│       └── DomainException.kt
├── data/
│   ├── dto/
│   │   ├── UserDto.kt
│   │   └── AuthDto.kt
│   ├── mapper/
│   │   ├── UserMapper.kt
│   │   └── AuthMapper.kt
│   ├── network/
│   │   ├── AuthService.kt
│   │   └── UserService.kt
│   ├── local/
│   │   ├── UserEntity.kt
│   │   ├── UserDao.kt
│   │   └── AppDatabase.kt
│   └── repository/
│       ├── AuthRepositoryImpl.kt
│       └── UserRepositoryImpl.kt
└── presentation/
    ├── theme/
    │   └── AppTheme.kt
    ├── navigation/
    │   └── Route.kt
    └── feature/
        ├── login/
        │   ├── LoginScreen.kt
        │   ├── LoginViewModel.kt
        │   └── LoginUiState.kt
        └── home/
            ├── HomeScreen.kt
            ├── HomeViewModel.kt
            └── HomeUiState.kt

androidTest/
└── kotlin/
    └── com/example/app/
        ├── ui/
        │   ├── LoginScreenTest.kt
        │   └── HomeScreenTest.kt
        └── repository/
            └── UserRepositoryImplTest.kt

test/
└── kotlin/
    └── com/example/app/
        ├── domain/
        │   ├── GetUserUseCaseTest.kt
        │   └── LoginUseCaseTest.kt
        ├── data/
        │   └── UserRepositoryImplTest.kt
        └── presentation/
            ├── LoginViewModelTest.kt
            └── HomeViewModelTest.kt
```

---

## Dependency Injection (Hilt)

### Setup
- `@HiltAndroidApp` on Application class
- `@AndroidEntryPoint` on Activities/Fragments (if needed)
- Hilt version 2.45+

### Module Patterns

**@Binds (Interface → Implementation)**
```kotlin
@Module
@InstallIn(SingletonComponent::class)
interface UserModule {
    @Binds
    @Singleton
    fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}
```

**@Provides (Complex Objects)**
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .build()

    @Provides
    @Singleton
    fun provideUserService(retrofit: Retrofit): UserService =
        retrofit.create(UserService::class.java)
}
```

**@HiltViewModel (ViewModels)**
```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val loginUseCase: LoginUseCase
) : ViewModel()
```

### Scopes
- **@Singleton:** App-wide (repositories, services, database)
- **@ActivityScoped:** Activity lifetime (if using Fragments)
- **@ViewModelScoped:** ViewModel lifetime (rarely needed with Compose)

---

## State Management

### StateFlow + UiState Sealed Class

**Pattern:**
```kotlin
sealed class LoginUiState {
    object Idle : LoginUiState()
    object Loading : LoginUiState()
    data class Success(val data: T) : LoginUiState()
    data class Error(val message: String) : LoginUiState()
}

class LoginViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<LoginUiState>(LoginUiState.Idle)
    val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()

    fun performAction() {
        viewModelScope.launch {
            _uiState.value = LoginUiState.Loading
            val result = useCase(...)
            _uiState.value = when {
                result.isSuccess -> LoginUiState.Success(result.getOrNull()!!)
                else -> LoginUiState.Error(...)
            }
        }
    }
}
```

**Collection in Compose:**
```kotlin
@Composable
fun LoginScreen(viewModel: LoginViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    // Use uiState...
}
```

**Key Rules:**
- One `UiState` sealed class per screen
- `MutableStateFlow` private, expose immutable `StateFlow`
- Only emit state in `viewModelScope.launch`
- Collect with `collectAsStateWithLifecycle()` (prevents leaks)
- Never hold mutable state in Composables (use ViewModel)

### Unidirectional Data Flow
```
User Input (click button)
    ↓
ViewModel.action() called
    ↓
ViewModel updates _uiState (emit new state)
    ↓
Compose screen collects state
    ↓
UI re-renders
```

No reverse data flow (UI never directly modifies ViewModel properties).

---

## Persistence

### Room (SQLite)
- **Version:** 2.5+
- **Database:** Single `AppDatabase` instance (provide via Hilt)
- **Entities:** `@Entity` data classes in domain layer
- **DAOs:** `@Dao` interfaces with `@Query`, `@Insert`, `@Update`, `@Delete`
- **Migrations:** Document schema changes in `@Migration` blocks
- **Dispatcher:** Room uses `Dispatchers.IO` by default (safe for suspend functions)

**Example:**
```kotlin
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val email: String
)

@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUser(id: String): UserEntity?

    @Query("SELECT * FROM users")
    fun watchAllUsers(): Flow<List<UserEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: UserEntity)

    @Query("DELETE FROM users")
    suspend fun deleteAll()
}

@Database(
    entities = [UserEntity::class],
    version = 1,
    exportSchema = false
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

### SharedPreferences (Simple KV)
- Use for non-sensitive, simple key-value data (e.g., app preferences, theme setting)
- For sensitive data (JWT tokens), use EncryptedSharedPreferences
- Avoid for structured data (use Room instead)

---

## Networking

### Retrofit
- **Version:** 2.10+
- **Converter:** Kotlinx Serialization (not Gson)
- **Timeout:** 30 seconds (configurable via interceptor)
- **Interceptors:** Add auth tokens, error handling

**Example:**
```kotlin
interface UserService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): UserDto

    @POST("users")
    suspend fun createUser(@Body dto: UserDto): UserDto

    @PUT("users/{id}")
    suspend fun updateUser(@Path("id") id: String, @Body dto: UserDto): UserDto
}
```

### DTOs (@Serializable)
- Use Kotlinx Serialization `@Serializable` annotation
- Map JSON field names via `@SerialName`

**Example:**
```kotlin
@Serializable
data class UserDto(
    @SerialName("user_id")
    val userId: String,
    @SerialName("email_address")
    val email: String
)
```

---

## Testing

### Framework
- **Unit Tests:** JUnit 5
- **Mocking:** MockK
- **UI Tests:** Compose Testing + Espresso
- **Test Runners:** Gradle tasks `./gradlew test`, `./gradlew connectedAndroidTest`

### Layer-Aware Testing

**Domain (80%+ coverage):**
- Test Use Cases with mocked repositories
- Pure Kotlin, no Android mocks
- Example:
  ```kotlin
  class GetUserUseCaseTest {
      private val mockRepository = mockk<UserRepository>()
      private val useCase = GetUserUseCase(mockRepository)

      @Test
      fun `invoke returns user on success`() = runTest {
          coEvery { mockRepository.getUser("1") } returns Result.success(user)
          val result = useCase("1")
          assertTrue(result.isSuccess)
      }
  }
  ```

**Data (70%+ coverage):**
- Test Repository implementations with mocked Retrofit/Room
- Example:
  ```kotlin
  class UserRepositoryImplTest {
      private val mockService = mockk<UserService>()
      private val mockDao = mockk<UserDao>()
      private val repo = UserRepositoryImpl(mockService, mockDao)

      @Test
      fun `getUser fetches from network and caches`() = runTest {
          coEvery { mockService.getUser("1") } returns userDto
          coEvery { mockDao.insert(any()) } just runs

          repo.getUser("1")

          coVerify { mockDao.insert(any()) }
      }
  }
  ```

**Presentation (60%+ coverage):**
- Unit tests for ViewModel state emissions
- UI tests for screen rendering

**ViewModel Test:**
```kotlin
class LoginViewModelTest {
    private val mockLoginUseCase = mockk<LoginUseCase>()
    private val viewModel = LoginViewModel(mockLoginUseCase)

    @get:Rule
    val instantExecutorRule = InstantTaskExecutorRule()

    @Test
    fun `login emits Loading then Success`() = runTest {
        coEvery { mockLoginUseCase(...) } returns Result.success(token)

        viewModel.login("email", "password")
        advanceUntilIdle()

        assertEquals(LoginUiState.Success(token), viewModel.uiState.value)
    }
}
```

**Compose UI Test:**
```kotlin
@RunWith(AndroidJUnit4::class)
class LoginScreenTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun loginScreen_showsLoadingOnLogin() {
        composeTestRule.setContent {
            LoginScreen()
        }

        composeTestRule.onNodeWithText("Login").performClick()
        composeTestRule.onNodeWithContentDescription("Loading").assertIsDisplayed()
    }
}
```

---

## Code Style & Naming

### Naming Conventions

| Element | Pattern | Example |
|---------|---------|---------|
| **Classes/Types** | PascalCase | `User`, `LoginViewModel`, `UserRepository` |
| **Functions/Variables** | camelCase | `getUser()`, `login()`, `userId` |
| **Constants** | SCREAMING_SNAKE_CASE | `API_BASE_URL`, `MAX_RETRY_COUNT` |
| **Data Classes (DTO)** | `[Entity]Dto` | `UserDto`, `LoginDto` |
| **Room Entities** | `[Entity]Entity` | `UserEntity`, `AuthTokenEntity` |
| **ViewModels** | `[Feature]ViewModel` | `LoginViewModel`, `UserViewModel` |
| **Compose Screens** | `[Feature]Screen` | `LoginScreen`, `UserScreen` |
| **UiState Classes** | `[Feature]UiState` | `LoginUiState`, `UserUiState` |
| **Use Cases** | `[Action][Entity]UseCase` | `GetUserUseCase`, `LoginUseCase` |
| **Repositories (interface)** | `[Entity]Repository` | `UserRepository`, `AuthRepository` |
| **Repositories (impl)** | `[Entity]RepositoryImpl` | `UserRepositoryImpl`, `AuthRepositoryImpl` |
| **Hilt Modules** | `[Feature]Module` | `AuthModule`, `UserModule`, `NetworkModule` |
| **Navigation Routes** | `[Feature]Route` or nested sealed class | `AuthRoute.Login`, `MainRoute.Home` |

### Code Formatting

- **ktlint:** Enforces Kotlin formatting standard
- **detekt:** Enforces complexity, naming, and architecture rules
- Run before commit: `./gradlew ktlintFormat && ./gradlew detekt`

---

## Build Configuration

### build.gradle.kts
```kotlin
plugins {
    id("com.android.application")
    kotlin("android")
    kotlin("kapt")
    id("dagger.hilt.android.plugin")
}

android {
    namespace = "com.example.app"
    compileSdk = 34
    defaultConfig {
        applicationId = "com.example.app"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0.0"
    }
}

dependencies {
    // Kotlin
    implementation("org.jetbrains.kotlin:kotlin-stdlib:1.9.21")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.5.1")

    // Jetpack
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.activity:activity-compose:1.8.0")
    implementation("androidx.compose.ui:ui:1.5.4")
    implementation("androidx.compose.material3:material3:1.1.2")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.6.2")
    implementation("androidx.navigation:navigation-compose:2.7.5")

    // Room
    implementation("androidx.room:room-runtime:2.5.2")
    implementation("androidx.room:room-ktx:2.5.2")
    kapt("androidx.room:room-compiler:2.5.2")

    // Retrofit
    implementation("com.squareup.retrofit2:retrofit:2.10.0")
    implementation("com.squareup.retrofit2:converter-kotlinx-serialization:2.10.0")

    // Hilt
    implementation("com.google.dagger:hilt-android:2.48")
    kapt("com.google.dagger:hilt-compiler:2.48")

    // Testing
    testImplementation("junit:junit:4.13.2")
    testImplementation("io.mockk:mockk:1.13.8")
    androidTestImplementation("androidx.compose.ui:ui-test-junit4:1.5.4")
}
```

---

## Error Handling

### Domain Layer
- Throw custom exceptions (no Android exceptions)
- Use `Result<T>` sealed class for return values

```kotlin
sealed class DomainResult<out T> {
    data class Success<T>(val data: T) : DomainResult<T>()
    data class Failure(val exception: Exception) : DomainResult<Nothing>()
}
```

### Data Layer
- Catch framework exceptions (Retrofit, Room)
- Convert to domain exceptions
- Return `Result<T>` to domain

```kotlin
override suspend fun getUser(id: String): Result<User> =
    runCatching {
        val dto = userService.getUser(id)
        dto.toDomain()
    }
```

### Presentation Layer
- Catch exceptions in `viewModelScope.launch`
- Map to `UiState.Error`
- Display user-friendly error message

```kotlin
fun login(...) {
    viewModelScope.launch {
        val result = loginUseCase(...)
        _uiState.value = when {
            result.isFailure -> LoginUiState.Error(result.exceptionOrNull()?.message ?: "Unknown error")
            else -> LoginUiState.Success(...)
        }
    }
}
```

---

## Accessibility

- **Content Descriptions:** `contentDescription` on all images/icons
- **Touch Targets:** Minimum 48dp × 48dp
- **Color Contrast:** WCAG AA compliant (4.5:1 text, 3:1 non-text)
- **Semantics:** Proper roles for interactive elements (button, text input, etc.)
- **Font Size:** Respect user font size preferences (use `sp` units)

---

## Performance Guidelines

- **Offline-First:** Cache data in Room, sync with Retrofit on network
- **Pagination:** Use Paging 3 for large lists
- **Lazy Loading:** `LazyColumn` with `items` for dynamic lists
- **Flow for Reactive Updates:** `watchUser(): Flow<User>` for real-time UI sync
- **Debounce:** `debounce()` on search/input flows to avoid rapid API calls

---

## Gradle Tasks

```bash
./gradlew build              # Full build + test
./gradlew assembleDebug      # Debug APK
./gradlew assembleRelease    # Release APK (requires signing config)
./gradlew test               # Unit tests
./gradlew connectedAndroidTest  # Instrumented tests
./gradlew ktlintFormat       # Auto-format code
./gradlew detekt             # Static analysis
./gradlew lint               # Android lint
```

---

## Summary Checklist

- ✓ Kotlin 1.9+, min SDK 24+, target SDK 34+
- ✓ Jetpack Compose only (no XML layouts)
- ✓ Material Design 3 components + theming
- ✓ Clean Architecture (Domain, Data, Presentation)
- ✓ Domain: zero Android imports
- ✓ Hilt DI (@HiltViewModel, @Module, @Binds, @Provides)
- ✓ StateFlow + UiState sealed classes
- ✓ Retrofit + Room (Kotlinx Serialization)
- ✓ Type-safe Navigation Compose
- ✓ JUnit5 + MockK testing (layer-aware)
- ✓ ktlint + detekt (auto-format + lint)
- ✓ Accessibility (descriptions, touch targets, contrast)
- ✓ Performance (offline-first, lazy loading, Flow)
