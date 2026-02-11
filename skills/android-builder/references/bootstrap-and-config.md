# Bootstrap and Configuration

## Overview

Bootstrap is the initialization phase that detects the project structure, existing architecture pattern, and creates a stage configuration JSON. It's the gateway to all other pipeline stages.

## Bootstrap Algorithm

### Step 1: Project Detection

Scan project root for Android project indicators:

```
✓ build.gradle.kts                      # Gradle config (Kotlin DSL)
✓ src/main/AndroidManifest.xml          # Android app manifest
✓ src/main/kotlin/                      # Kotlin source root
✓ gradle/wrapper/gradle-wrapper.jar     # Gradle wrapper
✓ .gitignore (contains "*.apk")         # Common Android ignore rules
```

**Outcome:** Confirm Android project or fail gracefully.

### Step 2: Architecture Detection

Scan for folder structure patterns that indicate the current architecture:

#### Clean Architecture
```
src/main/kotlin/com/example/app/
├── domain/
│   ├── entity/
│   ├── usecase/
│   └── repository/
├── data/
│   ├── dto/
│   ├── mapper/
│   ├── repository/
│   ├── network/
│   └── local/
└── presentation/
    ├── feature/
    ├── ui/
    └── navigation/
```
**Signal:** Presence of `domain/`, `data/`, `presentation/` folders. → **Architecture: clean**

#### MVVM
```
src/main/kotlin/com/example/app/
├── viewmodels/
├── ui/
│   └── screens/
├── repositories/
└── models/
```
**Signal:** Presence of `viewmodels/` and `ui/screens/`. → **Architecture: mvvm**

#### MVI
```
src/main/kotlin/com/example/app/
├── intent/
├── state/
├── reducer/
├── effect/
└── views/
```
**Signal:** Presence of `intent/`, `state/`, `reducer/`, `effect/`. → **Architecture: mvi**

#### MVP
```
src/main/kotlin/com/example/app/
├── presenters/
├── views/
└── models/
```
**Signal:** Presence of `presenters/` and `views/`. → **Architecture: mvp**

#### None (Greenfield)
**Signal:** No recognizable architecture structure. → **Architecture: none**

### Step 3: Technology Stack Detection

Scan Kotlin files and `build.gradle.kts` for indicators:

```
Dependency Pattern             → Technology
presence("androidx.compose")   → UI: Jetpack Compose
absence("android.widget.*")    → (confirms Compose, not XML)
presence("com.google.dagger.hilt")  → DI: Hilt
presence("androidx.navigation:navigation-compose")  → Navigation: Compose Navigation
presence("androidx.room:room")  → Persistence: Room
presence("com.squareup.retrofit2:retrofit")  → HTTP: Retrofit
presence("org.jetbrains.kotlinx:kotlinx-serialization")  → Serialization: Kotlinx
presence("io.kotest")          → Test: Kotest or JUnit5 (scan test files)
presence("io.mockk")           → Mocking: MockK
```

### Step 4: Load Pipeline Memory

Load `pipeline_memory.json` from prior runs (if exists). Extract learned patterns:

```json
{
  "patterns": [
    {
      "pattern": "Serializable annotation",
      "context": "DTOs",
      "example": "@Serializable\ndata class UserDto(@SerialName(\"user_id\") val userId: String)",
      "frequency": 4
    },
    {
      "pattern": "UiState sealed class",
      "context": "ViewModels",
      "example": "sealed class LoginUiState { object Loading : LoginUiState() }",
      "frequency": 5
    }
  ],
  "error_patterns": [
    {
      "error": "Android import in Domain",
      "solution": "Move to Data layer",
      "frequency": 2
    }
  ]
}
```

### Step 5: User Configuration

Prompt user for configuration decisions:

```
🔍 Detected Architecture: Clean Architecture
✓ Kotlin 1.9+
✓ Jetpack Compose
✓ Hilt DI
✓ Room (persistence)
✓ Retrofit (networking)

Configure Pipeline:

1. Confirm detected architecture (or override)?
   > [clean | mvvm | mvi | mvp | none]
   Default: clean

2. Enable/disable stages?
   [x] Test              (JUnit5 + MockK + Espresso + Compose Testing)
   [x] Lint              (ktlint + detekt)
   [x] Build             (./gradlew assemble)
   [x] Deploy            (Firebase App Distribution)
   [x] Git               (PR creation + push)

3. Deploy target (if deploy enabled)?
   > [debug-device | firebase | play-store]
   Default: firebase-app-distribution

4. Test framework?
   > [junit5 | kotest]
   Default: junit5 (with MockK, Espresso, Compose Testing)

5. Code generation style?
   > [clean-layers | feature-modules | hybrid]
   Default: clean-layers

6. Reference pipeline memory patterns?
   [x] Yes (loaded 3 patterns from prior runs)
   > Press Enter to confirm
```

**Save User Responses** → config.json

### Step 6: Output Configuration File

```json
{
  "project_name": "MyApp",
  "package_name": "com.example.myapp",
  "detected_architecture": "clean",
  "kotlin_version": "1.9.21",
  "compose_enabled": true,
  "hilt_enabled": true,
  "room_enabled": true,
  "retrofit_enabled": true,
  "serialization_enabled": true,
  "stages": {
    "scaffold": true,
    "test": true,
    "lint": true,
    "build": true,
    "deploy": true,
    "git": true
  },
  "deploy_target": "firebase-app-distribution",
  "test_framework": "junit5",
  "build_type": "debug",
  "code_gen_style": "clean-layers",
  "memory_patterns": 3,
  "created_at": "2025-02-11T10:30:00Z"
}
```

## Configuration Options Reference

### Architecture Selection
- **clean**: Domain (pure Kotlin) + Data (Retrofit/Room) + Presentation (Compose/ViewModel)
  - Enforces layer boundaries, zero Android imports in Domain
  - Best for: production apps, team scaling, maintainability
- **mvvm**: ViewModel + UI + Repository + Model (lightweight)
  - Fewer layers, faster iteration, less ceremony
  - Best for: medium-sized apps, simpler domain logic
- **mvi**: Intent + Reducer + State + Effect (event-driven)
  - Functional, predictable state management
  - Best for: complex UIs, cross-platform sync
- **mvp**: Presenter + View + Model (testable UI)
  - Separated UI logic, good for unit testing
  - Best for: UI-heavy apps
- **none**: Greenfield, auto-scaffold Clean Architecture
  - Best for: new projects

### Stage Toggles
- **scaffold**: Create folder structure, Hilt modules skeleton
  - Skip if existing project with structure already in place
- **test**: Generate + run JUnit5 + MockK + Espresso tests
  - Skip for quick prototypes
- **lint**: ktlint (format) + detekt (analysis)
  - Always recommended; catch bugs early
- **build**: `./gradlew assemble` (Debug or Release)
  - Always enabled for pipeline completion
- **deploy**: Push artifact to device/Firebase/Play Store
  - Skip for local testing
- **git**: Create PR, push branch
  - Skip for local-only work

### Deploy Targets
- **debug-device**: `./gradlew installDebug` (local emulator/device)
  - Instant, best for dev loop
- **firebase-app-distribution**: Firebase Hosting for QA testers
  - Requires Firebase project setup
- **play-store**: Google Play Console (Release builds only)
  - Requires signed keystore + Play Store account

### Test Framework
- **junit5**: JUnit 5 + MockK + Espresso + Compose Testing
  - Recommended, modern, flexible
- **kotest**: Kotest + MockK + Espresso + Compose Testing
  - Alternative, BDD-style specs

### Code Generation Style
- **clean-layers**: Generate Domain, Data, Presentation separately, Hilt bindings
  - Most structured, enforces layer separation
- **feature-modules**: One Gradle module per feature, internal Clean Architecture
  - Scales for large apps, enables parallel Gradle builds
- **hybrid**: Mix of clean layers + feature modules for specific domains
  - Flexible, requires more configuration

## Pipeline Memory Integration

After successful pipeline runs, Bootstrap loads learned patterns to accelerate future runs:

```json
{
  "serialization_pattern": "@Serializable + @SerialName for DTOs",
  "error_handling_pattern": "Result<T> sealed class or try-catch in ViewModel",
  "viewmodel_pattern": "@HiltViewModel + MutableStateFlow + viewModelScope.launch",
  "navigation_pattern": "sealed class Route, type-safe, NavHost in MainActivity",
  "naming_convention": {
    "dto": "UserDto, LoginDto (Data layer)",
    "entity": "User, AuthToken (Domain layer)",
    "viewmodel": "UserViewModel, LoginViewModel",
    "screen": "UserScreen, LoginScreen",
    "usecase": "GetUserUseCase, LoginUseCase"
  },
  "detekt_violations_fixed": [
    "Android import in Domain (moved to Data)",
    "Function complexity > 15 (extracted helper function)"
  ]
}
```

**Bootstrap Action:** Display learned patterns to coordinator/agents.

Example prompt:
```
📚 Pipeline Memory (from 3 prior runs):
• DTOs use @Serializable + @SerialName for JSON field mapping
• Exceptions: Result<T> sealed class in domain, try-catch in ViewModel
• Every ViewModel: @HiltViewModel, _uiState: MutableStateFlow, viewModelScope.launch
• ViewModels never call repositories directly; always use Use Cases
• Navigation: type-safe routes (sealed class), NavHost wraps all screens

These patterns will guide Code Gen Team for consistency.
```

## Troubleshooting Bootstrap

### Problem: Architecture Detection Fails
**Symptom:** "Could not detect architecture"
**Causes:**
- Non-standard folder structure
- Hybrid or custom architecture pattern

**Solution:**
- Bootstrap prompts: "Manually select architecture: clean | mvvm | mvi | mvp | none"
- User selects → Scaffold scaffolds detected architecture

### Problem: Kotlin Version Mismatch
**Symptom:** "Kotlin 1.8.x detected, but Compose requires 1.9+"
**Causes:**
- Outdated build.gradle.kts

**Solution:**
- Bootstrap suggests: Update build.gradle.kts `kotlin { jvmTarget = "11" }` and Kotlin gradle plugin to 1.9+
- User confirms → Bootstrap re-validates

### Problem: Missing Gradle Wrapper
**Symptom:** "gradle/wrapper/gradle-wrapper.jar not found"
**Causes:**
- Project initialized without wrapper

**Solution:**
- Bootstrap suggests: `gradle wrapper --gradle-version 8.0+`
- User runs → Bootstrap retries

## Next Steps After Bootstrap

✓ Bootstrap complete → **Coordinator** receives:
- Detected architecture + tech stack
- Stage configuration
- Pipeline memory patterns

Coordinator then routes the user's request to the appropriate agent (Product → Code Gen → Test ‖ Lint → Build → Deploy → Git).
