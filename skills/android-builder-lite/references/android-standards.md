# Android Standards for Lite Pipeline

## Language & Coroutines

### Kotlin
- Latest stable version (1.9+)
- Use Kotlin idioms: data classes, sealed classes, extension functions, scope functions (let, apply, run, with, also)
- Immutability: `val` preferred over `var`
- Null safety: use non-null types and `?.` safe calls

### Coroutines
- `suspend` functions for async operations
- `Flow<T>` for reactive streams
- `viewModelScope.launch` for UI updates
- `CoroutineScope` for scoped operations
- `Dispatchers.IO` for network/database, `Dispatchers.Main` for UI

```kotlin
// Good: suspend function
suspend fun getRecipes(): List<Recipe>

// Good: Flow for streams
fun observeRecipes(): Flow<List<Recipe>>

// Good: viewModelScope for lifecycle-aware cancellation
viewModelScope.launch {
    val recipes = getRecipesUseCase()
}

// Avoid: callback-based
fun getRecipes(callback: (List<Recipe>) -> Unit)
```

---

## UI Framework

### Jetpack Compose Only
- No XML layouts
- All UI defined in @Composable functions
- Compose is the standard for Android UI

```kotlin
@Composable
fun RecipeListScreen() {
    Column {
        RecipeSearchBar()
        RecipeList()
    }
}
```

### Material Design 3
- Use Material3 components (Button, TextField, Card, etc.)
- Material3 color scheme: primary, secondary, tertiary, error, surface, outline
- Typography: displayLarge, displayMedium, titleLarge, bodyMedium, bodySmall, labelSmall
- Spacing: 4dp, 8dp, 16dp, 24dp, 32dp (design tokens)

```kotlin
@Composable
fun RecipeCard(recipe: Recipe) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(8.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp),
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surfaceVariant
        )
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text = recipe.name,
                style = MaterialTheme.typography.titleLarge,
                color = MaterialTheme.colorScheme.onSurface
            )
        }
    }
}
```

### Compose Navigation
- `NavHost` with `composable` routes
- Type-safe routes (if using Kotlin serialization + custom setup) or string-based routes
- NavHostController for programmatic navigation
- Arguments via route parameters or SavedStateHandle

```kotlin
@Composable
fun AppNavHost(
    navController: NavHostController = rememberNavController()
) {
    NavHost(navController = navController, startDestination = "recipe-list") {
        composable("recipe-list") {
            RecipeListScreen(
                onNavigateToDetail = { id -> navController.navigate("recipe-detail/$id") }
            )
        }
        composable("recipe-detail/{id}") { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id") ?: return@composable
            RecipeDetailScreen(id, onBack = { navController.popBackStack() })
        }
    }
}
```

---

## Architecture: MVVM + Clean Architecture

### Three-Layer Structure

#### Domain Layer (Pure Kotlin)
- **No Android imports**: No Context, Activity, Fragment, View, etc.
- **Data Classes** (Entities): Immutable, business logic, validation
- **Use Cases**: One responsibility per class, `suspend operator fun invoke()`
- **Repository Interfaces**: Abstract contracts, no implementation details
- **Value Objects**: Strongly-typed primitives (UserId vs String)
- **Exceptions**: DomainException sealed classes

```kotlin
// ✓ Good: Pure Kotlin
data class Recipe(val id: String, val name: String)

interface RecipeRepository {
    suspend fun getRecipes(): Result<List<Recipe>>
}

class GetRecipesUseCase(val repo: RecipeRepository) {
    suspend operator fun invoke() = repo.getRecipes()
}

sealed class DomainException : Exception()
data class RecipeNotFound(val id: String) : DomainException()

// ✗ Bad: Android imports
import android.content.Context  // ← No!
import androidx.room.Entity      // ← No!
```

#### Data Layer (Android + Domain)
- **DTOs** (@Serializable): Mirror API shapes, serialization metadata
- **Mappers**: DTO ↔ Domain, DTO ↔ Entity
- **Retrofit Service**: Remote API calls (@GET, @POST, suspend)
- **Room**: Local persistence (entities, DAOs, Database)
- **Repository Implementations** (@Binds): Coordinate remote + local

```kotlin
// DTOs with serialization
@Serializable
data class RecipeDto(val id: String, val name: String)

// Mapper: DTO → Domain
fun RecipeDto.toDomain() = Recipe(id, name)

// Retrofit service
interface RecipeService {
    @GET("/recipes")
    suspend fun getRecipes(): List<RecipeDto>
}

// Room
@Entity(tableName = "recipes")
data class RecipeEntity(val id: String, val name: String)

@Dao
interface RecipeDao {
    @Query("SELECT * FROM recipes")
    fun getAllRecipes(): Flow<List<RecipeEntity>>
}

// Repository impl
class RecipeRepositoryImpl(val service: RecipeService, val dao: RecipeDao) : RecipeRepository {
    override suspend fun getRecipes(): Result<List<Recipe>> = try {
        val dtos = service.getRecipes()
        Result.Success(dtos.map { it.toDomain() })
    } catch (e: Exception) {
        Result.Error(DomainException.NetworkError(e))
    }
}
```

#### Presentation Layer (Compose + Domain)
- **ViewModels** (@HiltViewModel): Inject use cases, expose StateFlow<UiState>
- **UiState** (sealed classes): Loading, Success(data), Error(message)
- **Screens** (@Composable): Render based on UiState, collect with collectAsState()
- **Navigation**: NavHost, routes, arguments
- **NO direct Data references**: Always via Domain interfaces

```kotlin
// UiState
sealed class RecipeListUiState {
    object Loading : RecipeListUiState()
    data class Success(val recipes: List<Recipe>) : RecipeListUiState()
    data class Error(val message: String) : RecipeListUiState()
}

// ViewModel with Hilt
@HiltViewModel
class RecipeListViewModel @Inject constructor(
    val getRecipesUseCase: GetRecipesUseCase
) : ViewModel() {
    private val _uiState = MutableStateFlow<RecipeListUiState>(RecipeListUiState.Loading)
    val uiState = _uiState.asStateFlow()

    init {
        viewModelScope.launch {
            _uiState.value = when (val result = getRecipesUseCase()) {
                is Result.Success -> RecipeListUiState.Success(result.data)
                is Result.Error -> RecipeListUiState.Error(result.exception.message ?: "Unknown")
            }
        }
    }
}

// Screen: Compose only
@Composable
fun RecipeListScreen(
    onNavigateToDetail: (String) -> Unit,
    viewModel: RecipeListViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()

    when (uiState) {
        is RecipeListUiState.Loading -> LoadingIndicator()
        is RecipeListUiState.Success -> RecipeListContent(
            recipes = (uiState as RecipeListUiState.Success).recipes,
            onRecipeClick = onNavigateToDetail
        )
        is RecipeListUiState.Error -> ErrorMessage((uiState as RecipeListUiState.Error).message)
    }
}

// ✗ Bad: Direct reference to DTO (data layer)
data class RecipeDto(...)  // Never use in Presentation!
```

---

## Dependency Injection: Hilt

### @HiltAndroidApp
Mark the Application class:

```kotlin
@HiltAndroidApp
class App : Application()
```

### @AndroidEntryPoint
Mark activities and fragments that use injection:

```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity()

@AndroidEntryPoint
class MyFragment : Fragment()
```

### @HiltViewModel
Mark ViewModels for injection:

```kotlin
@HiltViewModel
class RecipeListViewModel @Inject constructor(
    val getRecipesUseCase: GetRecipesUseCase
) : ViewModel()
```

### @Module + @InstallIn
Define dependency providers:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
interface RepositoryModule {
    @Binds
    @Singleton
    fun bindRecipeRepository(impl: RecipeRepositoryImpl): RecipeRepository
}

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideRecipeService(): RecipeService =
        Retrofit.Builder().baseUrl("https://api.example.com/").build()
            .create(RecipeService::class.java)
}
```

### Scopes
- `@Singleton`: Application-wide (1 instance)
- `@ViewModelScoped`: Per ViewModel (same ViewModel = same instance)
- `@ActivityScoped`: Per Activity (same Activity = same instance)

---

## State Management: StateFlow + UiState

### StateFlow + UiState Pattern
Define state as a sealed class, expose via StateFlow:

```kotlin
sealed class MyUiState {
    object Loading : MyUiState()
    data class Success(val data: String) : MyUiState()
    data class Error(val message: String) : MyUiState()
}

class MyViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<MyUiState>(MyUiState.Loading)
    val uiState: StateFlow<MyUiState> = _uiState.asStateFlow()

    fun loadData() {
        viewModelScope.launch {
            try {
                val data = fetchData()
                _uiState.value = MyUiState.Success(data)
            } catch (e: Exception) {
                _uiState.value = MyUiState.Error(e.message ?: "Unknown")
            }
        }
    }
}
```

### Unidirectional Data Flow (UDF)
```
User Action → ViewModel Event → Update State → Recompose Screen
```

```kotlin
@Composable
fun MyScreen(viewModel: MyViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()

    when (uiState) {
        is MyUiState.Loading -> LoadingIndicator()
        is MyUiState.Success -> SuccessContent((uiState as MyUiState.Success).data)
        is MyUiState.Error -> ErrorContent((uiState as MyUiState.Error).message)
    }
}
```

---

## Data Persistence

### Room Database
- Entities (@Entity), DAOs (@Dao), Database (@Database)
- Type converters for complex types
- Migrations for schema changes

```kotlin
@Entity(tableName = "recipes")
data class RecipeEntity(
    @PrimaryKey val id: String,
    val name: String,
    val cuisine: String
)

@Dao
interface RecipeDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertRecipes(recipes: List<RecipeEntity>)

    @Query("SELECT * FROM recipes")
    fun getAllRecipes(): Flow<List<RecipeEntity>>

    @Query("DELETE FROM recipes WHERE id = :id")
    suspend fun deleteRecipe(id: String)
}

@Database(entities = [RecipeEntity::class], version = 1, exportSchema = true)
abstract class AppDatabase : RoomDatabase() {
    abstract fun recipeDao(): RecipeDao
}
```

### SharedPreferences / DataStore
For simple key-value storage:

```kotlin
// Using DataStore (preferred)
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

val userNameKey = stringPreferencesKey("user_name")

// Write
dataStore.edit { preferences ->
    preferences[userNameKey] = "John"
}

// Read
val userName = dataStore.data.map { preferences ->
    preferences[userNameKey] ?: ""
}
```

---

## Networking: Retrofit + Kotlinx.Serialization

### Retrofit Service
```kotlin
interface RecipeService {
    @GET("/recipes")
    suspend fun getRecipes(): List<RecipeDto>

    @GET("/recipes/{id}")
    suspend fun getRecipe(@Path("id") id: String): RecipeDto

    @POST("/recipes")
    suspend fun createRecipe(@Body recipe: RecipeDto): RecipeDto

    @PUT("/recipes/{id}")
    suspend fun updateRecipe(
        @Path("id") id: String,
        @Body recipe: RecipeDto
    ): RecipeDto

    @DELETE("/recipes/{id}")
    suspend fun deleteRecipe(@Path("id") id: String)
}
```

### DTOs with @Serializable
```kotlin
@Serializable
data class RecipeDto(
    val id: String,
    val name: String,
    @SerialName("prep_time_minutes")
    val prepTime: Int,
    val ingredients: List<IngredientDto>
)

@Serializable
data class IngredientDto(
    val name: String,
    val amount: Float,
    val unit: String
)
```

### Setup Retrofit with Hilt
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .addConverterFactory(
                Json.asConverterFactory("application/json".toMediaType())
            )
            .build()

    @Provides
    @Singleton
    fun provideRecipeService(retrofit: Retrofit): RecipeService =
        retrofit.create(RecipeService::class.java)
}
```

---

## Naming Conventions

### Packages
- `com.example.app` — root
- `com.example.app.feature.recipe` — feature modules
- `com.example.app.feature.recipe.data` — data layer
- `com.example.app.feature.recipe.domain` — domain layer
- `com.example.app.feature.recipe.presentation` — UI layer
- `com.example.app.di` — Hilt modules
- `com.example.app.core.ui` — shared Compose components
- `com.example.app.navigation` — navigation setup

### Classes & Types
- **PascalCase** for classes: `RecipeListViewModel`, `RecipeRepository`, `GetRecipesUseCase`
- **PascalCase** for sealed classes: `RecipeListUiState`, `Result<T>`
- **PascalCase** for data classes: `Recipe`, `Ingredient`
- **camelCase** for functions: `getRecipes()`, `validateInput()`
- **camelCase** for variables: `val recipe: Recipe`, `var count: Int`
- **UPPER_SNAKE_CASE** for constants: `API_TIMEOUT = 30000`, `DEFAULT_PAGE_SIZE = 10`

### ViewModels
- Suffix: `ViewModel`
- Private state: `_uiState`, `_events`
- Public state: `uiState`, `events`

```kotlin
@HiltViewModel
class RecipeListViewModel @Inject constructor(...) : ViewModel() {
    private val _uiState = MutableStateFlow<RecipeListUiState>(RecipeListUiState.Loading)
    val uiState: StateFlow<RecipeListUiState> = _uiState.asStateFlow()
}
```

### Compose Functions
- **PascalCase**: `RecipeListScreen()`, `RecipeCard()`, `LoadingIndicator()`
- Parameter naming: descriptive, often callbacks: `onRecipeClick`, `onNavigateToDetail`

```kotlin
@Composable
fun RecipeListScreen(
    onNavigateToDetail: (recipeId: String) -> Unit,
    viewModel: RecipeListViewModel = hiltViewModel()
)

@Composable
fun RecipeCard(
    recipe: Recipe,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
)
```

### Lifecycle Functions
- `onCreate`, `onStart`, `onResume`, `onPause`, `onStop`, `onDestroy` (Activity)
- `init`, `onCleared` (ViewModel)
- `@Composable` functions (no lifecycle, composition-based)

---

## File Structure

```
app/src/main/kotlin/com/example/app/
├── App.kt                             # @HiltAndroidApp
├── MainActivity.kt                    # @AndroidEntryPoint, setContent(AppNavHost)
│
├── di/                                # Hilt modules
│   ├── RepositoryModule.kt            # @Binds for repositories
│   ├── NetworkModule.kt               # @Provides for Retrofit, OkHttp
│   └── DatabaseModule.kt              # @Provides for Room
│
├── feature/
│   └── recipe/                        # Feature module
│       ├── data/
│       │   ├── local/
│       │   │   ├── RecipeEntity.kt
│       │   │   ├── RecipeDao.kt
│       │   │   └── AppDatabase.kt
│       │   ├── remote/
│       │   │   ├── RecipeService.kt
│       │   │   └── dto/
│       │   │       ├── RecipeDto.kt
│       │   │       └── IngredientDto.kt
│       │   ├── mapper/
│       │   │   └── RecipeMapper.kt
│       │   └── repository/
│       │       └── RecipeRepositoryImpl.kt
│       │
│       ├── domain/
│       │   ├── entity/
│       │   │   ├── Recipe.kt
│       │   │   └── Ingredient.kt
│       │   ├── repository/
│       │   │   └── RecipeRepository.kt
│       │   ├── usecase/
│       │   │   ├── GetRecipesUseCase.kt
│       │   │   ├── GetRecipeUseCase.kt
│       │   │   └── SearchRecipesUseCase.kt
│       │   └── exception/
│       │       └── DomainException.kt
│       │
│       └── presentation/
│           ├── uistate/
│           │   └── RecipeListUiState.kt
│           ├── viewmodel/
│           │   ├── RecipeListViewModel.kt
│           │   └── RecipeDetailViewModel.kt
│           ├── screen/
│           │   ├── RecipeListScreen.kt
│           │   └── RecipeDetailScreen.kt
│           └── component/
│               ├── RecipeCard.kt
│               ├── LoadingIndicator.kt
│               └── ErrorMessage.kt
│
├── core/
│   ├── ui/
│   │   ├── component/
│   │   │   └── CommonComponents.kt
│   │   ├── theme/
│   │   │   └── Theme.kt
│   │   └── theme/
│   │       ├── Color.kt
│   │       └── Type.kt
│   └── domain/
│       ├── result/
│       │   └── Result.kt
│       └── exception/
│           └── DomainException.kt
│
└── navigation/
    └── AppNavHost.kt
```

---

## Layer Import Rules

**Domain**: Imports from domain only (no dependencies on other layers)
```kotlin
// ✓ Good
import com.example.app.feature.recipe.domain.entity.Recipe
import com.example.app.feature.recipe.domain.exception.DomainException

// ✗ Bad
import com.example.app.feature.recipe.data.remote.RecipeService
import android.content.Context
```

**Data**: Imports from domain + Android (Room, Retrofit)
```kotlin
// ✓ Good
import com.example.app.feature.recipe.domain.entity.Recipe
import com.example.app.feature.recipe.domain.repository.RecipeRepository
import androidx.room.*
import retrofit2.*

// ✗ Bad
import com.example.app.feature.recipe.presentation.viewmodel.RecipeListViewModel
import androidx.compose.runtime.Composable
```

**Presentation**: Imports from domain (via repository interfaces)
```kotlin
// ✓ Good
import com.example.app.feature.recipe.domain.entity.Recipe
import com.example.app.feature.recipe.domain.usecase.GetRecipesUseCase
import androidx.compose.runtime.*
import androidx.hilt.navigation.compose.hiltViewModel

// ✗ Bad
import com.example.app.feature.recipe.data.remote.RecipeService
import com.example.app.feature.recipe.data.local.RecipeEntity
```

---

## Code Quality: Linting & Analysis

### ktlint
Kotlin style linter, enforces Kotlin conventions.

```bash
# Check style
./gradlew ktlint

# Auto-fix (if available)
./gradlew ktlintFormat
```

**Common rules**:
- 4-space indentation
- Max line length 120
- No trailing whitespace
- No wildcard imports
- No unused imports

### detekt
Static analysis tool for code smell and design issues.

```bash
# Run analysis
./gradlew detekt

# Config file: detekt.yml
```

**Common rules**:
- Naming conventions
- Complexity (cognitive complexity, cyclomatic complexity)
- Code duplication
- Unused code
- Magic numbers

### build.gradle.kts
Include lint tasks:

```kotlin
plugins {
    id("org.jlleitschuh.gradle.ktlint") version "11.5.1"
    id("io.gitlab.arturbosch.detekt") version "1.23.1"
}

ktlint {
    version.set("0.50.0")
}

detekt {
    config.setFrom("detekt.yml")
}
```

---

## Testing: JUnit5 + MockK

### Domain Layer Tests
- Test entities, use cases, exceptions
- Mock repositories

```kotlin
class GetRecipesUseCaseTest {
    private val repository: RecipeRepository = mockk()
    private val useCase = GetRecipesUseCase(repository)

    @Test
    fun testGetRecipesReturnsSuccessResult() = runTest {
        val recipes = listOf(Recipe(...))
        coEvery { repository.getRecipes() } returns Result.Success(recipes)

        val result = useCase()

        assertEquals(recipes, (result as Result.Success).data)
    }
}
```

### Data Layer Tests
- Test mappers, repository implementations, DAOs
- Mock Retrofit service, Room database

```kotlin
class RecipeRepositoryImplTest {
    private val service: RecipeService = mockk()
    private val dao: RecipeDao = mockk()
    private val repository = RecipeRepositoryImpl(service, dao)

    @Test
    fun testGetRecipesMapsDtoToDomain() = runTest {
        val dto = RecipeDto(id = "1", name = "Pasta")
        coEvery { service.getRecipes() } returns listOf(dto)

        val result = repository.getRecipes()

        val success = (result as Result.Success).data
        assertEquals("Pasta", success[0].name)
    }
}
```

### Presentation Layer Tests
- Test ViewModels, UiState transitions
- Mock use cases

```kotlin
class RecipeListViewModelTest {
    private val getRecipesUseCase: GetRecipesUseCase = mockk()

    @Test
    fun testInitialStateIsLoading() {
        val viewModel = RecipeListViewModel(getRecipesUseCase)
        assertEquals(RecipeListUiState.Loading, viewModel.uiState.value)
    }

    @Test
    fun testSuccessStateWhenRecipesFetched() = runTest {
        val recipes = listOf(Recipe(...))
        coEvery { getRecipesUseCase() } returns Result.Success(recipes)

        val viewModel = RecipeListViewModel(getRecipesUseCase)
        advanceUntilIdle()

        val state = viewModel.uiState.value
        assertTrue(state is RecipeListUiState.Success)
    }
}
```

---

## Compose Preview

Use @Preview for UI testing:

```kotlin
@Preview(showBackground = true)
@Composable
fun RecipeCardPreview() {
    RecipeCard(
        recipe = Recipe(
            id = "1",
            name = "Pasta",
            cuisine = "Italian",
            servings = 4
        ),
        onClick = { }
    )
}
```

---

## Summary

- **Language**: Kotlin (latest) + Coroutines (suspend, Flow)
- **UI**: Jetpack Compose + Material Design 3
- **Architecture**: MVVM + Clean Architecture (Domain → Data → Presentation)
- **DI**: Hilt (@HiltViewModel, @Module, @Binds, @Provides)
- **State**: StateFlow + UiState sealed classes + UDF
- **Persistence**: Room + DataStore
- **Networking**: Retrofit + Kotlinx.serialization
- **Testing**: JUnit5 + MockK
- **Lint**: ktlint + detekt
- **Navigation**: Compose Navigation with type-safe or string-based routes

