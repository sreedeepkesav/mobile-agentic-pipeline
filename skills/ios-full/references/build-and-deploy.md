# Build and Deploy Agents Reference

This document details the Build Agent and Deploy Agent operations for the ios-full-pipeline, including build configuration, deployment strategies, and troubleshooting.

## Build Agent Overview

The Build Agent compiles the iOS application and verifies code signing integrity. It runs after successful Test and Lint stages.

**Configuration Options:**
- `build_strategy`: always | never | per-task (default: always)
- `build_configuration`: Debug | Release (default: Debug)
- `code_signing_strategy`: automatic | manual (default: automatic)
- `verbose_logging`: true | false (default: false)

**Execution Steps:**
1. Verify project structure and configuration
2. Resolve dependencies (if using SPM)
3. Run xcodebuild compilation
4. Execute code signing sub-agent
5. Verify build artifacts
6. Report success or escalate failure

## Build Command Execution

**Standard Build Command (Debug):**

```bash
xcodebuild build \
  -scheme MyApp \
  -configuration Debug \
  -destination 'generic/platform=iOS' \
  -derivedDataPath .build
```

**Release Build Command:**

```bash
xcodebuild build \
  -scheme MyApp \
  -configuration Release \
  -destination 'generic/platform=iOS' \
  -derivedDataPath .build
```

**Build with Code Signing:**

```bash
xcodebuild build \
  -scheme MyApp \
  -configuration Release \
  -destination 'generic/platform=iOS' \
  -derivedDataPath .build \
  -signingIdentity 'Apple Distribution: Company Name' \
  -provisioning-profile 'UUID-of-profile'
```

## Code Signing Sub-Agent

The Code Signing Sub-Agent handles certificate and provisioning profile management.

**Responsibilities:**
- Verify developer certificates are installed
- Check provisioning profile validity
- Handle code signing identity selection
- Manage team ID configuration

**Automatic Code Signing:**
- Xcode manages signing automatically
- Requires valid Apple developer account
- Team ID must be configured in build settings

**Manual Code Signing:**
- Requires signing identity (certificate) installed
- Requires provisioning profile downloaded
- Must match Bundle ID exactly
- Certificate must be valid and unexpired

**Certificate Management Workflow:**

```
Build Start
│
├─ Check Development Certificates
│  ├─ Found? → Verify validity
│  └─ Not found? → Escalate to manual setup
│
├─ Check Provisioning Profiles
│  ├─ Valid? → Use for signing
│  └─ Expired? → Escalate
│
├─ Code Sign Executable
│  ├─ Success → Continue build
│  └─ Failure → Report error
│
└─ Verify Signature
   ├─ Valid → Build complete
   └─ Invalid → Fail build
```

## Build Configurations

**Debug Configuration:**
- Optimization: none (-Onone)
- Debug symbols: included
- App stripping: disabled
- Binary size: larger
- Build time: faster
- Use case: development, testing

**Release Configuration:**
- Optimization: full (-O)
- Debug symbols: separate file (dSYM)
- App stripping: enabled
- Binary size: smaller (~30% reduction)
- Build time: slower
- Use case: TestFlight, App Store distribution

**Configuration Selection:**
```swift
// Xcode Build Settings
#if DEBUG
    let apiBaseURL = "https://staging-api.example.com"
#else
    let apiBaseURL = "https://api.example.com"
#endif
```

## Build Failure Handling

**Compilation Errors:**

**Error Type: Type Mismatch**
```
error: cannot convert value of type 'String' to expected argument type 'Int'
  at UserViewModel.swift:42
```
Route to: Code Gen → Presentation Lead

**Error Type: Missing Import**
```
error: cannot find 'UserRepository' in scope
  at LoginViewController.swift:15
```
Route to: Code Gen → affected layer (Data/Domain)

**Error Type: Unresolved Symbol**
```
error: undefined symbols for architecture arm64:
  "_main", referenced from:
  start in libSystem.B.dylib
```
Route to: Build Agent → Architecture verification

**Code Signing Errors:**

**Error Type: No Provisioning Profile**
```
error: Provisioning profile "My App Distribution" doesn't include signing certificate.
```
Resolution:
1. Download updated provisioning profile from Apple Developer
2. Install in Xcode
3. Clean build folder (Cmd+Shift+K)
4. Rebuild

**Error Type: Certificate Not Found**
```
error: No signing identity found
```
Resolution:
1. Verify certificate installed in Keychain (Keychain Access.app)
2. Export certificate from Apple Developer
3. Install by double-clicking .cer file
4. Retry build

**Error Type: Team ID Mismatch**
```
error: The team ID is not set. Please set the development team to sign the app.
```
Resolution:
1. Open project in Xcode
2. Select target → Signing & Capabilities
3. Set Team ID under Team dropdown
4. Retry build

## Xcconfig Usage for Environment Settings

**File: Config/Debug.xcconfig**

```xcconfig
// Development environment
API_BASE_URL = https://staging-api.example.com
LOG_LEVEL = debug
ENABLE_ANALYTICS = NO
CERTIFICATE_PINNING = NO
APP_NAME = MyApp Dev
```

**File: Config/Release.xcconfig**

```xcconfig
// Production environment
API_BASE_URL = https://api.example.com
LOG_LEVEL = error
ENABLE_ANALYTICS = YES
CERTIFICATE_PINNING = YES
APP_NAME = MyApp
```

**Using Xcconfig Values in Code:**

```swift
// Config/BuildConfiguration.swift
struct BuildConfiguration {
    static let apiBaseURL = Bundle.main.infoDictionary?["API_BASE_URL"] as? String ?? "https://api.example.com"
    static let logLevel = Bundle.main.infoDictionary?["LOG_LEVEL"] as? String ?? "error"
    static let analyticsEnabled = Bundle.main.infoDictionary?["ENABLE_ANALYTICS"] as? Bool ?? false
}

// Usage
let apiClient = HTTPClient(baseURL: URL(string: BuildConfiguration.apiBaseURL)!)
```

## Build Artifacts

**Generated Artifacts:**

1. **App Bundle (.app)**
   - Location: `.build/Debug-iphoneos/MyApp.app`
   - Contains: compiled code, resources, configuration
   - Size: varies (typically 20-100 MB)

2. **dSYM (Debug Symbols)**
   - Location: `.build/Debug-iphoneos/MyApp.app.dSYM`
   - Purpose: symbolication for crash reports
   - Required: for crash log analysis in production

3. **Build Log**
   - Location: `.build/build.log`
   - Purpose: troubleshooting build issues
   - Retention: 30 days

**Artifact Verification:**

```bash
# Verify app bundle structure
unzip -l MyApp.app

# Check binary architecture
file MyApp.app/MyApp

# Verify code signature
codesign -v MyApp.app
```

## Common Build Failures and Solutions

**Issue: "Cyclic dependency detected"**
- Cause: Layer imports (Presentation imports Data directly)
- Solution: Import Domain protocols only, not Data implementations
- File: Check Presentation/Features files for Data imports

**Issue: "Module not found: Domain"**
- Cause: Domain target not linked
- Solution:
  1. Select target → Build Phases → Link Binary with Libraries
  2. Add Domain framework
  3. Rebuild

**Issue: "Ambiguous use of subscript"**
- Cause: Protocol conformance conflict
- Solution: Explicitly specify type with `as` cast
- Example: `array[index] as? SomeType`

**Issue: "Expected 'some' or '*' in type"**
- Cause: Swift syntax error (possibly old Swift version syntax)
- Solution:
  1. Check Swift version: `swift --version`
  2. Update to Swift 5.9+
  3. Fix syntax using opaque types (some) or Any

## Deploy Agent Overview

The Deploy Agent packages and distributes the iOS app to testers and users.

**Configuration Options:**
- `deploy_strategy`: always | never | per-task (default: never)
- `deployment_target`: TestFlight | AppStore (default: TestFlight)
- `automatic_submission`: true | false (default: false)
- `skip_screenshots`: true | false (default: false)

**Important:** Deploy is OFF by default. Must be explicitly enabled per task.

**Deployment Targets:**
- TestFlight: distribute to internal testers and beta users (2-10 hours approval)
- App Store: release to public (24-48 hours review)

## When to Disable Deploy Agent

The Deploy Agent should be disabled (deploy_strategy: never) in these scenarios:

**Personal Projects:**
```yaml
# .ios-pipeline.yml
deploy_strategy: never
reason: "Personal development, no distribution needed"
```

**Private Libraries/Frameworks:**
```yaml
deploy_strategy: never
reason: "Library, not user-facing app"
```

**Local-Only Builds:**
```yaml
deploy_strategy: never
reason: "CI/CD testing, no real users"
```

**Prototyping:**
```yaml
deploy_strategy: never
reason: "Rapid iteration, not ready for beta"
```

**Default Project Structure:**
```yaml
# .ios-pipeline.yml (typical)
test_strategy: always
lint_strategy: always
build_strategy: always
deploy_strategy: never  # ← Disabled by default
git_auto_merge: false
```

## Fastlane Integration

**File: fastlane/Fastfile**

```ruby
default_platform(:ios)

platform :ios do
  desc "Build and upload to TestFlight"
  lane :beta do
    increment_build_number(xcodeproj: "MyApp.xcodeproj")

    build_app(
      workspace: "MyApp.xcworkspace",
      scheme: "MyApp",
      configuration: "Release",
      derived_data_path: ".build",
      destination: "generic/platform=iOS",
      export_method: "app-store",
      skip_package_ipa: false,
      skip_package_pkg: true
    )

    upload_to_testflight(
      app_identifier: "com.example.myapp",
      apple_id: "appstore@example.com",
      skip_waiting_for_build_processing: false
    )
  end

  desc "Build and submit to App Store"
  lane :release do
    increment_version_number(
      version_number: "1.0.0"
    )
    increment_build_number(xcodeproj: "MyApp.xcodeproj")

    build_app(
      workspace: "MyApp.xcworkspace",
      scheme: "MyApp",
      configuration: "Release",
      derived_data_path: ".build",
      destination: "generic/platform=iOS",
      export_method: "app-store"
    )

    upload_to_app_store(
      app_identifier: "com.example.myapp",
      apple_id: "appstore@example.com",
      skip_waiting_for_build_processing: false
    )
  end
end
```

## TestFlight Deployment

**Deployment Process:**

1. **Archive Generation:**
   ```bash
   xcodebuild archive \
     -scheme MyApp \
     -configuration Release \
     -archivePath MyApp.xcarchive
   ```

2. **Export IPA:**
   ```bash
   xcodebuild -exportArchive \
     -archivePath MyApp.xcarchive \
     -exportOptionsPlist ExportOptions.plist \
     -exportPath .
   ```

3. **Upload to TestFlight:**
   ```bash
   xcrun altool --upload-app \
     --file MyApp.ipa \
     --type ios \
     --username appstore@example.com \
     --password "@keychain:fastlane_password"
   ```

4. **Distribute to Testers:**
   - Testers notified via email
   - 2-10 hours processing time
   - Can test on registered devices
   - Feedback collected in TestFlight app

**TestFlight Configuration:**

**File: fastlane/Matchfile**

```ruby
git_url("https://github.com/example/certificates")

storage_mode("git")
type("appstore")
app_identifier("com.example.myapp")
username("appstore@example.com")
team_id("ABC123")
```

## App Store Deployment

**Deployment Process:**

1. **Prepare Metadata:**
   - App name, description, keywords
   - Category, content rating
   - Release notes

2. **Screenshots (if not skipped):**
   - 5.5" display (1242 x 2208 px)
   - 6.5" display (1284 x 2778 px)
   - iPad (2048 x 2732 px)

3. **Build Selection:**
   - Select build from TestFlight
   - Version number must be unique
   - Build must have passed TestFlight review

4. **Submission:**
   - Complete app information
   - Accept agreements
   - Submit for review
   - 24-48 hours processing

5. **Release:**
   - After approval, select release date
   - "Release Immediately" for urgent updates
   - Scheduled release for coordinated launches

**App Store Configuration:**

**File: fastlane/Appfile**

```ruby
app_identifier("com.example.myapp")
apple_id("appstore@example.com")
team_id("ABC123")
team_name("Example Corp")
itc_team_id("987654321")
```

## Deploy Failure Handling

**Failure Type: Build Upload Failed**
```
Error: Authentication token expired
```
Route to: Build Agent → Re-archive and retry

**Failure Type: Invalid Build**
```
Error: Build contains bitcode but app doesn't declare support
```
Route to: Code Gen → Build Settings review

**Failure Type: Metadata Validation**
```
Error: Screenshots must be 1242x2208 pixels
```
Route to: Product team → Screenshot update needed

**Failure Type: Entitlements Mismatch**
```
Error: The entitlements in your app don't match the provisioning profile
```
Route to: Build Agent → Code signing review

## Deployment Status Reporting

**Success Report:**
```
✓ Build completed successfully
✓ Code signing verified
✓ TestFlight upload complete
✓ Beta testers notified
✓ Status: Waiting for processing (ETA: 4-6 hours)
```

**Failure Report:**
```
✗ TestFlight upload failed
  Reason: Insufficient free space on server
  Action: Retry automatically in 5 minutes
  Contact: DevOps team for manual intervention
```

## Deployment Validation Checklist

Before enabling deploy_strategy: always, verify:

- [ ] Project has valid Bundle ID
- [ ] Developer certificates installed and valid
- [ ] Provisioning profiles current (not expiring soon)
- [ ] Version number incremented
- [ ] Build number incremented
- [ ] Fastlane configured with correct team IDs
- [ ] TestFlight internal testing group created
- [ ] App Store app entry created
- [ ] Privacy policy URL configured
- [ ] Support URL provided
- [ ] Category and content rating selected

## Environment-Specific Deployment

**Staging Deployment:**
```yaml
deploy_strategy: per-task
deployment_target: TestFlight
automatic_submission: false
# Manual review before TestFlight release
```

**Production Deployment:**
```yaml
deploy_strategy: per-task
deployment_target: AppStore
automatic_submission: false
# Requires explicit approval and metadata review
```

## Build and Deploy Timeline

**Typical Build Timeline:**
- Clean build: 2-5 minutes (first time, depends on dependencies)
- Incremental build: 30-60 seconds (for code changes)
- Archive for release: 3-7 minutes (additional optimization)

**Typical Deployment Timeline:**
- Build → TestFlight: 15 minutes
- TestFlight processing: 2-10 hours
- TestFlight → App Store: 5 minutes
- App Store review: 24-48 hours
- Release to users: immediate (after approval)
