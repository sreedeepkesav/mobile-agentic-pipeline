# 4-Phase Coder for Android Lite

## Overview

After design approval, the Coder executes in 4 sequential phases:
1. **Architect** — Plan (ADR, no code)
2. **Domain Lead** — Pure Kotlin domain layer
3. **Data ‖ Presentation** — Parallel implementation
4. **Integration** — Hilt wiring, lint, layer audits

Total time: ~10-15 hours for a medium feature.

---

## Phase 1: Architect (2-4 hours)

**Goal**: Plan the implementation without writing code.

### Inputs
- spec.md, screens.md, api-contracts.md
- data-models.md, design-tokens.json, edge-cases.md

### Research
- Kotlin coroutines (suspend, Flow, Channel)
- Jetpack Compose (state management, recomposition)
- Hilt scopes (Singleton, ViewModelScoped, ActivityScoped)
- Navigation Compose (NavHost, composable, type-safe routes)
- Room database (entities, DAOs, migrations)
- Retrofit (interceptors, error handling, serialization)
- MVVM and UiState patterns
- Clean Architecture layer boundaries

### Decisions to Document

#### 1. Module Structure
Decide how to organize the codebase:

```
Example: Recipe App
├── :app                           # Single Activity, MainActivity, AppNavHost
├── :feature:recipe                # Recipe feature
│   ├── data/
│   ├── domain/
│   └── presentation/
├── :feature:search                # Search feature (optional)
├── :core:ui                       # Shared Compose components, Material theme
├── :core:domain                   # Shared domain types (Result<T>, errors)
└── :core:data                     # Shared data types (pagination, cache)
```

Decision: Which features get separate modules? (Usually, yes, for modularity.)

#### 2. Hilt Setup

```
Scopes:
- @Singleton: RecipeService, RecipeDao, Repository
- @ViewModelScoped: ViewModels
- @ActivityScoped: Single-activity lifetime

Modules:
- NetworkModule: Provides Retrofit, OkHttpClient, interceptors
- DatabaseModule: Provides Room Database, DAOs
- RepositoryModule: Binds Repository implementations
- FeatureModule: Provides feature-specific dependencies
```

Decision: Which classes are Singleton? Which are ViewModelScoped?

#### 3. Navigation Graph

Type-safe routes using serializable objects:

```kotlin
// navigation/Routes.kt
object RecipeRoutes {
    const val RecipeListRoute = "recipe-list"
    const val RecipeDetailRoute = "recipe-detail"
}

@Serializable
data class RecipeDetailArgs(val recipeId: String)
```

Or simpler string-based:

```kotlin
NavHost(navController, startDestination = "recipe-list") {
    composable("recipe-list") { RecipeListScreen() }
    composable("recipe-detail/{id}") { backStackEntry ->
        RecipeDetailScreen(backStackEntry.arguments?.getString("id"))
    }
}
```

Decision: Type-safe routes (serializable) or string-based? Nested navigation?

#### 4. Error Handling

Define error types and Result wrapper:

```kotlin
// domain/error/DomainException.kt
sealed class DomainException(message: String) : Exception(message) {
    data class NotFound(val itemId: String) : DomainException("Not found: $itemId")
    data class ValidationError(val field: String) : DomainException("Invalid $field")
    data class NetworkError(val cause: Throwable) : DomainException("Network error")
}

// domain/result/Result.kt
sealed class Result<T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error<T>(val exception: DomainException) : Result<T>()
}
```

Decision: Throw exceptions or use Result<T>? Which error types?

#### 5. Testing Strategy

- **Domain**: 100% unit test coverage (pure functions, no dependencies)
- **Data**: Repository and DAO unit tests with mock network
- **Presentation**: ViewModel state tests with mock use cases
- **Integration**: Optional E2E tests for critical flows

Decision: Testing tools (JUnit5, MockK, Kotest)?

#### 6. Layer Audit Plan

Document the rules:
- Domain: Zero Android imports (no Context, View, etc.)
- Data: Can import Domain, Android (Room, Retrofit). Cannot import Presentation.
- Presentation: Can import Domain via interfaces. Cannot import Data DTOs directly.
- DI: Wires all layers together.

Decision: Enforcement method (linter rules, IDE inspection)?

### Output: Architecture Decision Record (ADR)

```markdown
# ADR: Recipe App Architecture

## Context
Building a recipe search and detail app with local caching and remote API.

## Decision
Modular MVVM + Clean Architecture with Hilt DI.

### Module Structure
- :app (single activity, navigation)
- :feature:recipe (recipe CRUD)
- :core:ui (Compose components, theme)
- :core:domain (shared exceptions, result types)
- :core:data (shared DTOs, pagination)

### Hilt Scopes
- @Singleton: RecipeService, RecipeDatabase, RecipeRepository
- @ViewModelScoped: ViewModels
- Per-feature @Module classes

### Navigation
String-based routes in single NavHost. Arguments passed via route parameters.

### Error Handling
Result<T> sealed class. Domain throws DomainException subclasses.

### Layer Boundaries
```
Domain (no Android)
  ↑
Data (Android: Room, Retrofit) → Domain
  ↑
Presentation (Compose) → Domain
  ↑
DI (Hilt) → all layers
```

### Testing
JUnit5 + MockK. Domain: 100%. Data: 80% (repos, DAOs). Presentation: 70% (VM state).

## Consequences
- Modularity aids testing and parallel development
- Hilt compilation adds slight build time overhead
- Clean boundaries ensure testability
- Single Activity pattern simplifies navigation

## Alternatives Considered
- Multi-activity architecture (rejected: harder navigation)
- MVI instead of MVVM (rejected: MVVM more standard in Jetpack)
- Manual DI (rejected: Hilt reduces boilerplate)
```

---

## Phase 2: Domain Lead (2-3 hours)

**Goal**: Implement pure Kotlin domain layer with zero Android imports.

### Responsibilities

1. **Data Classes (Entities)**
   - Immutable (data classes with val)
   - Business logic (calculated properties, validation methods)
   - No serialization annotations (@Serializable), no Room annotations

2. **Use Cases**
   - One responsibility per use case
   - @Inject constructor with repositories
   - Suspend operators for async
   - Handle Result<T> or throw DomainException

3. **Repository Interfaces**
   - Abstract contracts (no implementation details)
   - Suspend functions
   - Return Result<T> or domain entities

4. **Value Objects & Exceptions**
   - Strongly-typed primitives (UserId vs String)
   - Custom exception types

### Example: Recipe Domain Layer

```kotlin
// domain/entity/Recipe.kt
package com.example.recipe.domain.entity

data class Recipe(
    val id: String,
    val name: String,
    val ingredients: List<Ingredient>,
    val instructions: List<Instruction>,
    val cuisine: String,
    val servings: Int,
    val prepTime: Int,  // minutes
    val cookTime: Int,  // minutes
    val difficulty: Difficulty,
    val rating: Float = 0f
) {
    init {
        require(id.isNotBlank()) { "Recipe ID must not be blank" }
        require(name.isNotBlank()) { "Recipe name must not be blank" }
        require(servings > 0) { "Servings must be > 0" }
    }
}

enum class Difficulty {
    Easy, Medium, Hard
}

data class Ingredient(
    val name: String,
    val amount: Float,
    val unit: String
)

data class Instruction(
    val step: Int,
    val description: String
)

// domain/entity/SearchQuery.kt
data class SearchQuery(
    val id: String,
    val text: String,
    val timestamp: Long
)

// domain/error/DomainException.kt
sealed class DomainException(override val message: String) : Exception(message) {
    data class RecipeNotFound(val recipeId: String) :
        DomainException("Recipe not found: $recipeId")

    data class InvalidSearch(val reason: String) :
        DomainException("Invalid search: $reason")

    data class NetworkError(val cause: Throwable) :
        DomainException("Network error: ${cause.message}")
}

// domain/result/Result.kt
sealed class Result<T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error<T>(val exception: DomainException) : Result<T>()
}

// domain/repository/RecipeRepository.kt
interface RecipeRepository {
    suspend fun getRecipes(): Result<List<Recipe>>
    suspend fun getRecipe(id: String): Result<Recipe>
    suspend fun saveRecipe(recipe: Recipe): Result<Unit>
    suspend fun deleteRecipe(id: String): Result<Unit>
}

interface SearchQueryRepository {
    suspend fun getSearchHistory(): Result<List<SearchQuery>>
    suspend fun saveSearchQuery(query: SearchQuery): Result<Unit>
    suspend fun clearSearchHistory(): Result<Unit>
}

// domain/usecase/GetRecipesUseCase.kt
class GetRecipesUseCase(
    private val recipeRepository: RecipeRepository
) {
    suspend operator fun invoke(): Result<List<Recipe>> =
        recipeRepository.getRecipes()
}

// domain/usecase/GetRecipeUseCase.kt
class GetRecipeUseCase(
    private val recipeRepository: RecipeRepository
) {
    suspend operator fun invoke(id: String): Result<Recipe> {
        require(id.isNotBlank()) { "Recipe ID must not be blank" }
        return recipeRepository.getRecipe(id)
    }
}

// domain/usecase/SearchRecipesUseCase.kt
class SearchRecipesUseCase(
    private val recipeRepository: RecipeRepository,
    private val searchQueryRepository: SearchQueryRepository
) {
    suspend operator fun invoke(query: String): Result<List<Recipe>> {
        require(query.isNotBlank()) { "Search query must not be blank" }
        require(query.length >= 3) { "Search query must be at least 3 characters" }

        val searchQuery = SearchQuery(
            id = UUID.randomUUID().toString(),
            text = query,
            timestamp = System.currentTimeMillis()
        )

        // Save to history (fire and forget)
        searchQueryRepository.saveSearchQuery(searchQuery)

        // Search recipes
        return recipeRepository.getRecipes().mapCatching { recipes ->
            recipes.filter { recipe ->
                recipe.name.contains(query, ignoreCase = true) ||
                recipe.cuisine.contains(query, ignoreCase = true)
            }
        }
    }

    private fun <T> Result<T>.mapCatching(transform: (T) -> T): Result<T> =
        when (this) {
            is Result.Success -> Result.Success(transform(data))
            is Result.Error -> this
        }
}
```

### Key Rules

- **No Android imports**: No Context, View, Activity, Fragment, Lifecycle, etc.
- **No serialization annotations**: @Serializable, @SerialName belong in data layer
- **No persistence annotations**: @Entity, @Dao, @Database are in data layer
- **Use Kotlin stdlib**: collections, sequences, extensions, coroutines (suspend, Flow)
- **Suspend for async**: Use suspend operator fun, not callbacks
- **Immutability**: data classes with val, no vars
- **Validation in constructors**: init blocks for invariants

### Unit Tests (JUnit5 + MockK)

```kotlin
// domain/usecase/SearchRecipesUseCaseTest.kt
import io.mockk.*
import kotlinx.coroutines.test.runTest
import kotlin.test.Test
import kotlin.test.assertEquals

class SearchRecipesUseCaseTest {

    private val recipeRepository: RecipeRepository = mockk()
    private val searchQueryRepository: SearchQueryRepository = mockk()
    private val useCase = SearchRecipesUseCase(recipeRepository, searchQueryRepository)

    @Test
    fun testSearchReturnsMatchingRecipes() = runTest {
        val recipes = listOf(
            Recipe(id = "1", name = "Pasta", ...),
            Recipe(id = "2", name = "Salad", ...)
        )
        coEvery { recipeRepository.getRecipes() } returns Result.Success(recipes)
        coEvery { searchQueryRepository.saveSearchQuery(any()) } returns Result.Success(Unit)

        val result = useCase("Pasta")

        assertEquals(1, (result as Result.Success).data.size)
        assertEquals("Pasta", (result).data[0].name)
    }

    @Test
    fun testSearchRejectsShortQuery() = runTest {
        val result = kotlin.runCatching {
            useCase("ab")
        }

        assert(result.isFailure)
    }
}
```

---

## Phase 3: Data ‖ Presentation (Parallel, 3-4 hours each)

Two leads work in parallel to reduce cycle time.

---

### 3a. Data Lead

**Goal**: Implement remote and local data sources, mappers, repository implementations.

#### Responsibilities

1. **DTOs** (@Serializable for Kotlinx.serialization)
   - Mirror API response shapes
   - Can include serialization metadata (@SerialName, @Serializable)
   - Nullable fields as needed

2. **Mappers**
   - DTO → Domain entity
   - Domain entity → Room entity
   - One mapper class per mapping direction

3. **Retrofit Service**
   - @GET, @POST, @PUT, @DELETE annotations
   - Suspend functions
   - Proper error handling with interceptors

4. **Room Database**
   - @Entity classes (can be same as DTO or separate)
   - @Dao interfaces with queries
   - @Database class
   - Type converters for complex types

5. **Repository Implementation** (@Binds in Hilt)
   - Implement domain repository interface
   - Coordinate remote and local sources
   - Cache strategy (write-through, write-behind, etc.)

#### Example: Recipe Data Layer

```kotlin
// data/remote/dto/RecipeDto.kt
package com.example.recipe.data.remote.dto

import kotlinx.serialization.Serializable

@Serializable
data class RecipeDto(
    val id: String,
    val name: String,
    val ingredients: List<IngredientDto>,
    val instructions: List<InstructionDto>,
    val cuisine: String,
    val servings: Int,
    val prepTime: Int,
    val cookTime: Int,
    val difficulty: String,
    val rating: Float = 0f
)

@Serializable
data class IngredientDto(
    val name: String,
    val amount: Float,
    val unit: String
)

@Serializable
data class InstructionDto(
    val step: Int,
    val description: String
)

// data/remote/RecipeService.kt
package com.example.recipe.data.remote

import retrofit2.http.GET
import retrofit2.http.Path

interface RecipeService {
    @GET("/recipes")
    suspend fun getRecipes(): List<RecipeDto>

    @GET("/recipes/{id}")
    suspend fun getRecipe(@Path("id") id: String): RecipeDto
}

// data/mapper/RecipeMapper.kt
package com.example.recipe.data.mapper

import com.example.recipe.data.remote.dto.RecipeDto
import com.example.recipe.domain.entity.*

fun RecipeDto.toDomain(): Recipe = Recipe(
    id = id,
    name = name,
    ingredients = ingredients.map { it.toDomain() },
    instructions = instructions.map { it.toDomain() },
    cuisine = cuisine,
    servings = servings,
    prepTime = prepTime,
    cookTime = cookTime,
    difficulty = Difficulty.valueOf(difficulty),
    rating = rating
)

fun IngredientDto.toDomain(): Ingredient = Ingredient(
    name = name,
    amount = amount,
    unit = unit
)

fun InstructionDto.toDomain(): Instruction = Instruction(
    step = step,
    description = description
)

// data/local/entity/RecipeEntity.kt
package com.example.recipe.data.local.entity

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "recipes")
data class RecipeEntity(
    @PrimaryKey val id: String,
    val name: String,
    val cuisine: String,
    val servings: Int,
    val prepTime: Int,
    val cookTime: Int,
    val difficulty: String,
    val rating: Float
)

// data/local/RecipeDao.kt
package com.example.recipe.data.local

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.Query
import kotlinx.coroutines.flow.Flow

@Dao
interface RecipeDao {
    @Insert(onConflict = androidx.room.OnConflictStrategy.REPLACE)
    suspend fun insertRecipes(recipes: List<RecipeEntity>)

    @Query("SELECT * FROM recipes")
    fun getAllRecipes(): Flow<List<RecipeEntity>>

    @Query("SELECT * FROM recipes WHERE id = :id")
    suspend fun getRecipe(id: String): RecipeEntity?

    @Query("DELETE FROM recipes")
    suspend fun clearAll()
}

// data/local/AppDatabase.kt
package com.example.recipe.data.local

import androidx.room.Database
import androidx.room.RoomDatabase

@Database(entities = [RecipeEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun recipeDao(): RecipeDao
}

// data/repository/RecipeRepositoryImpl.kt
package com.example.recipe.data.repository

import com.example.recipe.data.local.RecipeDao
import com.example.recipe.data.mapper.toDomain
import com.example.recipe.data.remote.RecipeService
import com.example.recipe.domain.entity.Recipe
import com.example.recipe.domain.error.DomainException
import com.example.recipe.domain.repository.RecipeRepository
import com.example.recipe.domain.result.Result
import javax.inject.Inject

class RecipeRepositoryImpl @Inject constructor(
    private val recipeService: RecipeService,
    private val recipeDao: RecipeDao
) : RecipeRepository {

    override suspend fun getRecipes(): Result<List<Recipe>> = try {
        val dtos = recipeService.getRecipes()
        val recipes = dtos.map { it.toDomain() }
        Result.Success(recipes)
    } catch (e: Exception) {
        Result.Error(DomainException.NetworkError(e))
    }

    override suspend fun getRecipe(id: String): Result<Recipe> = try {
        val dto = recipeService.getRecipe(id)
        val recipe = dto.toDomain()
        Result.Success(recipe)
    } catch (e: Exception) {
        Result.Error(DomainException.NetworkError(e))
    }

    override suspend fun saveRecipe(recipe: Recipe): Result<Unit> = TODO()
    override suspend fun deleteRecipe(id: String): Result<Unit> = TODO()
}
```

---

### 3b. Presentation Lead

**Goal**: Implement ViewModels, UiState, Compose screens, and Navigation.

#### Responsibilities

1. **UiState Sealed Classes**
   - One per screen (e.g., RecipeListUiState, RecipeDetailUiState)
   - Loading, Success, Error states
   - Include all data needed for UI

2. **ViewModels** (@HiltViewModel)
   - Inject use cases
   - Expose StateFlow<UiState>
   - Handle events (onClick, onSubmit, etc.)
   - Launch coroutines in viewModelScope

3. **Compose Screens**
   - @Composable functions
   - Accept ViewModel via hiltViewModel()
   - Render based on UiState
   - Material Design 3 styling

4. **Navigation**
   - NavHost setup
   - Route definitions
   - Argument passing

#### Example: Recipe Presentation Layer

```kotlin
// presentation/uistate/RecipeListUiState.kt
package com.example.recipe.presentation.uistate

import com.example.recipe.domain.entity.Recipe

sealed class RecipeListUiState {
    object Loading : RecipeListUiState()
    data class Success(val recipes: List<Recipe>) : RecipeListUiState()
    data class Error(val message: String) : RecipeListUiState()
}

sealed class RecipeDetailUiState {
    object Loading : RecipeDetailUiState()
    data class Success(val recipe: Recipe) : RecipeDetailUiState()
    data class Error(val message: String) : RecipeDetailUiState()
}

// presentation/viewmodel/RecipeListViewModel.kt
package com.example.recipe.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.recipe.domain.result.Result
import com.example.recipe.domain.usecase.GetRecipesUseCase
import com.example.recipe.presentation.uistate.RecipeListUiState
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import javax.inject.Inject

@HiltViewModel
class RecipeListViewModel @Inject constructor(
    private val getRecipesUseCase: GetRecipesUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow<RecipeListUiState>(RecipeListUiState.Loading)
    val uiState: StateFlow<RecipeListUiState> = _uiState.asStateFlow()

    init {
        loadRecipes()
    }

    private fun loadRecipes() {
        viewModelScope.launch {
            when (val result = getRecipesUseCase()) {
                is Result.Success -> {
                    _uiState.value = RecipeListUiState.Success(result.data)
                }
                is Result.Error -> {
                    _uiState.value = RecipeListUiState.Error(
                        result.exception.message ?: "Unknown error"
                    )
                }
            }
        }
    }

    fun onRetry() {
        loadRecipes()
    }
}

// presentation/viewmodel/RecipeDetailViewModel.kt
package com.example.recipe.presentation.viewmodel

import androidx.lifecycle.SavedStateHandle
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.recipe.domain.result.Result
import com.example.recipe.domain.usecase.GetRecipeUseCase
import com.example.recipe.presentation.uistate.RecipeDetailUiState
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import javax.inject.Inject

@HiltViewModel
class RecipeDetailViewModel @Inject constructor(
    private val getRecipeUseCase: GetRecipeUseCase,
    savedStateHandle: SavedStateHandle
) : ViewModel() {

    private val recipeId: String = checkNotNull(savedStateHandle.get<String>("id"))

    private val _uiState = MutableStateFlow<RecipeDetailUiState>(RecipeDetailUiState.Loading)
    val uiState: StateFlow<RecipeDetailUiState> = _uiState.asStateFlow()

    init {
        loadRecipe()
    }

    private fun loadRecipe() {
        viewModelScope.launch {
            when (val result = getRecipeUseCase(recipeId)) {
                is Result.Success -> {
                    _uiState.value = RecipeDetailUiState.Success(result.data)
                }
                is Result.Error -> {
                    _uiState.value = RecipeDetailUiState.Error(
                        result.exception.message ?: "Unknown error"
                    )
                }
            }
        }
    }
}

// presentation/screen/RecipeListScreen.kt
package com.example.recipe.presentation.screen

import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Add
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import com.example.recipe.domain.entity.Recipe
import com.example.recipe.presentation.uistate.RecipeListUiState
import com.example.recipe.presentation.viewmodel.RecipeListViewModel

@Composable
fun RecipeListScreen(
    onNavigateToDetail: (recipeId: String) -> Unit,
    viewModel: RecipeListViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()

    Scaffold(
        floatingActionButton = {
            FloatingActionButton(onClick = { /* TODO */ }) {
                Icon(Icons.Default.Add, contentDescription = "Add recipe")
            }
        }
    ) { paddingValues ->
        when (uiState) {
            is RecipeListUiState.Loading -> {
                LoadingIndicator(modifier = Modifier.padding(paddingValues))
            }
            is RecipeListUiState.Success -> {
                RecipeListContent(
                    recipes = (uiState as RecipeListUiState.Success).recipes,
                    onRecipeClick = onNavigateToDetail,
                    modifier = Modifier.padding(paddingValues)
                )
            }
            is RecipeListUiState.Error -> {
                ErrorMessage(
                    message = (uiState as RecipeListUiState.Error).message,
                    onRetry = viewModel::onRetry,
                    modifier = Modifier.padding(paddingValues)
                )
            }
        }
    }
}

@Composable
fun RecipeListContent(
    recipes: List<Recipe>,
    onRecipeClick: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    if (recipes.isEmpty()) {
        Box(
            modifier = modifier
                .fillMaxSize()
                .wrapContentSize(Alignment.Center),
            contentAlignment = Alignment.Center
        ) {
            Text("No recipes found")
        }
        return
    }

    LazyColumn(modifier = modifier) {
        items(recipes.size) { index ->
            RecipeCard(
                recipe = recipes[index],
                onClick = { onRecipeClick(recipes[index].id) },
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(8.dp)
            )
        }
    }
}

@Composable
fun RecipeCard(
    recipe: Recipe,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        modifier = modifier.clickable(onClick = onClick),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp)
        ) {
            Text(
                text = recipe.name,
                style = MaterialTheme.typography.titleLarge
            )
            Text(
                text = "${recipe.cuisine} • ${recipe.servings} servings",
                style = MaterialTheme.typography.bodyMedium
            )
            Text(
                text = "Rating: ${recipe.rating} ⭐",
                style = MaterialTheme.typography.bodySmall
            )
        }
    }
}

@Composable
fun LoadingIndicator(modifier: Modifier = Modifier) {
    Box(
        modifier = modifier
            .fillMaxSize()
            .wrapContentSize(Alignment.Center),
        contentAlignment = Alignment.Center
    ) {
        CircularProgressIndicator()
    }
}

@Composable
fun ErrorMessage(
    message: String,
    onRetry: () -> Unit,
    modifier: Modifier = Modifier
) {
    Column(
        modifier = modifier
            .fillMaxSize()
            .wrapContentSize(Alignment.Center),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        Text(text = message, style = MaterialTheme.typography.bodyLarge)
        Button(onClick = onRetry) {
            Text("Retry")
        }
    }
}

// presentation/screen/RecipeDetailScreen.kt
package com.example.recipe.presentation.screen

import androidx.compose.foundation.layout.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.ArrowBack
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import com.example.recipe.domain.entity.Recipe
import com.example.recipe.presentation.uistate.RecipeDetailUiState
import com.example.recipe.presentation.viewmodel.RecipeDetailViewModel

@Composable
fun RecipeDetailScreen(
    onBack: () -> Unit,
    viewModel: RecipeDetailViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Recipe Details") },
                navigationIcon = {
                    IconButton(onClick = onBack) {
                        Icon(Icons.Default.ArrowBack, contentDescription = "Back")
                    }
                }
            )
        }
    ) { paddingValues ->
        when (uiState) {
            is RecipeDetailUiState.Loading -> {
                LoadingIndicator(modifier = Modifier.padding(paddingValues))
            }
            is RecipeDetailUiState.Success -> {
                RecipeDetailContent(
                    recipe = (uiState as RecipeDetailUiState.Success).recipe,
                    modifier = Modifier.padding(paddingValues)
                )
            }
            is RecipeDetailUiState.Error -> {
                ErrorMessage(
                    message = (uiState as RecipeDetailUiState.Error).message,
                    onRetry = { },
                    modifier = Modifier.padding(paddingValues)
                )
            }
        }
    }
}

@Composable
fun RecipeDetailContent(
    recipe: Recipe,
    modifier: Modifier = Modifier
) {
    LazyColumn(
        modifier = modifier
            .fillMaxSize()
            .padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        item {
            Text(text = recipe.name, style = MaterialTheme.typography.displaySmall)
        }
        item {
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                Text("${recipe.servings} servings")
                Text("${recipe.prepTime}m prep")
                Text("${recipe.cookTime}m cook")
            }
        }
        item {
            Text("Ingredients:", style = MaterialTheme.typography.titleMedium)
            Column {
                recipe.ingredients.forEach { ingredient ->
                    Text("• ${ingredient.amount} ${ingredient.unit} ${ingredient.name}")
                }
            }
        }
        item {
            Text("Instructions:", style = MaterialTheme.typography.titleMedium)
            Column {
                recipe.instructions.forEach { instruction ->
                    Text("${instruction.step}. ${instruction.description}")
                }
            }
        }
    }
}

// navigation/AppNavHost.kt
package com.example.recipe.navigation

import androidx.compose.runtime.Composable
import androidx.navigation.NavHostController
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import com.example.recipe.presentation.screen.RecipeDetailScreen
import com.example.recipe.presentation.screen.RecipeListScreen

@Composable
fun AppNavHost(
    navController: NavHostController = rememberNavController()
) {
    NavHost(navController = navController, startDestination = "recipe-list") {
        composable(route = "recipe-list") {
            RecipeListScreen(
                onNavigateToDetail = { id ->
                    navController.navigate("recipe-detail/$id")
                }
            )
        }
        composable(route = "recipe-detail/{id}") { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id") ?: return@composable
            RecipeDetailScreen(
                onBack = { navController.popBackStack() }
            )
        }
    }
}
```

---

## Phase 4: Integration (1-2 hours)

**Goal**: Wire everything together with Hilt, run linters, verify layer boundaries.

### Tasks

1. **Hilt Modules** — Create @Module classes with @Provides and @Binds

```kotlin
// di/RepositoryModule.kt
package com.example.di

import com.example.recipe.data.repository.RecipeRepositoryImpl
import com.example.recipe.data.repository.SearchQueryRepositoryImpl
import com.example.recipe.domain.repository.RecipeRepository
import com.example.recipe.domain.repository.SearchQueryRepository
import dagger.Bind
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
interface RepositoryModule {
    @Binds
    @Singleton
    fun bindRecipeRepository(impl: RecipeRepositoryImpl): RecipeRepository

    @Binds
    @Singleton
    fun bindSearchQueryRepository(impl: SearchQueryRepositoryImpl): SearchQueryRepository
}

// di/NetworkModule.kt
package com.example.di

import com.example.recipe.data.remote.RecipeService
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import kotlinx.serialization.json.Json
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import retrofit2.Retrofit
import retrofit2.converter.kotlinx.serialization.asConverterFactory
import java.util.concurrent.TimeUnit
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .build()

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(
                Json.asConverterFactory("application/json".toMediaType())
            )
            .build()

    @Provides
    @Singleton
    fun provideRecipeService(retrofit: Retrofit): RecipeService =
        retrofit.create(RecipeService::class.java)
}

// di/DatabaseModule.kt
package com.example.di

import android.content.Context
import androidx.room.Room
import com.example.recipe.data.local.AppDatabase
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideAppDatabase(
        @ApplicationContext context: Context
    ): AppDatabase =
        Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        ).build()

    @Provides
    @Singleton
    fun provideRecipeDao(database: AppDatabase) =
        database.recipeDao()
}
```

2. **Application Setup** — @HiltAndroidApp and MainActivity

```kotlin
// App.kt
package com.example

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class App : Application()

// MainActivity.kt
package com.example

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import com.example.recipe.navigation.AppNavHost
import com.example.ui.theme.RecipeAppTheme
import dagger.hilt.android.AndroidEntryPoint

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            RecipeAppTheme {
                Surface(color = MaterialTheme.colorScheme.background) {
                    AppNavHost()
                }
            }
        }
    }
}
```

3. **NavHost Registration** — Ensure all routes are in AppNavHost

4. **Layer Audits**
   - Domain: Zero Android imports ✓
   - Data: Only Android + Domain ✓
   - Presentation: Only Domain ✓
   - DI: Wires all ✓

5. **Lint & Format**
   - `./gradlew ktlint` — Check style
   - `./gradlew detekt` — Check quality
   - `./gradlew ktlintFormat` — Auto-fix (if available)

```bash
./gradlew ktlint
./gradlew detekt
./gradlew ktlintFormat
```

---

## Summary

| Phase | Lead | Duration | Output |
|-------|------|----------|--------|
| 1. Architect | Architect | 2-4h | ADR.md, no code |
| 2. Domain | Domain Lead | 2-3h | entities, use cases, repo interfaces, exceptions |
| 3. Data ‖ Presentation | 2 leads in parallel | 3-4h each | DTOs, mappers, Retrofit, Room, VMs, screens, navigation |
| 4. Integration | Integration Lead | 1-2h | Hilt modules, linting, audits, build |
| **Total** | — | ~10-15h | Complete, tested feature |

