# Code Generation Phases (4-Phase Team)

## Overview

The Code Generation team consists of 5 specialized sub-agents working across 4 phases:

| Phase | Lead Agent | Role | Output |
|-------|-----------|------|--------|
| **1** | Architect | Design module boundaries, interfaces, error handling, Hilt module plan | Blueprint ADR |
| **2a** | Domain Lead | Pure Kotlin entities, use cases, repository interfaces | Domain layer code |
| **2b** | Data Lead | DTOs, mappers, Retrofit services, Room DAOs, repo implementations, Hilt @Binds | Data layer code |
| **2b** | Presentation Lead | ViewModels, Compose screens, Navigation, UiState sealed classes | Presentation layer code |
| **3** | Integration | Hilt module assembly, layer boundary validation, compile check, ktlint/detekt prep | Integrated codebase |

**Execution Model:**
```
Phase 1 (Architect) completes
  ↓
Phase 2a (Domain) ‖ Phase 2b (Data + Presentation) execute in parallel
  ↓
Phase 3 (Integration) validates + assembles all code
```

---

## Phase 1: Architect (Design Lead)

**Goal:** Design module boundaries, interface contracts, error handling strategy, Hilt binding plan.

**Inputs:**
- Detailed spec from Product Agent
- Detected existing architecture
- Pipeline Memory patterns

**Actions:**

1. **Analyze Module Boundaries**
   - Which modules are affected (existing + new)?
   - Are there cross-module dependencies?
   - Recommended folder structure

   Example:
   ```
   auth/ (new feature module)
   ├── domain/
   │   ├── entity/
   │   ├── repository/ (interface)
   │   └── usecase/
   ├── data/
   │   ├── dto/
   │   ├── mapper/
   │   ├── repository/ (impl)
   │   ├── network/
   │   └── local/
   └── presentation/
       ├── login/ (screens)
       └── splash/ (screens)

   Dependency Flow:
   presentation → domain → (isolated)
   data → domain → (isolated)
   (no circular deps, no presentation ↔ data)
   ```

2. **Define Interface Contracts (Domain Layer)**
   - Repository interfaces (method signatures, suspend functions, Flow return types)
   - Use Case signatures (suspend operator fun invoke, Result<T> or Either<L, R>)
   - Exceptions (custom domain exceptions, no Android imports)

   Example:
   ```kotlin
   // AuthRepository interface
   interface AuthRepository {
       suspend fun login(email: String, password: String): Result<AuthToken>
       suspend fun logout(): Result<Unit>
       fun watchUser(): Flow<User>
   }

   // Exceptions
   sealed class AuthException : Exception() {
       object InvalidCredentials : AuthException()
       object NetworkError : AuthException()
       object TokenExpired : AuthException()
   }
   ```

3. **Error Handling Strategy**
   - Use `Result<T>` sealed class or `Either<L, R>` pattern?
   - Where to catch exceptions (Domain vs. Data vs. Presentation)?
   - How to propagate errors to UI (sealed class state)?

   Decision:
   ```kotlin
   // Domain: use Result<T>
   suspend fun login(): Result<AuthToken>

   // Presentation: map to UiState
   sealed class LoginUiState {
       object Loading : LoginUiState()
       data class Success(val token: AuthToken) : LoginUiState()
       data class Error(val message: String) : LoginUiState()
   }

   // ViewModel: catch and map
   fun login() {
       viewModelScope.launch {
           _uiState.value = LoginUiState.Loading
           val result = loginUseCase(...)
           _uiState.value = when {
               result.isSuccess -> LoginUiState.Success(...)
               else -> LoginUiState.Error(...)
           }
       }
   }
   ```

4. **Hilt Module Plan**
   - Which interfaces need @Binds bindings?
   - Which services/DAOs need @Provides?
   - Scope: @Singleton vs. @ViewModelScoped?
   - Module location: @InstallIn(SingletonComponent::class) vs. other?

   Plan:
   ```
   AuthModule.kt (@Module, @InstallIn(SingletonComponent::class))
   ├── @Binds AuthRepository(impl: AuthRepositoryImpl)
   ├── @Provides AuthService(retrofit)
   ├── @Provides AuthDatabase → db.authDao()
   └── @Singleton scope for all

   LoginFeatureModule.kt (@Module, @InstallIn(SingletonComponent::class))
   └── @Provides LoginUseCase(authRepository)
   ```

5. **Navigation Graph Plan**
   - New routes to add (sealed class or data class)?
   - Route arguments (if any)?
   - NavHost entry point (MainActivityor nested graph)?

   Plan:
   ```kotlin
   sealed class Route {
       object AuthFlow : Route()
       object LoginFlow : Route()
       object HomeFlow : Route()
   }

   object AuthRoute {
       object Splash : Route()
       object Login : Route()
   }
   ```

6. **Produce Architecture Decision Record (ADR)**

   ```markdown
   # ADR: Authentication Module Architecture

   ## Problem
   MyApp needs JWT-based authentication with offline caching.

   ## Decision
   Implement Clean Architecture:
   - Domain: AuthRepository interface, LoginUseCase, AuthToken entity
   - Data: AuthRepositoryImpl, AuthService (Retrofit), AuthDao (Room), DTOs + mappers
   - Presentation: LoginViewModel, LoginScreen (Compose), UiState sealed class

   ## Rationale
   - Testable (pure domain logic, mockable data layer)
   - Scalable (clear layer boundaries)
   - Follows project conventions (Clean Architecture already in use)

   ## Error Handling
   - Domain: Result<T> sealed class
   - Presentation: UiState.Error mapped from domain result
   - Catch exceptions in Data layer, propagate as Result

   ## Hilt Modules
   - AuthModule: @Binds AuthRepository(impl)
   - AuthProvidersModule: @Provides AuthService, @Provides AuthDao
   - All @Singleton scope (auth state app-wide)

   ## Navigation
   - New routes: AuthRoute.Splash, AuthRoute.Login (in nested graph)
   - MainActivityNavHost: route from Splash → Login → Home

   ## Acceptance
   - ✓ Domain zero Android imports
   - ✓ All repositories bound
   - ✓ Navigation routes in NavHost
   - ✓ Hilt compilation succeeds
   ```

**Outputs:**
- **Module diagram** (ASCII or visual)
- **Repository/Use Case interface contracts** (Kotlin code)
- **Error handling sealed class** (Kotlin code)
- **Hilt module structure plan** (high-level)
- **Navigation route plan** (Kotlin code sketch)
- **Architecture Decision Record** (Markdown)

**Next Steps:**
- Architect returns blueprint to Coordinator
- Coordinator approves or asks for refinement
- Approve → Domain Lead + Data Lead + Pres Lead proceed in parallel

---

## Phase 2a: Domain Lead (Pure Kotlin — Zero Android Imports)

**Goal:** Generate domain layer code (entities, use cases, repository interfaces) with ZERO Android SDK dependencies.

**Constraint (CRITICAL):**
```
Only allowed imports in domain/:
✓ kotlin.*
✓ kotlin.coroutines (suspend, Flow)
✓ kotlin.serialization (if DTOs in domain)
✓ Custom exceptions (AppException, etc.)
✗ android.* (BANNED)
✗ androidx.* (BANNED)
✗ com.google.* (BANNED, except for annotations if unavoidable)
```

**Detekt Rule:** Any Android import in domain/ → automatic fail + escalate to Domain Lead.

**Inputs:**
- Architect's blueprint (interfaces, error handling, module plan)
- Detailed spec from Product Agent
- Pipeline Memory patterns

**Actions:**

1. **Create Entity Data Classes**

   ```kotlin
   // domain/entity/User.kt
   data class User(
       val id: String,
       val email: String,
       val name: String
   )

   // domain/entity/AuthToken.kt
   data class AuthToken(
       val token: String,
       val refreshToken: String,
       val expiresIn: Long,
       val issuedAt: Long = System.currentTimeMillis()
   )

   // No @Entity, no @Serializable in domain (only in data layer)
   // No Android imports
   ```

2. **Create Exception Hierarchy**

   ```kotlin
   // domain/exception/AppException.kt
   sealed class AppException : Exception() {
       data class InvalidInput(val field: String) : AppException()
       object NetworkError : AppException()
       object TimeoutError : AppException()
       object ServerError : AppException()
       object AuthenticationError : AppException()
       data class Unknown(override val message: String) : AppException()
   }
   ```

3. **Create Repository Interfaces**

   ```kotlin
   // domain/repository/UserRepository.kt
   interface UserRepository {
       suspend fun getUser(id: String): Result<User>
       suspend fun saveUser(user: User): Result<Unit>
       fun watchUser(id: String): Flow<User>
   }

   // domain/repository/AuthRepository.kt
   interface AuthRepository {
       suspend fun login(email: String, password: String): Result<AuthToken>
       suspend fun logout(): Result<Unit>
       suspend fun refreshToken(refreshToken: String): Result<AuthToken>
       suspend fun getCachedToken(): Result<AuthToken?>
   }
   ```

   **Pattern:** Use `suspend fun` for one-time operations, `Flow<T>` for streams, `Result<T>` for error handling.

4. **Create Use Cases**

   ```kotlin
   // domain/usecase/LoginUseCase.kt
   class LoginUseCase @Inject constructor(
       private val authRepository: AuthRepository,
       private val userRepository: UserRepository
   ) {
       suspend operator fun invoke(email: String, password: String): Result<Unit> =
           runCatching {
               val token = authRepository.login(email, password)
                   .getOrElse { throw it }
               // Post-login: fetch user profile
               userRepository.getUser("self")
                   .getOrElse { throw it }
           }
   }

   // domain/usecase/GetUserUseCase.kt
   class GetUserUseCase @Inject constructor(
       private val userRepository: UserRepository
   ) {
       suspend operator fun invoke(id: String): Result<User> =
           userRepository.getUser(id)
   }

   // domain/usecase/WatchUserUseCase.kt
   class WatchUserUseCase @Inject constructor(
       private val userRepository: UserRepository
   ) {
       operator fun invoke(id: String): Flow<User> =
           userRepository.watchUser(id)
   }
   ```

   **Pattern:** `@Inject constructor`, `suspend operator fun invoke(...)`, return `Result<T>` or `Flow<T>`, use `runCatching` to wrap domain logic.

5. **Validate Zero Android Imports**
   - Run detekt with custom rule: no `android.*`, `androidx.*` imports
   - Scan code for any forbidden imports
   - If found, report to Architect

6. **Create Layer Boundary Tests (Optional)**
   - Unit tests for use cases (pure Kotlin, no mocks of repositories)
   - Verify logic without Android context

**Outputs:**
- **Entities** (data classes): User, AuthToken, etc.
- **Exceptions** (sealed class): AppException + variants
- **Repository Interfaces**: UserRepository, AuthRepository, etc.
- **Use Cases** (classes with @Inject): LoginUseCase, GetUserUseCase, WatchUserUseCase, etc.
- **Validation Report**: "Zero Android imports ✓"

**Feedback Route (on error):**
- **Android import found** → escalate to Architect (design violation)
- **Logic error** → stay with Domain Lead (refine use case logic)

**Next Step:**
- Domain Lead returns code to Coordinator
- Code waits for Data Lead + Pres Lead to complete Phase 2b (parallel execution)

---

## Phase 2b: Data Lead (Retrofit + Room + Mappers)

**Goal:** Generate data layer (DTOs, mappers, Retrofit services, Room DAOs, repository implementations, Hilt bindings).

**Parallel Execution:** Simultaneous with Presentation Lead.

**Inputs:**
- Architect's blueprint
- Domain layer code (entities, repository interfaces, use cases)
- Detailed spec with API contracts
- Pipeline Memory patterns (e.g., "@Serializable + @SerialName")

**Actions:**

1. **Create DTOs (Data Transfer Objects)**

   ```kotlin
   // data/dto/LoginRequestDto.kt
   @Serializable
   data class LoginRequestDto(
       val email: String,
       val password: String
   )

   // data/dto/LoginResponseDto.kt
   @Serializable
   data class LoginResponseDto(
       @SerialName("access_token")
       val token: String,
       @SerialName("refresh_token")
       val refreshToken: String,
       @SerialName("expires_in")
       val expiresIn: Long,
       val user: UserDto
   )

   // data/dto/UserDto.kt
   @Serializable
   data class UserDto(
       val id: String,
       val email: String,
       val name: String
   )
   ```

   **Pattern:** `@Serializable` annotation (Kotlinx), `@SerialName` for JSON field mapping, match API response structure.

2. **Create Mappers (DTO ↔ Domain)**

   ```kotlin
   // data/mapper/UserMapper.kt
   fun UserDto.toDomain(): User =
       User(id = id, email = email, name = name)

   fun User.toDto(): UserDto =
       UserDto(id = id, email = email, name = name)

   // data/mapper/AuthTokenMapper.kt
   fun LoginResponseDto.toAuthToken(): AuthToken =
       AuthToken(token = token, refreshToken = refreshToken, expiresIn = expiresIn)
   ```

   **Pattern:** Extension functions on DTOs, pure mapping logic (no side effects).

3. **Create Retrofit Services**

   ```kotlin
   // data/network/AuthService.kt
   interface AuthService {
       @POST("auth/login")
       suspend fun login(@Body request: LoginRequestDto): LoginResponseDto

       @POST("auth/refresh")
       suspend fun refreshToken(@Body request: RefreshTokenRequestDto): LoginResponseDto

       @POST("auth/logout")
       suspend fun logout(): Unit
   }

   // data/network/UserService.kt
   interface UserService {
       @GET("users/{id}")
       suspend fun getUser(@Path("id") id: String): UserDto

       @GET("users/me")
       suspend fun getSelf(): UserDto
   }
   ```

   **Pattern:** `@GET`, `@POST`, `@Body`, `@Path`, suspend functions, return DTOs.

4. **Create Room Entities + DAOs**

   ```kotlin
   // data/local/UserEntity.kt
   @Entity(tableName = "users")
   data class UserEntity(
       @PrimaryKey val id: String,
       val email: String,
       val name: String,
       val cachedAt: Long
   )

   // data/local/UserDao.kt
   @Dao
   interface UserDao {
       @Query("SELECT * FROM users WHERE id = :id")
       suspend fun getUser(id: String): UserEntity?

       @Insert(onConflict = OnConflictStrategy.REPLACE)
       suspend fun insert(user: UserEntity)

       @Query("SELECT * FROM users WHERE id = :id")
       fun watchUser(id: String): Flow<UserEntity>

       @Query("DELETE FROM users")
       suspend fun deleteAll()
   }

   // data/local/AppDatabase.kt
   @Database(
       entities = [UserEntity::class],
       version = 1,
       exportSchema = false
   )
   abstract class AppDatabase : RoomDatabase() {
       abstract fun userDao(): UserDao
   }
   ```

   **Pattern:** `@Entity`, `@Dao`, `@Query`, suspend for one-time, `Flow<T>` for streams, `@Insert(onConflict = REPLACE)` for upserts.

5. **Create Repository Implementations**

   ```kotlin
   // data/repository/AuthRepositoryImpl.kt
   @Singleton
   class AuthRepositoryImpl @Inject constructor(
       private val authService: AuthService,
       private val userService: UserService,
       private val tokenCache: TokenCache, // SharedPrefs wrapper (provided by Hilt)
       private val userDao: UserDao
   ) : AuthRepository {
       override suspend fun login(email: String, password: String): Result<AuthToken> =
           runCatching {
               val request = LoginRequestDto(email, password)
               val response = authService.login(request)
               val token = response.toAuthToken()
               tokenCache.save(token)
               response.user.toDomain().let { user ->
                   userDao.insert(UserEntity(user.id, user.email, user.name, System.currentTimeMillis()))
               }
               token
           }

       override suspend fun logout(): Result<Unit> =
           runCatching {
               authService.logout()
               tokenCache.clear()
               userDao.deleteAll()
           }

       override suspend fun refreshToken(refreshToken: String): Result<AuthToken> =
           runCatching {
               val request = RefreshTokenRequestDto(refreshToken)
               val response = authService.refreshToken(request)
               val token = response.toAuthToken()
               tokenCache.save(token)
               token
           }

       override suspend fun getCachedToken(): Result<AuthToken?> =
           runCatching {
               tokenCache.get()
           }
   }

   // data/repository/UserRepositoryImpl.kt
   @Singleton
   class UserRepositoryImpl @Inject constructor(
       private val userService: UserService,
       private val userDao: UserDao
   ) : UserRepository {
       override suspend fun getUser(id: String): Result<User> =
           runCatching {
               val dto = userService.getUser(id)
               val user = dto.toDomain()
               userDao.insert(UserEntity(user.id, user.email, user.name, System.currentTimeMillis()))
               user
           }

       override suspend fun saveUser(user: User): Result<Unit> =
           runCatching {
               userDao.insert(UserEntity(user.id, user.email, user.name, System.currentTimeMillis()))
           }

       override fun watchUser(id: String): Flow<User> =
           userDao.watchUser(id).map { entity ->
               User(entity.id, entity.email, entity.name)
           }
   }
   ```

   **Pattern:** `@Singleton`, `@Inject constructor`, implement repository interface, call service + DAO, catch errors via `runCatching`, map DTOs to domain entities.

6. **Create Hilt @Binds Modules**

   ```kotlin
   // di/AuthModule.kt
   @Module
   @InstallIn(SingletonComponent::class)
   interface AuthModule {
       @Binds
       @Singleton
       fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository
   }

   // di/UserModule.kt
   @Module
   @InstallIn(SingletonComponent::class)
   interface UserModule {
       @Binds
       @Singleton
       fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
   }
   ```

   **Pattern:** `@Module`, `@InstallIn(SingletonComponent::class)`, `@Binds` for interface → impl mapping, `@Singleton` scope.

7. **Create Hilt @Provides Modules for Services/DAOs**

   ```kotlin
   // di/NetworkModule.kt
   @Module
   @InstallIn(SingletonComponent::class)
   object NetworkModule {
       @Provides
       @Singleton
       fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit =
           Retrofit.Builder()
               .baseUrl("https://api.example.com/")
               .client(okHttpClient)
               .addConverterFactory(Json.asConverterFactory("application/json".toMediaType()))
               .build()

       @Provides
       @Singleton
       fun provideAuthService(retrofit: Retrofit): AuthService =
           retrofit.create(AuthService::class.java)

       @Provides
       @Singleton
       fun provideUserService(retrofit: Retrofit): UserService =
           retrofit.create(UserService::class.java)

       @Provides
       @Singleton
       fun provideOkHttpClient(): OkHttpClient =
           OkHttpClient.Builder()
               .connectTimeout(30, TimeUnit.SECONDS)
               .readTimeout(30, TimeUnit.SECONDS)
               .addInterceptor(AuthInterceptor()) // Adds JWT to requests
               .build()
   }

   // di/DatabaseModule.kt
   @Module
   @InstallIn(SingletonComponent::class)
   object DatabaseModule {
       @Provides
       @Singleton
       fun provideAppDatabase(@ApplicationContext context: Context): AppDatabase =
           Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
               .build()

       @Provides
       @Singleton
       fun provideUserDao(database: AppDatabase): UserDao =
           database.userDao()
   }

   // di/CacheModule.kt
   @Module
   @InstallIn(SingletonComponent::class)
   object CacheModule {
       @Provides
       @Singleton
       fun provideTokenCache(@ApplicationContext context: Context): TokenCache =
           SharedPrefsTokenCache(context)
   }
   ```

   **Pattern:** `@Module`, `@InstallIn(SingletonComponent::class)`, `@Provides` for complex objects, `@Singleton` scope, inject dependencies (Retrofit, Context, etc.).

**Outputs:**
- **DTOs** (@Serializable): LoginRequestDto, LoginResponseDto, UserDto, etc.
- **Mappers**: DTO → Domain entity extension functions
- **Retrofit Services**: AuthService, UserService, etc.
- **Room Entities + DAOs**: UserEntity, UserDao, AppDatabase
- **Repository Implementations**: AuthRepositoryImpl, UserRepositoryImpl
- **Hilt @Binds + @Provides Modules**: AuthModule, NetworkModule, DatabaseModule, CacheModule

**Feedback Route (on error):**
- **DTO serialization error** → stay with Data Lead (fix @Serializable)
- **Retrofit compilation error** → stay with Data Lead (check API contracts)
- **Room migration error** → stay with Data Lead (schema version)
- **Hilt binding missing** → escalate to Integration (module assembly issue)

**Next Step:**
- Data Lead returns code to Coordinator
- Code waits for Pres Lead to complete Phase 2b (parallel execution)

---

## Phase 2b: Presentation Lead (Compose + ViewModel + Navigation)

**Goal:** Generate presentation layer (ViewModels, Compose screens, Navigation, UiState sealed classes).

**Parallel Execution:** Simultaneous with Data Lead.

**Inputs:**
- Architect's blueprint
- Domain layer code (use cases, entities)
- Detailed spec with screen mockups
- Pipeline Memory patterns (ViewModel + UiState patterns)

**Actions:**

1. **Create UiState Sealed Classes**

   ```kotlin
   // presentation/feature/login/LoginUiState.kt
   sealed class LoginUiState {
       object Idle : LoginUiState()
       object Loading : LoginUiState()
       data class Success(val token: AuthToken) : LoginUiState()
       data class Error(val message: String) : LoginUiState()
   }

   // presentation/feature/user/UserUiState.kt
   sealed class UserUiState {
       object Loading : UserUiState()
       data class Success(val user: User) : UserUiState()
       data class Error(val message: String) : UserUiState()
   }
   ```

   **Pattern:** Sealed class with object/data variants (Loading, Success, Error, Idle). One state class per screen.

2. **Create ViewModels**

   ```kotlin
   // presentation/feature/login/LoginViewModel.kt
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

   // presentation/feature/user/UserViewModel.kt
   @HiltViewModel
   class UserViewModel @Inject constructor(
       private val getUserUseCase: GetUserUseCase,
       private val watchUserUseCase: WatchUserUseCase,
       savedStateHandle: SavedStateHandle
   ) : ViewModel() {
       private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
       val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

       init {
           viewModelScope.launch {
               val userId = savedStateHandle.get<String>("userId") ?: return@launch
               val result = getUserUseCase(userId)
               _uiState.value = when {
                   result.isSuccess -> UserUiState.Success(result.getOrNull()!!)
                   else -> UserUiState.Error(result.exceptionOrNull()?.message ?: "Unknown error")
               }
           }
       }
   }
   ```

   **Pattern:**
   - `@HiltViewModel`
   - `@Inject constructor` (inject use cases)
   - `MutableStateFlow` (private, mutable)
   - `asStateFlow()` (expose immutable)
   - `viewModelScope.launch { }` (lifecycle-aware coroutines)
   - Emit states as `_uiState.value = ...`
   - Use `SavedStateHandle` for navigation arguments

3. **Create Compose Screens**

   ```kotlin
   // presentation/feature/login/LoginScreen.kt
   @Composable
   fun LoginScreen(
       viewModel: LoginViewModel = hiltViewModel(),
       onNavigateToHome: () -> Unit
   ) {
       val uiState by viewModel.uiState.collectAsStateWithLifecycle()
       var email by remember { mutableStateOf("") }
       var password by remember { mutableStateOf("") }

       Box(modifier = Modifier.fillMaxSize()) {
           when (uiState) {
               is LoginUiState.Idle -> {
                   Column(
                       modifier = Modifier
                           .align(Alignment.Center)
                           .padding(16.dp)
                   ) {
                       OutlinedTextField(
                           value = email,
                           onValueChange = { email = it },
                           label = { Text("Email") },
                           modifier = Modifier.fillMaxWidth()
                       )
                       Spacer(modifier = Modifier.height(8.dp))
                       OutlinedTextField(
                           value = password,
                           onValueChange = { password = it },
                           label = { Text("Password") },
                           visualTransformation = PasswordVisualTransformation(),
                           modifier = Modifier.fillMaxWidth()
                       )
                       Spacer(modifier = Modifier.height(16.dp))
                       Button(
                           onClick = { viewModel.login(email, password) },
                           modifier = Modifier.fillMaxWidth()
                       ) {
                           Text("Login")
                       }
                   }
               }
               is LoginUiState.Loading -> {
                   CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
               }
               is LoginUiState.Success -> {
                   LaunchedEffect(Unit) {
                       onNavigateToHome()
                   }
               }
               is LoginUiState.Error -> {
                   Text(
                       text = (uiState as LoginUiState.Error).message,
                       color = MaterialTheme.colorScheme.error,
                       modifier = Modifier.align(Alignment.Center)
                   )
               }
           }
       }
   }

   // presentation/feature/user/UserScreen.kt
   @Composable
   fun UserScreen(
       viewModel: UserViewModel = hiltViewModel()
   ) {
       val uiState by viewModel.uiState.collectAsStateWithLifecycle()

       when (uiState) {
           is UserUiState.Loading -> {
               Box(modifier = Modifier.fillMaxSize()) {
                   CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
               }
           }
           is UserUiState.Success -> {
               val user = (uiState as UserUiState.Success).user
               Column(modifier = Modifier.padding(16.dp)) {
                   Text("Name: ${user.name}", style = MaterialTheme.typography.headlineSmall)
                   Text("Email: ${user.email}")
               }
           }
           is UserUiState.Error -> {
               Box(modifier = Modifier.fillMaxSize()) {
                   Text(
                       text = (uiState as UserUiState.Error).message,
                       color = MaterialTheme.colorScheme.error,
                       modifier = Modifier.align(Alignment.Center)
                   )
               }
           }
       }
   }
   ```

   **Pattern:**
   - `@Composable` function
   - `hiltViewModel()` injection
   - `collectAsStateWithLifecycle()` for state collection
   - `when(uiState)` for state rendering
   - Material Design 3 components (OutlinedTextField, Button, Text, etc.)
   - `Modifier` chains for layout
   - `LaunchedEffect` for side effects (navigation)

4. **Create Navigation Routes (Type-Safe)**

   ```kotlin
   // presentation/navigation/Route.kt
   sealed class Route {
       object AuthFlow : Route()
       object MainFlow : Route()
   }

   object AuthRoute {
       object Login : AuthRoute()
       object Splash : AuthRoute()
   }

   object MainRoute {
       object Home : MainRoute()
       data class UserDetail(val userId: String) : MainRoute()
   }
   ```

   **Pattern:** Sealed classes for type-safe routing. Nested objects/data classes per flow.

5. **Create NavHost (in MainActivity or App Composable)**

   ```kotlin
   // presentation/MainActivity.kt
   @AndroidEntryPoint
   class MainActivity : ComponentActivity() {
       override fun onCreate(savedInstanceState: Bundle?) {
           super.onCreate(savedInstanceState)
           setContent {
               AppTheme {
                   val navController = rememberNavController()
                   NavHost(navController, startDestination = Route.AuthFlow) {
                       navigation<Route.AuthFlow>(startDestination = AuthRoute.Splash) {
                           composable<AuthRoute.Splash> {
                               SplashScreen(
                                   onNavigateToLogin = {
                                       navController.navigate(AuthRoute.Login)
                                   },
                                   onNavigateToHome = {
                                       navController.navigate(Route.MainFlow) {
                                           popUpTo(Route.AuthFlow) { inclusive = true }
                                       }
                                   }
                               )
                           }
                           composable<AuthRoute.Login> {
                               LoginScreen(
                                   onNavigateToHome = {
                                       navController.navigate(Route.MainFlow) {
                                           popUpTo(Route.AuthFlow) { inclusive = true }
                                       }
                                   }
                               )
                           }
                       }

                       navigation<Route.MainFlow>(startDestination = MainRoute.Home) {
                           composable<MainRoute.Home> {
                               HomeScreen()
                           }
                           composable<MainRoute.UserDetail> { backStackEntry ->
                               val userId = backStackEntry.arguments?.getString("userId") ?: return@composable
                               UserDetailScreen(userId = userId)
                           }
                       }
                   }
               }
           }
       }
   }

   // presentation/theme/AppTheme.kt
   @Composable
   fun AppTheme(content: @Composable () -> Unit) {
       MaterialTheme(
           colorScheme = dynamicLightColorScheme(LocalContext.current),
           typography = Typography(),
           shapes = Shapes(),
           content = content
       )
   }
   ```

   **Pattern:**
   - `NavHost` with `startDestination`
   - `navigation` for nested graphs (AuthFlow, MainFlow)
   - `composable` for individual routes
   - `popUpTo(...) { inclusive = true }` for back stack management
   - Material Design 3 theme wrapper

6. **Create Compose Preview Tests (Optional)**

   ```kotlin
   @Preview
   @Composable
   fun LoginScreenPreview() {
       AppTheme {
           LoginScreen(onNavigateToHome = {})
       }
   }

   @Preview
   @Composable
   fun UserScreenPreview() {
       AppTheme {
           UserScreen()
       }
   }
   ```

**Outputs:**
- **UiState Sealed Classes**: LoginUiState, UserUiState, etc.
- **ViewModels** (@HiltViewModel): LoginViewModel, UserViewModel, etc.
- **Compose Screens**: LoginScreen, UserScreen, HomeScreen, etc.
- **Navigation Routes** (sealed class): AuthRoute, MainRoute, etc.
- **NavHost** (in MainActivity): Full navigation graph with nested flows
- **Theme** (AppTheme): Material Design 3 theming

**Feedback Route (on error):**
- **ViewModel test fails** → stay with Pres Lead (fix StateFlow logic)
- **Compose compilation error** → stay with Pres Lead (check composable signature)
- **Navigation crash** → stay with Pres Lead (check route definition)
- **Hilt @HiltViewModel injection fails** → escalate to Integration (binding issue)

**Next Step:**
- Pres Lead returns code to Coordinator
- Code waits for Data Lead to complete Phase 2b (parallel execution)

---

## Phase 3: Integration (Assembly & Validation)

**Goal:** Assemble Hilt modules, validate layer boundaries, compile check, prepare for testing.

**Sequential Execution:** After Phase 2a + Phase 2b complete.

**Inputs:**
- Domain layer code (entities, use cases, repository interfaces)
- Data layer code (DTOs, mappers, services, DAOs, repository implementations, @Binds modules)
- Presentation layer code (ViewModels, screens, navigation, UiState)
- Architect's blueprint

**Actions:**

1. **Assemble All Hilt Modules**

   ```kotlin
   // di/AppModule.kt (main assembly)
   @Module
   @InstallIn(SingletonComponent::class)
   class AppModule {
       // No explicit bindings here, but imports all sub-modules via @Module annotations
   }

   // Included modules (via @Module in their respective files):
   // - AuthModule (@Binds AuthRepository)
   // - UserModule (@Binds UserRepository)
   // - NetworkModule (@Provides Retrofit, AuthService, UserService, OkHttpClient)
   // - DatabaseModule (@Provides AppDatabase, UserDao)
   // - CacheModule (@Provides TokenCache)
   ```

   **Verification:**
   - All repository interfaces have @Binds or @Provides
   - All @Module classes use @InstallIn(SingletonComponent::class)
   - No circular dependencies
   - All scopes are @Singleton (no ViewModel-scoped unless necessary)

2. **Validate Layer Boundaries**

   ```
   ✓ Domain layer (domain/):
     - No android.* imports
     - No androidx.* imports
     - Only Kotlin, kotlin.coroutines, custom exceptions

   ✓ Data layer (data/):
     - No presentation.* imports
     - Can import domain.* freely
     - Has android.* (Room, Retrofit), androidx.* (Room)

   ✓ Presentation layer (presentation/):
     - No data.* imports (except mappers if co-located with viewmodels)
     - Can import domain.* freely
     - Has androidx.compose, androidx.viewmodel
   ```

   **Detekt Rule Check:**
   ```bash
   ./gradlew detekt
   ```
   Expected: Zero layer violations.

3. **Verify ViewModel + NavHost Integration**

   ```
   ✓ Every ViewModel:
     - Annotated with @HiltViewModel
     - Has @Inject constructor
     - Injects only Use Cases (domain layer)
     - Has UiState sealed class
     - Uses MutableStateFlow + asStateFlow()
     - Uses viewModelScope.launch { }

   ✓ NavHost:
     - All routes defined as sealed class properties
     - All routes appear in composable<Route> blocks
     - Navigation callbacks pass route to navController.navigate()
     - Back stack management uses popUpTo() + inclusive = true
   ```

4. **Compile Check (Dry Run)**

   ```bash
   ./gradlew build --dry-run
   ```

   Expected: No compile errors (just a dry run to catch syntax errors early).

5. **Run ktlint + detekt**

   ```bash
   ./gradlew ktlintFormat
   ./gradlew detekt
   ```

   Expected: Zero violations after ktlintFormat.

6. **Generate Integration Report**

   ```markdown
   # Integration Report

   ## Hilt Bindings
   ✓ AuthRepository: @Binds AuthRepositoryImpl
   ✓ UserRepository: @Binds UserRepositoryImpl
   ✓ AuthService: @Provides (NetworkModule)
   ✓ UserService: @Provides (NetworkModule)
   ✓ AppDatabase: @Provides (DatabaseModule)
   ✓ UserDao: @Provides (DatabaseModule)
   ✓ TokenCache: @Provides (CacheModule)

   ## Layer Boundaries
   ✓ Domain: 0 Android imports
   ✓ Data: No presentation imports
   ✓ Presentation: No data imports

   ## ViewModels & Navigation
   ✓ LoginViewModel: @HiltViewModel, injects LoginUseCase
   ✓ UserViewModel: @HiltViewModel, injects GetUserUseCase, WatchUserUseCase
   ✓ NavHost: AuthRoute.Login, AuthRoute.Splash, MainRoute.Home, MainRoute.UserDetail

   ## Code Quality
   ✓ ktlint: 0 violations
   ✓ detekt: 0 violations (0 complexity > 15, 0 naming violations, 0 layer violations)

   ## Compile Check
   ✓ ./gradlew build --dry-run: SUCCESS

   ---

   **Status:** ✓ READY FOR TESTING
   Next: Test Agent + Lint Agent (parallel)
   ```

**Outputs:**
- **Integrated codebase** (all layers assembled)
- **Hilt module assembly** (verified all @Binds + @Provides)
- **Layer boundary validation** (zero violations)
- **ktlint + detekt** (auto-format + analysis)
- **Integration report** (summary of checks)

**Feedback Routing (on error):**
- **Hilt missing binding** → Architect re-review + correction
- **Layer violation** → violating lead (Domain, Data, or Pres) re-review
- **ViewModel test fails** → Pres Lead (fix @HiltViewModel injection)
- **Compile error** → specific layer lead (syntax error in generated code)
- **ktlint/detekt violations** → Integration runs auto-format + re-checks

**Next Step:**
- Integration returns verified, formatted codebase to Coordinator
- Coordinator triggers Test Agent ‖ Lint Agent (parallel)

---

## Phase 4 Summary: From Blueprint to Integrated Codebase

```
Architect (Phase 1)
  ↓
  ├─→ Domain Lead (Phase 2a) — entities, use cases, repo interfaces
  │
  ├─→ Data Lead (Phase 2b) — DTOs, mappers, services, DAOs, repo impls, Hilt @Binds
  │   (parallel)
  │
  └─→ Pres Lead (Phase 2b) — ViewModels, screens, navigation, UiState
      (parallel)
         ↓
      Integration (Phase 3) — assemble Hilt, validate layers, compile check
         ↓
      READY FOR TEST ‖ LINT
```

Each phase produces code that is:
- ✓ Consistent with architecture
- ✓ Following Material Design 3 (Pres) + Jetpack conventions
- ✓ Testable (layers separate, interfaces clear)
- ✓ Injected via Hilt (@Inject, @Binds, @Provides)
- ✓ Formatted (ktlint) + analyzed (detekt)
- ✓ Compiled successfully

**Duration:** 30–60 minutes (4 phases: 1 sequential + 2 parallel + 1 sequential).
