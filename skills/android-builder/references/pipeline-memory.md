# Pipeline Memory

## Overview

**Pipeline Memory** is a cross-run learning system that captures successful patterns, error resolutions, and naming conventions. It accelerates future pipeline executions by providing agents with learned practices from prior runs.

## Memory Storage Format

Pipeline Memory is stored as JSON at `pipeline_memory.json`:

```json
{
  "version": "1.0",
  "created_date": "2025-02-11",
  "last_updated": "2025-02-11",
  "patterns": [
    {
      "pattern_id": "serializable_annotation",
      "category": "data_layer",
      "description": "DTOs use @Serializable + @SerialName for JSON field mapping",
      "context": "auth feature, LoginDto, UserDto",
      "example": "@Serializable\ndata class LoginDto(\n  @SerialName(\"user_id\")\n  val userId: String\n)",
      "frequency": 4,
      "date_first_observed": "2025-02-10",
      "date_last_used": "2025-02-11"
    },
    {
      "pattern_id": "viewmodel_uistate",
      "category": "presentation_layer",
      "description": "Every ViewModel: @HiltViewModel + MutableStateFlow + UiState sealed class + viewModelScope.launch",
      "context": "LoginViewModel, UserViewModel, ProfileViewModel",
      "example": "@HiltViewModel\nclass LoginViewModel @Inject constructor(...) : ViewModel() {\n  private val _uiState = MutableStateFlow<LoginUiState>(LoginUiState.Idle)\n  val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()\n  fun login(...) {\n    viewModelScope.launch { ... }\n  }\n}",
      "frequency": 5,
      "date_first_observed": "2025-02-08",
      "date_last_used": "2025-02-11"
    },
    {
      "pattern_id": "error_handling_result",
      "category": "domain_layer",
      "description": "Error handling uses Result<T> sealed class in domain functions, try-catch in ViewModel",
      "context": "LoginUseCase, GetUserUseCase, error handling",
      "example": "// Domain: suspend fun execute(...): Result<AuthToken>\n// Data: runCatching { ... }\n// Presentation: when { result.isSuccess -> ..., else -> ... }",
      "frequency": 3,
      "date_first_observed": "2025-02-09",
      "date_last_used": "2025-02-11"
    },
    {
      "pattern_id": "navigation_typesafe",
      "category": "presentation_layer",
      "description": "Navigation uses type-safe sealed class routes, NavHost in MainActivity, type-safe transitions",
      "context": "AuthRoute.Login, MainRoute.Home, navigation graph",
      "example": "sealed class AuthRoute {\n  object Login : AuthRoute()\n  object Splash : AuthRoute()\n}\n\nNavHost(navController, startDestination = AuthRoute.Splash) {\n  composable<AuthRoute.Login> { LoginScreen(...) }\n}",
      "frequency": 3,
      "date_first_observed": "2025-02-08",
      "date_last_used": "2025-02-11"
    },
    {
      "pattern_id": "hilt_binds_binding",
      "category": "di",
      "description": "@Module + @InstallIn(SingletonComponent::class) with @Binds for repository interfaces",
      "context": "AuthModule, UserModule, RepositoryModule",
      "example": "@Module\n@InstallIn(SingletonComponent::class)\ninterface AuthModule {\n  @Binds\n  @Singleton\n  fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository\n}",
      "frequency": 2,
      "date_first_observed": "2025-02-10",
      "date_last_used": "2025-02-11"
    },
    {
      "pattern_id": "domain_pure_kotlin",
      "category": "domain_layer",
      "description": "Domain layer: zero Android imports, only Kotlin stdlib + coroutines + custom exceptions",
      "context": "domain/entity, domain/usecase, domain/repository",
      "example": "// Allowed: kotlin.*, kotlin.coroutines.*, custom exceptions\n// BANNED: android.*, androidx.*",
      "frequency": 4,
      "date_first_observed": "2025-02-08",
      "date_last_used": "2025-02-11"
    }
  ],
  "error_patterns": [
    {
      "error_id": "android_import_in_domain",
      "description": "Android import detected in domain layer (android.app.*, androidx.*)",
      "example": "import android.util.Log (found in domain/usecase/LoginUseCase.kt)",
      "solution": "Move logging or Android-dependent code to Data or Presentation layer",
      "frequency": 2,
      "resolution_time_minutes": 5,
      "severity": "HIGH"
    },
    {
      "error_id": "hilt_binding_missing",
      "description": "Hilt binding missing for repository interface (no @Binds or @Provides)",
      "example": "AuthRepositoryImpl injected but AuthRepository interface not bound",
      "solution": "Add @Binds or @Provides in @Module for repository interface",
      "frequency": 1,
      "resolution_time_minutes": 3,
      "severity": "HIGH"
    },
    {
      "error_id": "viewmodel_not_collecting_properly",
      "description": "Screen collects StateFlow with .collect() instead of .collectAsStateWithLifecycle()",
      "example": "LoginScreen uses viewModel.uiState.collect() → memory leak on screen rotation",
      "solution": "Use .collectAsStateWithLifecycle() in Compose screens",
      "frequency": 1,
      "resolution_time_minutes": 3,
      "severity": "MEDIUM"
    },
    {
      "error_id": "room_migration_version_mismatch",
      "description": "Room database version incremented but migration not provided",
      "example": "UserEntity schema changed, version 1 → 2, but no migration block",
      "solution": "Create @Migration(from = 1, to = 2) or provide fallback destructive migration",
      "frequency": 1,
      "resolution_time_minutes": 10,
      "severity": "HIGH"
    },
    {
      "error_id": "detekt_naming_violation",
      "description": "Function name contains underscores (detekt FunctionNamingRules)",
      "example": "fun get_user() instead of fun getUser()",
      "solution": "Rename to camelCase: getUser(), loginUser(), saveUser()",
      "frequency": 2,
      "resolution_time_minutes": 2,
      "severity": "LOW"
    }
  ],
  "naming_conventions": {
    "data_transfer_objects": {
      "pattern": "[Entity]Dto",
      "examples": ["UserDto", "LoginDto", "AuthTokenDto"],
      "location": "data/dto/"
    },
    "room_entities": {
      "pattern": "[Entity]Entity",
      "examples": ["UserEntity", "AuthTokenEntity"],
      "location": "data/local/"
    },
    "view_models": {
      "pattern": "[Feature]ViewModel",
      "examples": ["LoginViewModel", "UserViewModel", "ProfileViewModel"],
      "location": "presentation/feature/"
    },
    "compose_screens": {
      "pattern": "[Feature]Screen",
      "examples": ["LoginScreen", "UserScreen", "ProfileScreen", "HomeScreen"],
      "location": "presentation/feature/"
    },
    "ui_state_sealed_classes": {
      "pattern": "[Feature]UiState",
      "examples": ["LoginUiState", "UserUiState", "ProfileUiState"],
      "location": "presentation/feature/"
    },
    "use_cases": {
      "pattern": "[Action][Entity]UseCase",
      "examples": ["LoginUseCase", "GetUserUseCase", "UpdateProfileUseCase"],
      "location": "domain/usecase/"
    },
    "repositories": {
      "pattern": "[Entity]Repository",
      "examples": ["UserRepository", "AuthRepository", "ProfileRepository"],
      "location": "domain/repository/"
    },
    "repository_implementations": {
      "pattern": "[Entity]RepositoryImpl",
      "examples": ["UserRepositoryImpl", "AuthRepositoryImpl"],
      "location": "data/repository/"
    },
    "hilt_modules": {
      "pattern": "[Entity]Module or [Entity]ProvidersModule",
      "examples": ["AuthModule", "UserModule", "NetworkModule", "DatabaseModule"],
      "location": "di/"
    }
  },
  "architecture_decisions": [
    {
      "decision": "Clean Architecture (Domain, Data, Presentation layers)",
      "rationale": "Testability, scalability, clear separation of concerns",
      "date_adopted": "2025-02-08",
      "projects_using": 3
    },
    {
      "decision": "Hilt for dependency injection (@HiltViewModel, @Binds, @Provides)",
      "rationale": "Android standard, Jetpack integration, compile-time safety",
      "date_adopted": "2025-02-08",
      "projects_using": 3
    },
    {
      "decision": "Jetpack Compose for UI (no XML layouts)",
      "rationale": "Modern, declarative, Material Design 3 native",
      "date_adopted": "2025-02-08",
      "projects_using": 3
    },
    {
      "decision": "Result<T> sealed class for error handling in domain layer",
      "rationale": "Type-safe, explicitly models success/failure",
      "date_adopted": "2025-02-09",
      "projects_using": 2
    },
    {
      "decision": "StateFlow + UiState sealed class for screen state management",
      "rationale": "Unidirectional data flow, lifecycle-aware, predictable updates",
      "date_adopted": "2025-02-08",
      "projects_using": 3
    }
  ],
  "library_preferences": [
    {
      "category": "http_client",
      "preferred": "Retrofit 2.10+",
      "rationale": "Mature, well-integrated with Kotlin, coroutine support",
      "version_pinned": "2.10.0",
      "projects_using": 3
    },
    {
      "category": "persistence_database",
      "preferred": "Room 2.5+",
      "rationale": "Type-safe SQL, Kotlin integration, Flow support",
      "version_pinned": "2.5.2",
      "projects_using": 3
    },
    {
      "category": "serialization",
      "preferred": "Kotlinx Serialization",
      "rationale": "Kotlin-native, supports sealed classes, no reflection",
      "version_pinned": "1.5.1",
      "projects_using": 2
    },
    {
      "category": "testing_unit",
      "preferred": "JUnit5 + MockK",
      "rationale": "Modern, flexible mocking, Kotlin-first",
      "projects_using": 3
    },
    {
      "category": "testing_ui",
      "preferred": "Espresso + Compose Testing",
      "rationale": "Android standard, native Compose support",
      "projects_using": 2
    }
  ],
  "performance_patterns": [
    {
      "pattern": "Room offline-first caching with Retrofit",
      "description": "Save API response to Room, fetch from network on error",
      "use_case": "Data that needs offline access",
      "implementation": "Repository: fetch network → save to Room → emit from DAO. On network error: fetch from Room cache.",
      "date_observed": "2025-02-10"
    },
    {
      "pattern": "Flow for watching database changes",
      "description": "Use Room @Query returning Flow<T> for reactive screen updates",
      "use_case": "Real-time UI updates (e.g., user profile changes)",
      "implementation": "DAO: @Query returns Flow<UserEntity>, WatchUserUseCase returns Flow<User>",
      "date_observed": "2025-02-10"
    },
    {
      "pattern": "Lazy column pagination",
      "description": "LazyColumn with LazyPagingItems for large datasets",
      "use_case": "Lists of 100+ items",
      "implementation": "ViewModel: Pager + PagingData + Compose LazyColumn",
      "date_observed": "2025-02-11"
    }
  ],
  "testing_patterns": [
    {
      "layer": "domain",
      "pattern": "Unit tests, mock repositories, pure Kotlin",
      "target_coverage": "80%+",
      "frameworks": ["JUnit5", "MockK"],
      "example": "GetUserUseCaseTest: mock UserRepository, assert use case logic"
    },
    {
      "layer": "data",
      "pattern": "Unit tests, mock Retrofit/Room, verify mapping",
      "target_coverage": "70%+",
      "frameworks": ["JUnit5", "MockK"],
      "example": "UserRepositoryImplTest: mock UserService + UserDao, verify insert + mapping"
    },
    {
      "layer": "presentation",
      "pattern": "Unit tests (ViewModel state), UI tests (screen rendering)",
      "target_coverage": "60%+",
      "frameworks": ["JUnit5", "MockK", "Espresso", "Compose Testing"],
      "example": "LoginViewModelTest: mock LoginUseCase, verify state emissions. LoginScreenTest: render screen, assert UI elements."
    }
  ]
}
```

## Memory Loading & Usage

**When:** Bootstrap phase, when Coordinator receives a new request.

**How Bootstrap Uses Memory:**

```
Bootstrap scans project:
  ✓ Detected: Clean Architecture
  ✓ Detected: Jetpack Compose, Hilt, Room, Retrofit

✓ Load Pipeline Memory:
  - Found 6 patterns (Serializable annotation, ViewModel pattern, etc.)
  - Found 5 error patterns (Android import in domain, Hilt binding missing, etc.)
  - Found naming conventions (Dto, Entity, ViewModel, Screen, UiState)
  - Found architecture decisions (Clean Architecture, Hilt, Compose)

Bootstrap displays:
  📚 Pipeline Memory (from 3 prior successful runs):
  • DTOs use @Serializable + @SerialName for JSON field mapping
  • Every ViewModel: @HiltViewModel, MutableStateFlow, UiState sealed class, viewModelScope.launch
  • Error handling: Result<T> in domain, try-catch in ViewModel
  • Navigation: type-safe routes (sealed class), NavHost in MainActivity
  • Hilt: @Module + @InstallIn(SingletonComponent), @Binds for repos, @Singleton scope
  • Domain: zero Android imports (Kotlin only)

✓ Project Context loaded:
  - 12 API endpoints across 3 Retrofit services
  - 8 reusable Composables in DesignSystem
  - 14 Gradle dependencies tracked
  - 4 feature modules + 2 infrastructure modules
  - 6 domain entities with relationships
  - Naming conventions established

These patterns will guide Code Gen Team for consistency.
```

**How Agents Use Memory:**

### Architect Uses Memory
- "Based on prior runs, we should use Result<T> for error handling in domain layer"
- References to successful module structures and Hilt binding plans

### Domain Lead Uses Memory
- "Following prior pattern: DTOs with @Serializable + @SerialName"
- "Every ViewModel follows: @HiltViewModel + MutableStateFlow + UiState sealed class"

### Data Lead Uses Memory
- "@Serializable annotation applied to LoginDto, UserDto (matching 4 prior usages)"
- "Hilt @Binds binding for AuthRepository (consistent with prior runs)"

### Pres Lead Uses Memory
- "LoginViewModel structure: @HiltViewModel, MutableStateFlow + viewModelScope.launch (following pattern)"
- "UiState sealed class: Loading, Success, Error (standard pattern from prior runs)"

### Integration Uses Memory
- "Validating layer boundaries: Domain zero Android imports (per prior runs)"
- "Checking Hilt bindings: all @Binds modules in place (pattern consistency)"

### Test Agent Uses Memory
- "Domain test coverage target: 80%+ (per pattern)"
- "ViewModel tests: verify UiState emissions (per pattern)"

### Lint Agent Uses Memory
- "Detekt rule: no Android imports in domain/ (per pattern)"
- "Function naming: camelCase (per prior violations + fixes)"

## Memory Updates

**When:** After each successful pipeline run.

**What Gets Captured:**

1. **New Patterns:** Extract 3–5 key patterns from the successful run
   - How DTOs were structured
   - How ViewModels managed state
   - How repositories were bound
   - How layers were separated

2. **Error Resolutions:** If errors occurred, capture:
   - Error description
   - Resolution steps
   - Time to fix
   - Severity (high/medium/low)

3. **Performance Observations:** If notable performance achieved:
   - Offline-first caching implementation
   - Pagination pattern used
   - Flow usage for reactive updates

4. **Architecture Decisions:** Document any decisions made:
   - Why Clean Architecture (rationale)
   - Why Hilt over manual DI
   - Why Jetpack Compose over XML

**Memory Update Example:**

After successful "User Authentication" feature:

```json
{
  "pattern_id": "auth_offline_caching",
  "category": "data_layer",
  "description": "JWT token stored in SharedPreferences (encrypted), user profile cached in Room",
  "context": "auth feature, AuthRepositoryImpl, TokenCache wrapper",
  "example": "@Singleton\nclass TokenCache(context: Context) {\n  fun save(token: AuthToken) { ... }\n  fun get(): AuthToken? { ... }\n}",
  "frequency": 1,
  "date_first_observed": "2025-02-11",
  "date_last_used": "2025-02-11"
}
```

## Memory Lifespan

- **First use:** 1 run (frequency = 1)
- **Established pattern:** 3+ uses (frequency ≥ 3)
- **Deprecated pattern:** Not used for 10+ runs → mark as `deprecated` (but keep in history)

Example:
```json
{
  "pattern_id": "old_mvvm_pattern",
  "status": "deprecated",
  "last_used": "2024-12-01",
  "deprecation_reason": "Replaced by Clean Architecture"
}
```

## Memory Privacy & Scope

- **Scope:** Per project (one memory file per pipeline instance)
- **Sharing:** Memory is NOT shared across projects (each has independent memory)
- **Persistence:** Stored locally in `pipeline_memory.json` in project root

## Example: Memory-Guided Feature Implementation

**Scenario:** User requests "Add payment feature" for second time.

**Bootstrap:**
```
✓ Pipeline Memory loaded:
  - 6 patterns (including DTO, ViewModel, error handling, navigation, Hilt, domain purity)
  - 2 error patterns (Hilt binding, Android imports in domain)
  - Naming conventions established

Prompt: "Based on 3 prior successful runs, I'll apply these patterns:
  1. DTOs: @Serializable + @SerialName (field mapping)
  2. ViewModels: @HiltViewModel + MutableStateFlow + UiState sealed class
  3. Error handling: Result<T> in domain, try-catch in ViewModel
  4. Navigation: type-safe sealed class routes
  5. Hilt: @Module + @InstallIn, @Binds for repos
  6. Domain: zero Android imports

Ready to proceed?"
```

**Code Gen:**
```
Architect: "Following prior module structure for payments (domain/data/presentation)"
Domain Lead: "Entities + Use Cases following Kotlin-only pattern (4 prior uses)"
Data Lead: "@Serializable DTOs with @SerialName (consistent with prior pattern)"
Pres Lead: "@HiltViewModel + MutableStateFlow + UiState (standard pattern)"
Integration: "Hilt @Binds modules assembled per prior architecture"
```

**Test Agent:**
```
"Domain tests: 80%+ coverage (pattern from prior runs)
Data tests: 70%+ coverage (mappers, Retrofit mocks)
Presentation tests: 60%+ coverage (ViewModel state, Compose screens)"
```

**Result:** Faster implementation, fewer errors, consistent codebase.

---

## Memory Troubleshooting

### Problem: Memory Suggests Pattern That Fails

**Example:** "Use Result<T> for auth errors" but Result<T> doesn't compile in this context.

**Solution:**
1. Architect overrides memory: "Using Either<L, R> instead of Result<T> for this feature"
2. Pipeline Memory is updated ONLY if run succeeds
3. If override succeeds: new pattern "Either<L, R> for auth errors" is added to memory
4. If override fails: revert to Result<T>, no memory update

### Problem: Memory Conflict with New Architecture

**Example:** Memory suggests Clean Architecture, but new project uses MVVM.

**Solution:**
- Bootstrap detects new architecture during detection phase
- Memory patterns are contextualized: "Clean Architecture patterns: Domain, Data, Presentation"
- For MVVM: "MVVM patterns: ViewModel, UI, Repository, Model"
- Agents reference only relevant patterns based on detected architecture

### Problem: Memory Too Large (100+ patterns)

**Solution:**
- Consolidate patterns: merge similar patterns into one
- Deprecate low-frequency patterns (frequency = 1 for 10+ runs)
- Archive to `pipeline_memory_archive.json` for historical reference

---

## Project Context (Project Brain)

Beyond execution memory (patterns, error resolutions, naming conventions), Pipeline Memory also maintains a **Project Context** — a living snapshot of the project itself. This context grows automatically as the pipeline works on the project.

### Storage

Project context is stored alongside `pipeline_memory.json`:

```
project_context/
├── api-registry.json       # Known endpoints, auth patterns, response shapes
├── design-system.json      # Compose components, Material 3 tokens, theme
├── dependency-registry.json # Libraries, versions, rationale
├── module-map.json          # App modules, ownership, relationships
├── conventions.json         # Team naming, code style, PR process
└── domain-model.json        # Entities, relationships, business glossary
```

### API Registry

Tracks every API endpoint the pipeline discovers or creates.

```json
{
  "base_url": "https://api.myapp.com/v2",
  "auth": {
    "type": "bearer_jwt",
    "token_endpoint": "/auth/token",
    "refresh_endpoint": "/auth/refresh",
    "header": "Authorization: Bearer {token}"
  },
  "endpoints": [
    {
      "path": "/users/{id}",
      "method": "GET",
      "request": null,
      "response": "UserDto",
      "auth_required": true,
      "retrofit_service": "UserService",
      "added_by_feature": "user-profile",
      "date_added": "2025-02-10"
    },
    {
      "path": "/posts",
      "method": "POST",
      "request": "CreatePostRequest",
      "response": "PostDto",
      "auth_required": true,
      "retrofit_service": "PostService",
      "added_by_feature": "create-post",
      "date_added": "2025-02-12"
    }
  ]
}
```

**How agents use it:**
- **Product Agent**: Checks existing endpoints before proposing new ones
- **Data Lead**: Knows the auth pattern, base URL, existing Retrofit services and DTOs
- **Architect**: Sees the full API surface when planning new features

### Design System Registry

Captures the project's Material Design 3 language as it evolves.

```json
{
  "theme": {
    "color_scheme": "dynamic_color_with_fallback",
    "primary": "#6750A4",
    "secondary": "#625B71",
    "surface": "#FFFBFE",
    "error": "#B3261E",
    "typography": {
      "displayLarge": "Roboto, 57sp",
      "headlineMedium": "Roboto, 28sp",
      "bodyLarge": "Roboto, 16sp",
      "labelSmall": "Roboto, 11sp"
    },
    "spacing": { "xs": "4.dp", "sm": "8.dp", "md": "16.dp", "lg": "24.dp", "xl": "32.dp" }
  },
  "components": [
    {
      "name": "PrimaryButton",
      "file": "presentation/components/PrimaryButton.kt",
      "composable": true,
      "params": ["text: String", "onClick: () -> Unit", "isLoading: Boolean = false"],
      "usage_count": 8
    },
    {
      "name": "ErrorBanner",
      "file": "presentation/components/ErrorBanner.kt",
      "composable": true,
      "params": ["message: String", "onRetry: (() -> Unit)? = null"],
      "usage_count": 5
    }
  ],
  "conventions": {
    "loading_state": "CircularProgressIndicator() centered in Box(fillMaxSize)",
    "empty_state": "Column with illustration + message + Button",
    "error_state": "ErrorBanner at top + retry button"
  }
}
```

**How agents use it:**
- **Presentation Lead**: Reuses existing Composables instead of creating duplicates
- **Product Agent**: References established UI patterns when writing specs
- **Architect**: Knows the component library size when estimating effort

### Dependency Registry

Tracks all dependencies in `build.gradle.kts` and why they were chosen.

```json
{
  "build_system": "gradle_kts",
  "dependencies": [
    {
      "group": "com.squareup.retrofit2",
      "artifact": "retrofit",
      "version": "2.10.0",
      "purpose": "HTTP networking",
      "rationale": "Industry standard, coroutine-friendly, interceptor support",
      "layer": "data",
      "added_by_feature": "initial-setup"
    },
    {
      "group": "com.google.dagger",
      "artifact": "hilt-android",
      "version": "2.51",
      "purpose": "Dependency injection",
      "rationale": "Android-standard DI, compile-time safety, ViewModel integration",
      "layer": "di",
      "added_by_feature": "initial-setup"
    },
    {
      "group": "androidx.room",
      "artifact": "room-runtime",
      "version": "2.6.1",
      "purpose": "Local database",
      "rationale": "Type-safe SQL, Flow support, migration support",
      "layer": "data",
      "added_by_feature": "offline-cache"
    }
  ]
}
```

**How agents use it:**
- **Architect**: Knows what's available before proposing new libraries
- **Data Lead**: Uses existing networking/storage deps instead of adding alternatives
- **Integration**: Verifies no duplicate or conflicting dependencies in build.gradle.kts

### Module Map

Captures how the app is organized — features, infrastructure, shared modules.

```json
{
  "modules": [
    {
      "name": "Auth",
      "type": "feature",
      "layers": ["domain", "data", "presentation"],
      "entities": ["User", "AuthToken", "Session"],
      "screens": ["LoginScreen", "RegisterScreen", "ForgotPasswordScreen"],
      "hilt_modules": ["AuthModule", "AuthProvidersModule"],
      "dependencies": ["Networking", "Storage"]
    },
    {
      "name": "Networking",
      "type": "infrastructure",
      "layers": ["data"],
      "provides": ["ApiClient", "TokenInterceptor", "NetworkMonitor"],
      "hilt_modules": ["NetworkModule"],
      "used_by": ["Auth", "Posts", "Profile"]
    }
  ],
  "shared_modules": [
    {
      "name": "DesignSystem",
      "type": "ui-kit",
      "composables": 12,
      "used_by": ["Auth", "Posts", "Profile", "Settings"]
    }
  ]
}
```

**How agents use it:**
- **Coordinator**: Knows which module a task belongs to
- **Architect**: Sees module dependencies and Hilt module boundaries
- **Product Agent**: Understands existing feature boundaries

### Team Conventions

Captures naming, formatting, and process conventions.

```json
{
  "naming": {
    "branches": "feature/{ticket-id}-{short-description}",
    "commits": "conventional: feat:, fix:, refactor:, test:, docs:",
    "files": {
      "screens": "{Feature}Screen.kt",
      "viewmodels": "{Feature}ViewModel.kt",
      "use_cases": "{Action}{Entity}UseCase.kt",
      "repositories": "{Entity}Repository.kt",
      "repository_impls": "{Entity}RepositoryImpl.kt",
      "dtos": "{Entity}Dto.kt",
      "room_entities": "{Entity}Entity.kt",
      "hilt_modules": "{Feature}Module.kt"
    }
  },
  "code_style": {
    "coroutines": "suspend functions preferred over callbacks",
    "state": "StateFlow + UiState sealed class in every ViewModel",
    "nullability": "avoid !! operator, use safe calls and elvis",
    "compose": "stateless Composables, state hoisting to ViewModel"
  },
  "process": {
    "pr_size": "max 400 lines changed",
    "review_required": true,
    "ci_must_pass": true,
    "squash_on_merge": true
  }
}
```

### Domain Model

Captures entities, relationships, and business terminology.

```json
{
  "entities": [
    {
      "name": "User",
      "fields": ["id: String", "name: String", "email: String", "avatarUrl: String?"],
      "relationships": ["has many Posts", "has one Session"],
      "business_rules": ["Email must be unique", "Name 2-50 chars"]
    },
    {
      "name": "Post",
      "fields": ["id: String", "title: String", "body: String", "authorId: String", "createdAt: Instant"],
      "relationships": ["belongs to User", "has many Comments"],
      "business_rules": ["Title required, max 200 chars", "Body required, max 5000 chars"]
    }
  ],
  "glossary": {
    "Session": "Active authentication state with JWT and refresh token",
    "Draft": "Unpublished post stored in Room until user submits"
  }
}
```

### Context Auto-Population

Project Context populates **automatically** — no manual configuration needed.

**On bootstrap (first run):**
- Scans `build.gradle.kts` → populates dependency registry
- Scans project folders → populates module map
- Scans Kotlin data classes → populates domain model
- Detects Android Studio settings → populates conventions

**During each run:**
- Product Agent adds new entities to domain model
- Data Lead adds new endpoints to API registry, new Room entities
- Presentation Lead registers new Composables in design system
- Integration updates module dependencies and Hilt module map

**Progressive enrichment:** Context starts sparse and gets richer with every pipeline run. By run 5, the pipeline knows your project deeply.

### Context Queries

Agents query project context alongside execution memory:

```
context.api("user endpoints")         → all /users/* endpoints
context.components("button")          → PrimaryButton, SecondaryButton
context.module("Auth")                → full Auth module info
context.entity("User")               → User fields, relationships, rules
context.dependency("networking")      → Retrofit details
context.convention("branch naming")   → feature/{ticket-id}-{desc}
```
