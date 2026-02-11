# Build and Deploy

## Overview

After testing and linting pass, the **Build Agent** compiles the application into an APK, and the **Deploy Agent** distributes it to the target environment (local device, Firebase App Distribution, or Google Play Store).

---

## Build Agent

### Build Process

**Goal:** Compile Kotlin code, link resources, generate APK (Debug or Release).

**Command (Debug Build):**
```bash
./gradlew assembleDebug
```

**Output:**
```
app/build/outputs/apk/debug/app-debug.apk
```

**Command (Release Build with Signing):**
```bash
./gradlew assembleRelease \
  -Pandroid.injected.signing.store.file=$KEYSTORE_PATH \
  -Pandroid.injected.signing.store.password=$KEYSTORE_PASSWORD \
  -Pandroid.injected.signing.key.alias=$KEY_ALIAS \
  -Pandroid.injected.signing.key.password=$KEY_PASSWORD
```

**Output:**
```
app/build/outputs/apk/release/app-release.apk
```

### Signing Configuration

In `build.gradle.kts`:

```kotlin
android {
    signingConfigs {
        create("release") {
            storeFile = file(System.getenv("KEYSTORE_PATH") ?: "keystore.jks")
            storePassword = System.getenv("KEYSTORE_PASSWORD")
            keyAlias = System.getenv("KEY_ALIAS") ?: "release"
            keyPassword = System.getenv("KEY_PASSWORD")
        }
    }

    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
            minifyEnabled = true
            shrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }

        debug {
            debuggable = true
            minifyEnabled = false
        }
    }
}
```

**Environment Variables:**
- `KEYSTORE_PATH`: Path to .jks keystore file (e.g., `/home/user/keys/release.jks`)
- `KEYSTORE_PASSWORD`: Keystore password
- `KEY_ALIAS`: Key alias (e.g., `release`)
- `KEY_PASSWORD`: Key password

**Note:** For CI/CD, set these environment variables securely (GitHub Secrets, GitLab Variables, etc.)

### Build Phases

1. **Compilation Phase**
   ```bash
   ./gradlew compileDebugKotlin
   ```
   - Compile Kotlin source → intermediate bytecode
   - Check for compilation errors
   - Hilt annotation processing (generates DI code)

2. **Resource Linking Phase**
   ```bash
   ./gradlew mergeDebugResources
   ```
   - Merge Android resources (strings, layouts, drawables)
   - Process Material Design 3 themes
   - Generate R.java

3. **Packaging Phase**
   ```bash
   ./gradlew packageDebug
   ```
   - Package bytecode + resources into APK
   - Dex compilation (bytecode → DEX format for Android)

4. **Signing Phase (Release only)**
   ```bash
   jarsigner -keystore $KEYSTORE app-release-unsigned.apk $KEY_ALIAS
   zipalign -v 4 app-release-unsigned.apk app-release.apk
   ```
   - Sign APK with private key
   - Align resources for optimization

### Build Configuration

**Development Build (Debug):**
- No obfuscation (readable stack traces for debugging)
- Debuggable (can attach debugger)
- Unsigned or self-signed certificate
- Faster build time

**Production Build (Release):**
- ProGuard/R8 obfuscation (smaller APK, security)
- Resource shrinking (remove unused resources)
- Signed with release keystore
- Slower build time (more optimization)

### Build Output & Artifacts

```
app/build/outputs/
├── apk/
│   ├── debug/
│   │   ├── app-debug.apk              # Debug APK (install with adb)
│   │   └── app-debug-mapping.txt      # ProGuard mapping (debug symbols)
│   └── release/
│       ├── app-release.apk            # Release APK (signed, optimized)
│       ├── app-release-mapping.txt    # ProGuard mapping (prod symbols)
│       └── app-release-changelog.txt  # Build changelog
├── bundle/
│   └── release/
│       └── app-release.aab            # Android App Bundle (for Play Store)
└── logs/
    └── build-report.txt               # Build log with timings
```

### Build Troubleshooting

#### Compile Error

```
Error: Cannot resolve symbol 'LoginViewModel'
Location: LoginScreen.kt:5

→ Route to: Pres Lead
```

**Common causes:**
- ViewModel not annotated with @HiltViewModel
- ViewModel not injected via hiltViewModel()
- Import missing

#### Hilt Compilation Error

```
Error: [dagger.hilt.processor] Duplicate bindings for AuthRepository
Location: AuthModule.kt, UserModule.kt

→ Route to: Integration
```

**Common causes:**
- Two @Binds for same interface
- @Provides and @Binds for same type
- Module installed twice

**Fix:**
```kotlin
// AuthModule.kt
@Module
@InstallIn(SingletonComponent::class)
interface AuthModule {
    @Binds
    @Singleton
    fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository
}

// Remove duplicate from UserModule.kt
```

#### Gradle Sync Error

```
Error: Failed to resolve dependency: com.google.dagger:hilt-android:2.46

→ Route to: Build Agent (check Gradle dependencies)
```

**Common causes:**
- Typo in dependency version
- Hilt version incompatible with Kotlin version

**Fix:**
```kotlin
// build.gradle.kts
dependencies {
    implementation("com.google.dagger:hilt-android:2.48")  // Correct version
    kapt("com.google.dagger:hilt-compiler:2.48")            // Match versions
}
```

#### Signing Error

```
Error: Invalid keystore format. Keystore password is incorrect or keystore file not found.

→ Route to: Build Agent (check KEYSTORE_PATH, KEYSTORE_PASSWORD)
```

**Fix:**
```bash
# Verify keystore exists
ls -la $KEYSTORE_PATH

# Verify password (test extraction)
keytool -list -v -keystore $KEYSTORE_PATH -storepass $KEYSTORE_PASSWORD
```

### Build Report

After successful build:

```markdown
# Build Report

## Build Configuration
- Build Type: debug
- App ID: com.example.myapp
- Version: 1.0.0 (versionCode: 1)
- Min SDK: 24, Target SDK: 34

## Compilation Results
✓ Kotlin compilation: 45s
✓ Resource linking: 8s
✓ DEX generation: 12s
✓ Packaging: 3s

## Artifact Information
- APK Name: app-debug.apk
- APK Size: 45.2 MB
- Min Size (with minification): ~32 MB (Release)
- Supported ABIs: [arm64-v8a, armeabi-v7a, x86, x86_64]

## Dependencies Included
- Kotlin stdlib: 1.9.21
- Jetpack Compose: 1.5.4
- Hilt: 2.48
- Room: 2.5.2
- Retrofit: 2.10.0
- (+ 50 transitive dependencies)

## Build Success
✓ Compilation: SUCCESS
✓ Tests: PASSED (73% coverage)
✓ Lint: CLEAN (ktlint + detekt)
✓ APK generated: app/build/outputs/apk/debug/app-debug.apk

## Next: Deploy Agent will distribute APK
```

---

## Deploy Agent

### Deployment Targets

The Deploy Agent supports multiple deployment targets (configured during Bootstrap):

#### 1. Debug Device/Emulator (Local)

**Command:**
```bash
./gradlew installDebug
```

**Process:**
1. Build APK: `./gradlew assembleDebug`
2. Find connected device/emulator: `adb devices`
3. Install: `adb install app/build/outputs/apk/debug/app-debug.apk`
4. Launch: `adb shell am start -n com.example.myapp/.MainActivity`

**Best for:** Local development, quick testing, immediate feedback.

**Deployment Time:** 30–60 seconds (depends on device)

---

#### 2. Firebase App Distribution

**Purpose:** Distribute debug/internal builds to QA team, beta testers.

**Setup:** Firebase project with App Distribution enabled.

**Command:**
```bash
./gradlew appDistributionUploadDebug \
  --artifact-key="build/outputs/apk/debug/app-debug.apk" \
  --release-notes="Bug fixes and performance improvements"
```

**Alternative (Fastlane):**
```bash
fastlane run firebase_app_distribution \
  app: "app/build/outputs/apk/debug/app-debug.apk" \
  firebase_cli_token: "$FIREBASE_TOKEN" \
  release_notes: "New feature: user authentication"
```

**Process:**
1. Build APK
2. Upload to Firebase console
3. Firebase generates download link
4. Share link with testers (via email, Slack, etc.)
5. Testers click link, install via Firebase App Tester app

**Best for:** QA testing, internal beta, stakeholder demos.

**Deployment Time:** 2–5 minutes (including upload + email).

---

#### 3. Google Play Store (Production)

**Purpose:** Release to public app store.

**Prerequisites:**
- Google Play Developer Account ($25 one-time)
- Release keystore (never use debug key)
- App Bundle (.aab) for optimal delivery

**Command (Fastlane):**
```bash
fastlane run supply \
  apk_paths: ["app/build/outputs/apk/release/app-release.apk"] \
  release_notes: "v1.0.0: Initial release" \
  track: "internal"  # internal → beta → production
```

**Manual Process (if no Fastlane):**
1. Build Release APK: `./gradlew assembleRelease`
2. Generate App Bundle: `./gradlew bundleRelease`
3. Upload to Google Play Console manually
4. Configure release notes, screenshots, permissions
5. Submit for review (24–48 hours)
6. Staged rollout: 25% → 50% → 100%

**Best for:** Production release, public availability.

**Deployment Time:** Build: 3–5 min, Upload: 1 min, Review: 24–48 hours, Rollout: configurable.

---

### Deployment Workflow

```
Build Agent completes
  ↓
APK ready: app-debug.apk (or app-release.apk)
  ↓
Deploy Agent reads config:
  deploy_target: "firebase-app-distribution"
  ↓
Deploy Agent:
1. Verify APK exists + valid signature
2. Identify deployment method (Firebase, Play Store, local)
3. Execute deployment command (fastlane, adb, etc.)
4. Wait for completion
5. Generate deployment report
  ↓
Deployment success
  ↓
Git Agent: Create PR + push
```

### Deployment Validation

Before deploying, Deploy Agent validates:

1. **APK Integrity**
   ```bash
   # Verify APK is valid ZIP
   unzip -t app-debug.apk > /dev/null
   echo $?  # 0 = valid, 1+ = corrupted
   ```

2. **APK Signature (Release only)**
   ```bash
   # Verify signing certificate
   jarsigner -verify -verbose -certs app-release.apk
   ```

3. **APK Size Check**
   ```bash
   # Warn if larger than expected
   SIZE=$(stat -f%z app-debug.apk)
   if [ $SIZE -gt 100000000 ]; then
       echo "WARNING: APK > 100MB. Check for unnecessary resources or libraries."
   fi
   ```

4. **Target Compatibility**
   ```bash
   # Verify min/target SDK matches deployment target
   aapt dump badging app-debug.apk | grep "minSdkVersion\|targetSdkVersion"
   ```

### Deployment Failure Scenarios

#### Scenario 1: APK Upload to Firebase Fails

```
Deploy Agent attempts Firebase upload:
  ./gradlew appDistributionUploadDebug
  ↓
Error: "403 Forbidden: Invalid Firebase token"
  ↓
Route to: Build Agent (check FIREBASE_TOKEN env var)

Build Agent:
1. Verify token is set: echo $FIREBASE_TOKEN
2. Re-authenticate if expired: firebase login
3. Retry: ./gradlew appDistributionUploadDebug
4. Success
```

#### Scenario 2: Google Play Upload Rejected

```
Deploy Agent attempts Play Store upload:
  fastlane supply --apk app-release.apk --track internal
  ↓
Error: "This app version (versionCode: 1) was already uploaded"
  ↓
Route to: Build Agent

Build Agent:
1. Increment versionCode in build.gradle.kts
2. Rebuild: ./gradlew assembleRelease
3. Retry: fastlane supply --apk app-release.apk --track internal
4. Success
```

#### Scenario 3: Device/Emulator Not Found (Local)

```
Deploy Agent attempts local install:
  adb install app/build/outputs/apk/debug/app-debug.apk
  ↓
Error: "error: no devices/emulators found"
  ↓
Route to: User (start emulator or connect device)

User:
1. Start Android emulator: `emulator -avd MyDevice`
   OR
2. Connect physical device via USB
3. Re-run: ./gradlew installDebug
4. App installed and launches
```

### Deployment Report

After successful deployment:

```markdown
# Deployment Report

## Deployment Configuration
- Target: Firebase App Distribution
- APK: app-debug.apk (45.2 MB)
- Version: 1.0.0 (versionCode: 1)
- Build Type: debug

## Pre-Deployment Checks
✓ APK integrity: Valid
✓ APK signature: Valid (debug key)
✓ Min SDK: 24 (compatible)
✓ Target SDK: 34

## Deployment Execution
✓ Upload started: 14:32:15 UTC
✓ Upload completed: 14:34:22 UTC (127 seconds)
✓ Firebase processing: 30 seconds
✓ Testers notified: 14:35:00 UTC

## Deployment Result
✓ Status: SUCCESS
✓ Download URL: https://firebase.google.com/...
✓ Tester emails notified: qa-team@example.com
✓ Expiration: 90 days from upload

## Next: Git Agent will create PR + push
```

---

## Configuration Examples

### Local Development (Debug)

**Pipeline Config:**
```json
{
  "build_type": "debug",
  "deploy_target": "debug-device"
}
```

**Build Command:**
```bash
./gradlew assembleDebug
```

**Deploy Command:**
```bash
./gradlew installDebug
adb shell am start -n com.example.myapp/.MainActivity
```

---

### QA Testing (Firebase App Distribution)

**Pipeline Config:**
```json
{
  "build_type": "debug",
  "deploy_target": "firebase-app-distribution",
  "firebase_token": "$FIREBASE_TOKEN"
}
```

**Build Command:**
```bash
./gradlew assembleDebug
```

**Deploy Command:**
```bash
fastlane run firebase_app_distribution \
  app: "app/build/outputs/apk/debug/app-debug.apk" \
  firebase_cli_token: "$FIREBASE_TOKEN" \
  groups: "qa-team" \
  release_notes: "Testing authentication feature"
```

---

### Production Release (Play Store)

**Pipeline Config:**
```json
{
  "build_type": "release",
  "deploy_target": "play-store",
  "keystore_path": "$KEYSTORE_PATH",
  "keystore_password": "$KEYSTORE_PASSWORD",
  "key_alias": "release",
  "key_password": "$KEY_PASSWORD"
}
```

**Build Command:**
```bash
./gradlew assembleRelease \
  -Pandroid.injected.signing.store.file=$KEYSTORE_PATH \
  -Pandroid.injected.signing.store.password=$KEYSTORE_PASSWORD \
  -Pandroid.injected.signing.key.alias=$KEY_ALIAS \
  -Pandroid.injected.signing.key.password=$KEY_PASSWORD
```

**Deploy Command (Staged Rollout):**
```bash
fastlane supply \
  apk_paths: ["app/build/outputs/apk/release/app-release.apk"] \
  track: "internal" \
  rollout: 0.25  # Start with 25% rollout

# After monitoring for errors, continue:
fastlane supply \
  apk_paths: ["app/build/outputs/apk/release/app-release.apk"] \
  track: "production" \
  rollout: 1.0  # Full rollout to 100%
```

---

## Best Practices

1. **Always Build Release with Signing:** Never deploy production without proper signing.
2. **Version Management:** Increment versionCode for every Play Store upload.
3. **Staged Rollouts:** Deploy to 25% → 50% → 100% to catch issues early.
4. **Keep Keystore Safe:** Store .jks file securely (not in Git, use CI/CD secrets).
5. **Monitor Crashes:** Use Firebase Crashlytics to monitor deployed builds.
6. **Release Notes:** Always provide clear, user-facing release notes.
7. **Testing Before Deploy:** Verify on local device/emulator before Firebase/Play Store.

---

## Integration with Git

After Build + Deploy complete successfully:

```
Deploy Agent reports:
✓ Build: SUCCESS (app-release.apk)
✓ Deploy: SUCCESS (Firebase App Distribution)

↓
Git Agent triggered:
1. Commit changes with version tag
2. Push to remote
3. Create/update PR
4. Mark for human review/merge
```

See [git-workflow.md](./git-workflow.md) for details.
