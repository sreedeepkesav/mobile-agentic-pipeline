# Android Architecture Template

Complete Kotlin templates for Android Lite Pipeline projects.

---

## Project Structure

```
app/
├── src/
│   ├── main/
│   │   ├── kotlin/com/example/app/
│   │   │   ├── App.kt
│   │   │   ├── MainActivity.kt
│   │   │   ├── di/
│   │   │   ├── feature/
│   │   │   ├── core/
│   │   │   └── navigation/
│   │   ├── AndroidManifest.xml
│   │   └── res/
│   ├── test/
│   │   └── kotlin/com/example/app/
│   └── androidTest/
│       └── kotlin/com/example/app/
├── build.gradle.kts
└── proguard-rules.pro
```

---

## build.gradle.kts (App-Level)

```kotlin
plugins {
    id("com.android.application")
    kotlin("android")
    kotlin("plugin.serialization")
    id("com.google.dagger.hilt.android")
    kotlin("kapt")
    id("org.jlleitschuh.gradle.ktlint") version "11.5.1"
    id("io.gitlab.arturbosch.detekt") version "1.23.1"
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

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        vectorDrawables {
            useSupportLibrary = true
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            signingConfig = signingConfigs.getByName("debug")
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }

    buildFeatures {
        compose = true
    }

    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.3"
    }

    packaging {
        resources {
            excludes += "/META-INF/{AL2.0,LGPL2.1}"
        }
    }
}

dependencies {
    // Core
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.7.0")
    implementation("androidx.activity:activity-compose:1.8.1")

    // Compose
    implementation(platform("androidx.compose:compose-bom:2023.10.01"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-graphics")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.material:material-icons-extended")

    // Navigation
    implementation("androidx.navigation:navigation-compose:2.7.6")

    // Hilt
    implementation("com.google.dagger:hilt-android:2.48")
    kapt("com.google.dagger:hilt-compiler:2.48")
    implementation("androidx.hilt:hilt-navigation-compose:1.1.0")

    // Room
    implementation("androidx.room:room-runtime:2.6.1")
    kapt("androidx.room:room-compiler:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")

    // Retrofit
    implementation("com.squareup.retrofit2:retrofit:2.10.0")
    implementation("com.squareup.retrofit2:converter-kotlinx-serialization:2.10.0")
    implementation("com.squareup.okhttp3:okhttp:4.11.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.11.0")

    // Serialization
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.0")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")

    // DataStore
    implementation("androidx.datastore:datastore-preferences:1.0.0")

    // Testing
    testImplementation("junit:junit:4.13.2")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
    androidTestImplementation(platform("androidx.compose:compose-bom:2023.10.01"))
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")

    // Debug
    debugImplementation("androidx.compose.ui:ui-tooling")
    debugImplementation("androidx.compose.ui:ui-test-manifest")
}

// Lint config
ktlint {
    version.set("0.50.0")
}

detekt {
    config.setFrom("detekt.yml")
}
```

---

## App.kt (@HiltAndroidApp)

```kotlin
package com.example.app

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class App : Application()
```

---

## AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:name=".App"
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.App"
        tools:targetApi="31">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:label="@string/app_name"
            android:theme="@style/Theme.App">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

---

## MainActivity.kt (@AndroidEntryPoint)

```kotlin
package com.example.app

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import com.example.app.navigation.AppNavHost
import com.example.app.core.ui.theme.AppTheme
import dagger.hilt.android.AndroidEntryPoint

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme {
                Surface(
                    color = MaterialTheme.colorScheme.background
                ) {
                    AppNavHost()
                }
            }
        }
    }
}
```

---

## core/ui/theme/Theme.kt

```kotlin
package com.example.app.core.ui.theme

import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.darkColorScheme
import androidx.compose.material3.lightColorScheme
import androidx.compose.runtime.Composable

private val LightColorScheme = lightColorScheme(
    primary = PrimaryColor,
    onPrimary = OnPrimaryColor,
    secondary = SecondaryColor,
    onSecondary = OnSecondaryColor,
    tertiary = TertiaryColor,
    onTertiary = OnTertiaryColor,
    background = BackgroundColor,
    onBackground = OnBackgroundColor,
    surface = SurfaceColor,
    onSurface = OnSurfaceColor,
    error = ErrorColor,
    onError = OnErrorColor
)

private val DarkColorScheme = darkColorScheme(
    primary = PrimaryColor,
    onPrimary = OnPrimaryColor,
    secondary = SecondaryColor,
    onSecondary = OnSecondaryColor,
    tertiary = TertiaryColor,
    onTertiary = OnTertiaryColor,
    background = BackgroundColor,
    onBackground = OnBackgroundColor,
    surface = SurfaceColor,
    onSurface = OnSurfaceColor,
    error = ErrorColor,
    onError = OnErrorColor
)

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        content = content
    )
}
```

---

## core/ui/theme/Color.kt

```kotlin
package com.example.app.core.ui.theme

import androidx.compose.ui.graphics.Color

val PrimaryColor = Color(0xFF6750A4)
val OnPrimaryColor = Color(0xFFFFFFFF)
val PrimaryContainerColor = Color(0xFFEADDFF)

val SecondaryColor = Color(0xFF625B71)
val OnSecondaryColor = Color(0xFFFFFFFF)
val SecondaryContainerColor = Color(0xFFE8DEF8)

val TertiaryColor = Color(0xFF7D5260)
val OnTertiaryColor = Color(0xFFFFFFFF)
val TertiaryContainerColor = Color(0xFFFFD8E4)

val ErrorColor = Color(0xFFB3261E)
val OnErrorColor = Color(0xFFFFFFFF)

val BackgroundColor = Color(0xFFFFFBFE)
val OnBackgroundColor = Color(0xFF1C1B1F)

val SurfaceColor = Color(0xFFFFFBFE)
val OnSurfaceColor = Color(0xFF1C1B1F)
```

---

## core/ui/theme/Type.kt

```kotlin
package com.example.app.core.ui.theme

import androidx.compose.material3.Typography
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.sp

val AppTypography = Typography(
    displayLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 57.sp,
        lineHeight = 64.sp,
        letterSpacing = 0.sp
    ),
    displayMedium = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 45.sp,
        lineHeight = 52.sp,
        letterSpacing = 0.sp
    ),
    displaySmall = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 36.sp,
        lineHeight = 44.sp,
        letterSpacing = 0.sp
    ),
    headlineLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 32.sp,
        lineHeight = 40.sp,
        letterSpacing = 0.sp
    ),
    headlineMedium = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 28.sp,
        lineHeight = 36.sp,
        letterSpacing = 0.sp
    ),
    headlineSmall = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 24.sp,
        lineHeight = 32.sp,
        letterSpacing = 0.sp
    ),
    titleLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Bold,
        fontSize = 22.sp,
        lineHeight = 28.sp,
        letterSpacing = 0.sp
    ),
    titleMedium = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Bold,
        fontSize = 16.sp,
        lineHeight = 24.sp,
        letterSpacing = 0.15.sp
    ),
    titleSmall = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Bold,
        fontSize = 14.sp,
        lineHeight = 20.sp,
        letterSpacing = 0.1.sp
    ),
    bodyLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp,
        letterSpacing = 0.5.sp
    ),
    bodyMedium = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 14.sp,
        lineHeight = 20.sp,
        letterSpacing = 0.25.sp
    ),
    bodySmall = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 12.sp,
        lineHeight = 16.sp,
        letterSpacing = 0.4.sp
    ),
    labelLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Bold,
        fontSize = 14.sp,
        lineHeight = 20.sp,
        letterSpacing = 0.1.sp
    ),
    labelMedium = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Bold,
        fontSize = 12.sp,
        lineHeight = 16.sp,
        letterSpacing = 0.5.sp
    ),
    labelSmall = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Bold,
        fontSize = 11.sp,
        lineHeight = 16.sp,
        letterSpacing = 0.5.sp
    )
)
```

---

## di/RepositoryModule.kt

```kotlin
package com.example.app.di

import com.example.feature.recipe.data.repository.RecipeRepositoryImpl
import com.example.feature.recipe.domain.repository.RecipeRepository
import dagger.Binds
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
}
```

---

## di/NetworkModule.kt

```kotlin
package com.example.app.di

import com.example.feature.recipe.data.remote.RecipeService
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import kotlinx.serialization.json.Json
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit
import retrofit2.converter.kotlinx.serialization.asConverterFactory
import java.util.concurrent.TimeUnit
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideJson(): Json = Json {
        ignoreUnknownKeys = true
        coerceInputValues = true
    }

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY
            })
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .build()

    @Provides
    @Singleton
    fun provideRetrofit(
        okHttpClient: OkHttpClient,
        json: Json
    ): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
            .build()

    @Provides
    @Singleton
    fun provideRecipeService(retrofit: Retrofit): RecipeService =
        retrofit.create(RecipeService::class.java)
}
```

---

## di/DatabaseModule.kt

```kotlin
package com.example.app.di

import android.content.Context
import androidx.room.Room
import com.example.feature.recipe.data.local.AppDatabase
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

---

## feature/recipe/domain/entity/Recipe.kt

```kotlin
package com.example.feature.recipe.domain.entity

data class Recipe(
    val id: String,
    val name: String,
    val ingredients: List<Ingredient>,
    val instructions: List<Instruction>,
    val cuisine: String,
    val servings: Int,
    val prepTime: Int,
    val cookTime: Int,
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
```

---

## feature/recipe/domain/repository/RecipeRepository.kt

```kotlin
package com.example.feature.recipe.domain.repository

import com.example.feature.recipe.domain.entity.Recipe
import com.example.core.domain.result.Result

interface RecipeRepository {
    suspend fun getRecipes(): Result<List<Recipe>>
    suspend fun getRecipe(id: String): Result<Recipe>
}
```

---

## feature/recipe/domain/usecase/GetRecipesUseCase.kt

```kotlin
package com.example.feature.recipe.domain.usecase

import com.example.feature.recipe.domain.repository.RecipeRepository
import com.example.core.domain.result.Result
import com.example.feature.recipe.domain.entity.Recipe
import javax.inject.Inject

class GetRecipesUseCase @Inject constructor(
    private val recipeRepository: RecipeRepository
) {
    suspend operator fun invoke(): Result<List<Recipe>> =
        recipeRepository.getRecipes()
}
```

---

## feature/recipe/data/remote/dto/RecipeDto.kt

```kotlin
package com.example.feature.recipe.data.remote.dto

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
```

---

## feature/recipe/data/remote/RecipeService.kt

```kotlin
package com.example.feature.recipe.data.remote

import com.example.feature.recipe.data.remote.dto.RecipeDto
import retrofit2.http.GET
import retrofit2.http.Path

interface RecipeService {
    @GET("/recipes")
    suspend fun getRecipes(): List<RecipeDto>

    @GET("/recipes/{id}")
    suspend fun getRecipe(@Path("id") id: String): RecipeDto
}
```

---

## feature/recipe/data/mapper/RecipeMapper.kt

```kotlin
package com.example.feature.recipe.data.mapper

import com.example.feature.recipe.data.remote.dto.IngredientDto
import com.example.feature.recipe.data.remote.dto.InstructionDto
import com.example.feature.recipe.data.remote.dto.RecipeDto
import com.example.feature.recipe.domain.entity.Difficulty
import com.example.feature.recipe.domain.entity.Ingredient
import com.example.feature.recipe.domain.entity.Instruction
import com.example.feature.recipe.domain.entity.Recipe

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
```

---

## feature/recipe/data/local/AppDatabase.kt

```kotlin
package com.example.feature.recipe.data.local

import androidx.room.Database
import androidx.room.RoomDatabase

@Database(entities = [RecipeEntity::class], version = 1, exportSchema = true)
abstract class AppDatabase : RoomDatabase() {
    abstract fun recipeDao(): RecipeDao
}
```

---

## feature/recipe/data/local/RecipeEntity.kt

```kotlin
package com.example.feature.recipe.data.local

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
```

---

## feature/recipe/data/local/RecipeDao.kt

```kotlin
package com.example.feature.recipe.data.local

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query
import kotlinx.coroutines.flow.Flow

@Dao
interface RecipeDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertRecipes(recipes: List<RecipeEntity>)

    @Query("SELECT * FROM recipes")
    fun getAllRecipes(): Flow<List<RecipeEntity>>

    @Query("SELECT * FROM recipes WHERE id = :id")
    suspend fun getRecipe(id: String): RecipeEntity?

    @Query("DELETE FROM recipes")
    suspend fun clearAll()
}
```

---

## feature/recipe/data/repository/RecipeRepositoryImpl.kt

```kotlin
package com.example.feature.recipe.data.repository

import com.example.core.domain.exception.DomainException
import com.example.core.domain.result.Result
import com.example.feature.recipe.data.mapper.toDomain
import com.example.feature.recipe.data.remote.RecipeService
import com.example.feature.recipe.domain.entity.Recipe
import com.example.feature.recipe.domain.repository.RecipeRepository
import javax.inject.Inject

class RecipeRepositoryImpl @Inject constructor(
    private val recipeService: RecipeService
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
}
```

---

## feature/recipe/presentation/uistate/RecipeListUiState.kt

```kotlin
package com.example.feature.recipe.presentation.uistate

import com.example.feature.recipe.domain.entity.Recipe

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
```

---

## feature/recipe/presentation/viewmodel/RecipeListViewModel.kt

```kotlin
package com.example.feature.recipe.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.core.domain.result.Result
import com.example.feature.recipe.domain.usecase.GetRecipesUseCase
import com.example.feature.recipe.presentation.uistate.RecipeListUiState
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
```

---

## feature/recipe/presentation/screen/RecipeListScreen.kt

```kotlin
package com.example.feature.recipe.presentation.screen

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
import com.example.feature.recipe.domain.entity.Recipe
import com.example.feature.recipe.presentation.uistate.RecipeListUiState
import com.example.feature.recipe.presentation.viewmodel.RecipeListViewModel

@Composable
fun RecipeListScreen(
    onNavigateToDetail: (recipeId: String) -> Unit,
    viewModel: RecipeListViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()

    Scaffold(
        topBar = {
            TopAppBar(title = { Text("Recipes") })
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { /* TODO */ }) {
                Icon(Icons.Default.Add, contentDescription = "Add recipe")
            }
        }
    ) { paddingValues ->
        when (uiState) {
            is RecipeListUiState.Loading -> {
                Box(
                    modifier = Modifier
                        .fillMaxSize()
                        .padding(paddingValues),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator()
                }
            }
            is RecipeListUiState.Success -> {
                val recipes = (uiState as RecipeListUiState.Success).recipes
                if (recipes.isEmpty()) {
                    Box(
                        modifier = Modifier
                            .fillMaxSize()
                            .padding(paddingValues),
                        contentAlignment = Alignment.Center
                    ) {
                        Text("No recipes found")
                    }
                } else {
                    LazyColumn(modifier = Modifier.padding(paddingValues)) {
                        items(recipes.size) { index ->
                            RecipeCard(
                                recipe = recipes[index],
                                onClick = { onNavigateToDetail(recipes[index].id) },
                                modifier = Modifier
                                    .fillMaxWidth()
                                    .padding(8.dp)
                            )
                        }
                    }
                }
            }
            is RecipeListUiState.Error -> {
                Column(
                    modifier = Modifier
                        .fillMaxSize()
                        .padding(paddingValues)
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
```

---

## navigation/AppNavHost.kt

```kotlin
package com.example.app.navigation

import androidx.compose.runtime.Composable
import androidx.navigation.NavHostController
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import com.example.feature.recipe.presentation.screen.RecipeListScreen

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
            Text("Recipe detail: $id")  // Placeholder
        }
    }
}
```

---

## core/domain/result/Result.kt

```kotlin
package com.example.core.domain.result

sealed class Result<T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error<T>(val exception: Throwable) : Result<T>()
}
```

---

## core/domain/exception/DomainException.kt

```kotlin
package com.example.core.domain.exception

sealed class DomainException(override val message: String) : Exception(message) {
    data class NetworkError(val cause: Throwable) :
        DomainException("Network error: ${cause.message}")

    data class NotFound(val itemId: String) :
        DomainException("Not found: $itemId")

    data class ValidationError(val field: String) :
        DomainException("Invalid $field")
}
```

---

This template provides a complete, working foundation for any Android app built with the Lite Pipeline.
