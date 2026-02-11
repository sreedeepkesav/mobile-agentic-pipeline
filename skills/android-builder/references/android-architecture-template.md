# Android Architecture Template

This document provides concrete Kotlin/Jetpack Compose code templates that follow the standards defined in this pipeline.

## Project Structure

```
MyApp/
├── gradle/
│   └── wrapper/
├── app/
│   ├── build.gradle.kts
│   ├── src/
│   │   ├── main/
│   │   │   ├── AndroidManifest.xml
│   │   │   ├── kotlin/com/example/myapp/
│   │   │   │   ├── MyApp.kt                          # @HiltAndroidApp
│   │   │   │   ├── MainActivity.kt                   # @AndroidEntryPoint, NavHost
│   │   │   │   ├── di/
│   │   │   │   │   ├── AppModule.kt
│   │   │   │   │   ├── NetworkModule.kt              # @Provides Retrofit, Services
│   │   │   │   │   ├── DatabaseModule.kt             # @Provides Database, DAOs
│   │   │   │   │   ├── AuthModule.kt                 # @Binds AuthRepository
│   │   │   │   │   └── UserModule.kt                 # @Binds UserRepository
│   │   │   │   ├── domain/
│   │   │   │   │   ├── entity/
│   │   │   │   │   │   ├── User.kt
│   │   │   │   │   │   └── AuthToken.kt
│   │   │   │   │   ├── usecase/
│   │   │   │   │   │   ├── LoginUseCase.kt
│   │   │   │   │   │   ├── GetUserUseCase.kt
│   │   │   │   │   │   └── WatchUserUseCase.kt
│   │   │   │   │   ├── repository/
│   │   │   │   │   │   ├── AuthRepository.kt
│   │   │   │   │   │   └── UserRepository.kt
│   │   │   │   │   └── exception/
│   │   │   │   │       └── DomainException.kt
│   │   │   │   ├── data/
│   │   │   │   │   ├── dto/
│   │   │   │   │   │   ├── UserDto.kt
│   │   │   │   │   │   └── LoginDto.kt
│   │   │   │   │   ├── mapper/
│   │   │   │   │   │   ├── UserMapper.kt
│   │   │   │   │   │   └── AuthMapper.kt
│   │   │   │   │   ├── network/
│   │   │   │   │   │   ├── AuthService.kt
│   │   │   │   │   │   ├── UserService.kt
│   │   │   │   │   │   └── AuthInterceptor.kt
│   │   │   │   │   ├── local/
│   │   │   │   │   │   ├── UserEntity.kt
│   │   │   │   │   │   ├── UserDao.kt
│   │   │   │   │   │   ├── AppDatabase.kt
│   │   │   │   │   │   └── TokenCache.kt
│   │   │   │   │   └── repository/
│   │   │   │   │       ├── AuthRepositoryImpl.kt
│   │   │   │   │       └── UserRepositoryImpl.kt
│   │   │   │   ├── presentation/
│   │   │   │   │   ├── theme/
│   │   │   │   │   │   ├── Color.kt
│   │   │   │   │   │   ├── Type.kt
│   │   │   │   │   │   └── Theme.kt
│   │   │   │   │   ├── navigation/
│   │   │   │   │   │   └── Route.kt
│   │   │   │   │   └── feature/
│   │   │   │   │       ├── login/
│   │   │   │   │       │   ├── LoginScreen.kt
│   │   │   │   │       │   ├── LoginViewModel.kt
│   │   │   │   │       │   └── LoginUiState.kt
│   │   │   │   │       ├── splash/
│   │   │   │   │       │   ├── SplashScreen.kt
│   │   │   │   │       │   └── SplashViewModel.kt
│   │   │   │   │       └── home/
│   │   │   │   │           ├── HomeScreen.kt
│   │   │   │   │           ├── HomeViewModel.kt
│   │   │   │   │           └── HomeUiState.kt
│   │   │   └── res/
│   │   ├── test/
│   │   │   ├── kotlin/com/example/myapp/
│   │   │   │   ├── domain/
│   │   │   │   │   ├── GetUserUseCaseTest.kt
│   │   │   │   │   └── LoginUseCaseTest.kt
│   │   │   │   ├── data/
│   │   │   │   │   └── UserRepositoryImplTest.kt
│   │   │   │   └── presentation/
│   │   │   │       ├── LoginViewModelTest.kt
│   │   │   │       └── HomeViewModelTest.kt
│   │   └── androidTest/
│   │       └── kotlin/com/example/myapp/
│   │           └── ui/
│   │               ├── LoginScreenTest.kt
│   │               └── HomeScreenTest.kt
│   └── build.gradle.kts
├── build.gradle.kts
├── settings.gradle.kts
└── gradlew
```

---

## Application Class

**MyApp.kt:**
```kotlin
package com.example.myapp

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class MyApp : Application() {
    // Hilt initialization happens automatically
}
```

---

## MainActivity

**MainActivity.kt:**
```kotlin
package com.example.myapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import com.example.myapp.presentation.feature.home.HomeScreen
import com.example.myapp.presentation.feature.login.LoginScreen
import com.example.myapp.presentation.feature.splash.SplashScreen
import com.example.myapp.presentation.navigation.AuthRoute
import com.example.myapp.presentation.navigation.MainRoute
import com.example.myapp.presentation.navigation.Route
import com.example.myapp.presentation.theme.AppTheme
import dagger.hilt.android.AndroidEntryPoint

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    AppNavigation()
                }
            }
        }
    }
}

@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = Route.AuthFlow
    ) {
        // Auth flow with nested navigation
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

        // Main app flow
        navigation<Route.MainFlow>(startDestination = MainRoute.Home) {
            composable<MainRoute.Home> {
                HomeScreen()
            }
        }
    }
}
```

---

## Navigation Routes

**presentation/navigation/Route.kt:**
```kotlin
package com.example.myapp.presentation.navigation

sealed class Route {
    object AuthFlow : Route()
    object MainFlow : Route()
}

sealed class AuthRoute : Route() {
    object Splash : AuthRoute()
    object Login : AuthRoute()
}

sealed class MainRoute : Route() {
    object Home : MainRoute()
}
```

---

## Hilt Modules

**di/AppModule.kt:**
```kotlin
package com.example.myapp.di

import android.content.Context
import androidx.room.Room
import com.example.myapp.data.local.AppDatabase
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    // App-level providers can go here if needed
}
```

**di/NetworkModule.kt:**
```kotlin
package com.example.myapp.di

import com.example.myapp.data.network.AuthInterceptor
import com.example.myapp.data.network.AuthService
import com.example.myapp.data.network.UserService
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
    private const val BASE_URL = "https://api.example.com/"

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
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .addInterceptor(AuthInterceptor())
            .build()

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient, json: Json): Retrofit =
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
            .build()

    @Provides
    @Singleton
    fun provideAuthService(retrofit: Retrofit): AuthService =
        retrofit.create(AuthService::class.java)

    @Provides
    @Singleton
    fun provideUserService(retrofit: Retrofit): UserService =
        retrofit.create(UserService::class.java)
}
```

**di/DatabaseModule.kt:**
```kotlin
package com.example.myapp.di

import android.content.Context
import androidx.room.Room
import com.example.myapp.data.local.AppDatabase
import com.example.myapp.data.local.UserDao
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
    fun provideAppDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "myapp_database")
            .build()

    @Provides
    @Singleton
    fun provideUserDao(database: AppDatabase): UserDao =
        database.userDao()
}
```

**di/AuthModule.kt:**
```kotlin
package com.example.myapp.di

import com.example.myapp.data.repository.AuthRepositoryImpl
import com.example.myapp.domain.repository.AuthRepository
import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
interface AuthModule {
    @Binds
    @Singleton
    fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository
}
```

**di/UserModule.kt:**
```kotlin
package com.example.myapp.di

import com.example.myapp.data.repository.UserRepositoryImpl
import com.example.myapp.domain.repository.UserRepository
import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
interface UserModule {
    @Binds
    @Singleton
    fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}
```

---

## Domain Layer

**domain/entity/User.kt:**
```kotlin
package com.example.myapp.domain.entity

data class User(
    val id: String,
    val email: String,
    val name: String
)
```

**domain/entity/AuthToken.kt:**
```kotlin
package com.example.myapp.domain.entity

data class AuthToken(
    val token: String,
    val refreshToken: String,
    val expiresIn: Long,
    val issuedAt: Long = System.currentTimeMillis()
)
```

**domain/exception/DomainException.kt:**
```kotlin
package com.example.myapp.domain.exception

sealed class DomainException(
    message: String? = null,
    cause: Throwable? = null
) : Exception(message, cause) {
    data class InvalidInput(val field: String) : DomainException("Invalid input: $field")
    object NetworkError : DomainException("Network error occurred")
    object TimeoutError : DomainException("Request timeout")
    object ServerError : DomainException("Server error")
    object AuthenticationError : DomainException("Authentication failed")
    data class Unknown(override val message: String = "Unknown error") : DomainException(message)
}
```

**domain/repository/AuthRepository.kt:**
```kotlin
package com.example.myapp.domain.repository

import com.example.myapp.domain.entity.AuthToken

interface AuthRepository {
    suspend fun login(email: String, password: String): Result<AuthToken>
    suspend fun logout(): Result<Unit>
    suspend fun refreshToken(refreshToken: String): Result<AuthToken>
    suspend fun getCachedToken(): Result<AuthToken?>
}
```

**domain/repository/UserRepository.kt:**
```kotlin
package com.example.myapp.domain.repository

import com.example.myapp.domain.entity.User
import kotlinx.coroutines.flow.Flow

interface UserRepository {
    suspend fun getUser(id: String): Result<User>
    suspend fun saveUser(user: User): Result<Unit>
    fun watchUser(id: String): Flow<User>
}
```

**domain/usecase/LoginUseCase.kt:**
```kotlin
package com.example.myapp.domain.usecase

import com.example.myapp.domain.entity.AuthToken
import com.example.myapp.domain.repository.AuthRepository
import javax.inject.Inject

class LoginUseCase @Inject constructor(
    private val authRepository: AuthRepository
) {
    suspend operator fun invoke(email: String, password: String): Result<AuthToken> =
        authRepository.login(email, password)
}
```

**domain/usecase/GetUserUseCase.kt:**
```kotlin
package com.example.myapp.domain.usecase

import com.example.myapp.domain.entity.User
import com.example.myapp.domain.repository.UserRepository
import javax.inject.Inject

class GetUserUseCase @Inject constructor(
    private val userRepository: UserRepository
) {
    suspend operator fun invoke(id: String): Result<User> =
        userRepository.getUser(id)
}
```

**domain/usecase/WatchUserUseCase.kt:**
```kotlin
package com.example.myapp.domain.usecase

import com.example.myapp.domain.entity.User
import com.example.myapp.domain.repository.UserRepository
import kotlinx.coroutines.flow.Flow
import javax.inject.Inject

class WatchUserUseCase @Inject constructor(
    private val userRepository: UserRepository
) {
    operator fun invoke(id: String): Flow<User> =
        userRepository.watchUser(id)
}
```

---

## Data Layer

**data/dto/LoginDto.kt:**
```kotlin
package com.example.myapp.data.dto

import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

@Serializable
data class LoginRequestDto(
    val email: String,
    val password: String
)

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
```

**data/dto/UserDto.kt:**
```kotlin
package com.example.myapp.data.dto

import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

@Serializable
data class UserDto(
    @SerialName("user_id")
    val userId: String,
    @SerialName("email_address")
    val email: String,
    val name: String
)
```

**data/mapper/UserMapper.kt:**
```kotlin
package com.example.myapp.data.mapper

import com.example.myapp.data.dto.UserDto
import com.example.myapp.data.local.UserEntity
import com.example.myapp.domain.entity.User

fun UserDto.toDomain(): User =
    User(id = userId, email = email, name = name)

fun UserDto.toEntity(): UserEntity =
    UserEntity(id = userId, email = email, name = name, cachedAt = System.currentTimeMillis())

fun UserEntity.toDomain(): User =
    User(id = id, email = email, name = name)
```

**data/network/AuthService.kt:**
```kotlin
package com.example.myapp.data.network

import com.example.myapp.data.dto.LoginRequestDto
import com.example.myapp.data.dto.LoginResponseDto
import retrofit2.http.Body
import retrofit2.http.POST

interface AuthService {
    @POST("auth/login")
    suspend fun login(@Body request: LoginRequestDto): LoginResponseDto

    @POST("auth/logout")
    suspend fun logout()
}
```

**data/network/UserService.kt:**
```kotlin
package com.example.myapp.data.network

import com.example.myapp.data.dto.UserDto
import retrofit2.http.GET
import retrofit2.http.Path

interface UserService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): UserDto
}
```

**data/network/AuthInterceptor.kt:**
```kotlin
package com.example.myapp.data.network

import okhttp3.Interceptor
import okhttp3.Response

class AuthInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        // Add auth header (e.g., JWT token from cache)
        val authenticatedRequest = request.newBuilder()
            .addHeader("Authorization", "Bearer token_here")
            .build()
        return chain.proceed(authenticatedRequest)
    }
}
```

**data/local/UserEntity.kt:**
```kotlin
package com.example.myapp.data.local

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val email: String,
    val name: String,
    val cachedAt: Long
)
```

**data/local/UserDao.kt:**
```kotlin
package com.example.myapp.data.local

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query
import kotlinx.coroutines.flow.Flow

@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUser(id: String): UserEntity?

    @Query("SELECT * FROM users WHERE id = :id")
    fun watchUser(id: String): Flow<UserEntity>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: UserEntity)

    @Query("DELETE FROM users")
    suspend fun deleteAll()
}
```

**data/local/AppDatabase.kt:**
```kotlin
package com.example.myapp.data.local

import androidx.room.Database
import androidx.room.RoomDatabase

@Database(
    entities = [UserEntity::class],
    version = 1,
    exportSchema = false
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

**data/repository/AuthRepositoryImpl.kt:**
```kotlin
package com.example.myapp.data.repository

import com.example.myapp.data.dto.LoginRequestDto
import com.example.myapp.data.network.AuthService
import com.example.myapp.domain.entity.AuthToken
import com.example.myapp.domain.repository.AuthRepository
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class AuthRepositoryImpl @Inject constructor(
    private val authService: AuthService
) : AuthRepository {
    override suspend fun login(email: String, password: String): Result<AuthToken> =
        runCatching {
            val request = LoginRequestDto(email, password)
            val response = authService.login(request)
            AuthToken(
                token = response.token,
                refreshToken = response.refreshToken,
                expiresIn = response.expiresIn
            )
        }

    override suspend fun logout(): Result<Unit> =
        runCatching {
            authService.logout()
        }

    override suspend fun refreshToken(refreshToken: String): Result<AuthToken> {
        TODO("Implement token refresh logic")
    }

    override suspend fun getCachedToken(): Result<AuthToken?> {
        TODO("Implement cached token retrieval")
    }
}
```

**data/repository/UserRepositoryImpl.kt:**
```kotlin
package com.example.myapp.data.repository

import com.example.myapp.data.local.UserDao
import com.example.myapp.data.mapper.toDomain
import com.example.myapp.data.mapper.toEntity
import com.example.myapp.data.network.UserService
import com.example.myapp.domain.entity.User
import com.example.myapp.domain.repository.UserRepository
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class UserRepositoryImpl @Inject constructor(
    private val userService: UserService,
    private val userDao: UserDao
) : UserRepository {
    override suspend fun getUser(id: String): Result<User> =
        runCatching {
            val dto = userService.getUser(id)
            userDao.insert(dto.toEntity())
            dto.toDomain()
        }

    override suspend fun saveUser(user: User): Result<Unit> =
        runCatching {
            userDao.insert(user.let {
                com.example.myapp.data.local.UserEntity(
                    id = it.id,
                    email = it.email,
                    name = it.name,
                    cachedAt = System.currentTimeMillis()
                )
            })
        }

    override fun watchUser(id: String): Flow<User> =
        userDao.watchUser(id).map { it.toDomain() }
}
```

---

## Presentation Layer

**presentation/theme/Theme.kt:**
```kotlin
package com.example.myapp.presentation.theme

import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.lightColorScheme
import androidx.compose.runtime.Composable
import androidx.compose.ui.graphics.Color

private val LightColorScheme = lightColorScheme(
    primary = Color(0xFF6750A4),
    onPrimary = Color(0xFFFFFFFF),
    primaryContainer = Color(0xFFEADDFF),
    secondary = Color(0xFF625B71),
    error = Color(0xFFB3261E)
)

@Composable
fun AppTheme(content: @Composable () -> Unit) {
    MaterialTheme(
        colorScheme = LightColorScheme,
        typography = Typography(),
        shapes = Shapes(),
        content = content
    )
}
```

**presentation/feature/login/LoginUiState.kt:**
```kotlin
package com.example.myapp.presentation.feature.login

import com.example.myapp.domain.entity.AuthToken

sealed class LoginUiState {
    object Idle : LoginUiState()
    object Loading : LoginUiState()
    data class Success(val token: AuthToken) : LoginUiState()
    data class Error(val message: String) : LoginUiState()
}
```

**presentation/feature/login/LoginViewModel.kt:**
```kotlin
package com.example.myapp.presentation.feature.login

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.myapp.domain.usecase.LoginUseCase
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import javax.inject.Inject

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

**presentation/feature/login/LoginScreen.kt:**
```kotlin
package com.example.myapp.presentation.feature.login

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.input.PasswordVisualTransformation
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.lifecycle.compose.collectAsStateWithLifecycle

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
                        .fillMaxWidth()
                ) {
                    Text("Login", style = MaterialTheme.typography.headlineSmall)
                    Spacer(modifier = Modifier.height(16.dp))

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
                        modifier = Modifier
                            .fillMaxWidth()
                            .height(48.dp)
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
```

---

## Testing Templates

**test/kotlin/LoginViewModelTest.kt:**
```kotlin
package com.example.myapp.presentation.feature.login

import androidx.lifecycle.SavedStateHandle
import com.example.myapp.domain.entity.AuthToken
import com.example.myapp.domain.usecase.LoginUseCase
import io.mockk.coEvery
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.junit.Before
import org.junit.Test
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class LoginViewModelTest {
    private val mockLoginUseCase = mockk<LoginUseCase>()
    private lateinit var viewModel: LoginViewModel

    @Before
    fun setup() {
        viewModel = LoginViewModel(mockLoginUseCase)
    }

    @Test
    fun `login emits Loading then Success`() = runTest {
        val token = AuthToken("jwt...", "refresh...", 3600)
        coEvery { mockLoginUseCase("john@example.com", "password") } returns Result.success(token)

        viewModel.login("john@example.com", "password")
        advanceUntilIdle()

        val state = viewModel.uiState.value
        assertTrue(state is LoginUiState.Success)
        assertEquals(token, (state as LoginUiState.Success).token)
    }
}
```

---

This template provides a solid foundation for any Clean Architecture + Hilt + Jetpack Compose Android project following this pipeline's standards.
