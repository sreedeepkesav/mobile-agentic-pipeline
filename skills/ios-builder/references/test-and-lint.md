# Test and Lint Agents Reference

This document details the Test Agent and Lint Agent operations for the ios-builder, including execution strategies, configuration options, and quality gate enforcement.

## Test Agent Overview

The Test Agent is responsible for generating and executing comprehensive test suites across all architecture layers.

**Configuration Options:**
- `test_strategy`: always | never | per-task (default: per-task)
- `coverage_target`: percentage (default: 80%)
- `test_timeout`: seconds (default: 300)

**Parallel Sub-Agents:**
- Unit Test Generator: creates protocol-based mocks, domain/data/presentation layer tests
- UI Test Generator: creates XCUITest flows for navigation and critical user paths

**Execution:**
- Both generators run in parallel
- Results aggregated before Build stage
- Failures route through Coordinator to Code Gen lead
- Success status gates Build stage entry

## Layer-Aware Testing Strategy

### Domain Layer Testing

**Test Scope:**
- Test use cases directly via their protocols
- No mocks needed (repositories are injected)
- Mock repositories return hardcoded test data
- Focus: business logic correctness, error handling

**Example: FetchUserProfileUseCaseTests**

```swift
import XCTest
@testable import Domain

final class FetchUserProfileUseCaseTests: XCTestCase {
    private var sut: FetchUserProfileUseCase!
    private var mockRepository: MockUserRepository!

    override func setUp() {
        super.setUp()
        mockRepository = MockUserRepository()
        sut = FetchUserProfileUseCaseImpl(repository: mockRepository)
    }

    override func tearDown() {
        sut = nil
        mockRepository = nil
        super.tearDown()
    }

    // MARK: - Success Cases

    func test_execute_withValidUserId_returnsUser() async {
        // Arrange
        let expectedUser = User(
            id: "user-123",
            name: "John Doe",
            email: "john@example.com",
            profileImageURL: nil,
            joinDate: Date(),
            isActive: true
        )
        mockRepository.fetchUserProfileResult = .success(expectedUser)

        // Act
        let result = try await sut.execute(userId: "user-123")

        // Assert
        XCTAssertEqual(result, expectedUser)
    }

    func test_execute_withMultipleUsers_returnsCorrectUser() async {
        // Arrange
        let users = [
            User(id: "1", name: "Alice", email: "alice@example.com", profileImageURL: nil, joinDate: Date(), isActive: true),
            User(id: "2", name: "Bob", email: "bob@example.com", profileImageURL: nil, joinDate: Date(), isActive: false)
        ]
        mockRepository.fetchUserProfileResult = .success(users[0])

        // Act
        let result = try await sut.execute(userId: "1")

        // Assert
        XCTAssertEqual(result.name, "Alice")
    }

    // MARK: - Error Cases

    func test_execute_withEmptyUserId_throwsValidationError() async {
        // Act & Assert
        do {
            _ = try await sut.execute(userId: "")
            XCTFail("Expected ValidationError")
        } catch {
            XCTAssertEqual(error as? ValidationError, .emptyUserId)
        }
    }

    func test_execute_withRepositoryFailure_throwsError() async {
        // Arrange
        mockRepository.fetchUserProfileResult = .failure(NSError(domain: "Network", code: -1))

        // Act & Assert
        do {
            _ = try await sut.execute(userId: "user-123")
            XCTFail("Expected error")
        } catch {
            XCTAssertNotNil(error)
        }
    }

    // MARK: - Integration

    func test_execute_callsRepositoryWithCorrectUserId() async {
        // Arrange
        let expectedUserId = "user-123"
        mockRepository.fetchUserProfileResult = .success(mockUser())

        // Act
        _ = try await sut.execute(userId: expectedUserId)

        // Assert
        XCTAssertEqual(mockRepository.fetchUserProfileCalledWithId, expectedUserId)
    }

    private func mockUser() -> User {
        User(
            id: "user-123",
            name: "Test User",
            email: "test@example.com",
            profileImageURL: nil,
            joinDate: Date(),
            isActive: true
        )
    }
}

// MARK: - Mock Repository

final class MockUserRepository: UserRepository {
    var fetchUserProfileResult: Result<User, Error> = .failure(NSError(domain: "", code: -1))
    var fetchUserProfileCalledWithId: String?

    func fetchUserProfile(userId: String) async throws -> User {
        fetchUserProfileCalledWithId = userId
        switch fetchUserProfileResult {
        case .success(let user):
            return user
        case .failure(let error):
            throw error
        }
    }

    func updateUserProfile(_ user: User) async throws {
        // Unused in this test
    }

    func deleteAccount(userId: String) async throws {
        // Unused in this test
    }
}
```

**Domain Test Coverage Target:** 90%+

### Data Layer Testing

**Test Scope:**
- Mock APIClient/network calls
- Test DTO parsing and decoding
- Test mapper DTO→Domain transformations
- Test repository implementations
- Focus: data flow, API contract adherence, error mapping

**Example: UserRepositoryImplTests**

```swift
import XCTest
@testable import Data
@testable import Domain

final class UserRepositoryImplTests: XCTestCase {
    private var sut: UserRepositoryImpl!
    private var mockAPIClient: MockAPIClient!
    private var mapper: UserMapper!

    override func setUp() {
        super.setUp()
        mockAPIClient = MockAPIClient()
        mapper = UserMapper()
        sut = UserRepositoryImpl(
            apiClient: mockAPIClient,
            mapper: mapper
        )
    }

    override func tearDown() {
        sut = nil
        mockAPIClient = nil
        mapper = nil
        super.tearDown()
    }

    // MARK: - Fetch User Tests

    func test_fetchUserProfile_withValidResponse_returnsMappedUser() async {
        // Arrange
        let dto = UserDTO(
            id: "user-123",
            full_name: "John Doe",
            email_address: "john@example.com",
            profile_image_url: "https://example.com/image.jpg",
            join_date: "2024-01-15T10:30:00Z",
            is_active: true
        )
        mockAPIClient.requestResult = .success(dto)

        // Act
        let result = try await sut.fetchUserProfile(userId: "user-123")

        // Assert
        XCTAssertEqual(result.id, "user-123")
        XCTAssertEqual(result.name, "John Doe")
        XCTAssertEqual(result.email, "john@example.com")
        XCTAssertTrue(result.isActive)
    }

    func test_fetchUserProfile_withNullImageURL_returnsNilImageURL() async {
        // Arrange
        let dto = UserDTO(
            id: "user-123",
            full_name: "John Doe",
            email_address: "john@example.com",
            profile_image_url: nil,
            join_date: "2024-01-15T10:30:00Z",
            is_active: true
        )
        mockAPIClient.requestResult = .success(dto)

        // Act
        let result = try await sut.fetchUserProfile(userId: "user-123")

        // Assert
        XCTAssertNil(result.profileImageURL)
    }

    func test_fetchUserProfile_withNetworkError_throwsError() async {
        // Arrange
        let networkError = NSError(domain: "Network", code: -1, userInfo: [NSLocalizedDescriptionKey: "No internet"])
        mockAPIClient.requestResult = .failure(networkError)

        // Act & Assert
        do {
            _ = try await sut.fetchUserProfile(userId: "user-123")
            XCTFail("Expected error")
        } catch {
            XCTAssertNotNil(error)
        }
    }

    // MARK: - Update User Tests

    func test_updateUserProfile_withValidUser_callsAPIClient() async {
        // Arrange
        let user = User(
            id: "user-123",
            name: "Jane Doe",
            email: "jane@example.com",
            profileImageURL: nil,
            joinDate: Date(),
            isActive: true
        )
        mockAPIClient.requestResult = .success(EmptyResponse())

        // Act
        try await sut.updateUserProfile(user)

        // Assert
        XCTAssertTrue(mockAPIClient.requestCalled)
    }

    func test_updateUserProfile_withAPIError_throwsError() async {
        // Arrange
        let user = User(
            id: "user-123",
            name: "Jane Doe",
            email: "jane@example.com",
            profileImageURL: nil,
            joinDate: Date(),
            isActive: true
        )
        let apiError = NSError(domain: "API", code: 422, userInfo: [NSLocalizedDescriptionKey: "Validation failed"])
        mockAPIClient.requestResult = .failure(apiError)

        // Act & Assert
        do {
            try await sut.updateUserProfile(user)
            XCTFail("Expected error")
        } catch {
            XCTAssertNotNil(error)
        }
    }
}

// MARK: - Mock API Client

final class MockAPIClient: APIClient {
    var requestResult: Result<Any, Error> = .failure(NSError(domain: "", code: -1))
    var requestCalled = false

    func request<T: Decodable>(_ endpoint: APIEndpoint, as: T.Type) async throws -> T {
        requestCalled = true
        switch requestResult {
        case .success(let response):
            return response as! T
        case .failure(let error):
            throw error
        }
    }
}
```

**Data Test Coverage Target:** 85%+

### Presentation Layer Testing

**Test Scope:**
- Mock all use cases (injected into ViewModels)
- Test ViewModel state transitions
- Test @Published property updates
- Test async loading and error handling
- Focus: state management, user interaction handling

**Example: LoginViewModelTests**

```swift
import XCTest
@testable import Presentation
@testable import Domain

final class LoginViewModelTests: XCTestCase {
    private var sut: LoginViewModel!
    private var mockLoginUseCase: MockLoginUseCase!

    override func setUp() {
        super.setUp()
        mockLoginUseCase = MockLoginUseCase()
        sut = LoginViewModel(loginUseCase: mockLoginUseCase)
    }

    override func tearDown() {
        sut = nil
        mockLoginUseCase = nil
        super.tearDown()
    }

    // MARK: - Initial State Tests

    func test_init_setsIdleState() {
        // Assert
        if case .idle = sut.state {
            XCTAssertTrue(true)
        } else {
            XCTFail("Expected idle state")
        }
    }

    // MARK: - Login Success Tests

    func test_login_withValidCredentials_transitionsToSuccessState() async {
        // Arrange
        let expectedToken = AuthToken(token: "abc123", expiresIn: 3600)
        mockLoginUseCase.loginResult = .success(expectedToken)
        sut.email = "test@example.com"
        sut.password = "password123"

        // Act
        sut.login()
        try await Task.sleep(nanoseconds: 100_000_000) // Wait for async

        // Assert
        if case .success(let token) = sut.state {
            XCTAssertEqual(token.token, "abc123")
        } else {
            XCTFail("Expected success state")
        }
    }

    // MARK: - Login Failure Tests

    func test_login_withInvalidCredentials_transitionsToErrorState() async {
        // Arrange
        mockLoginUseCase.loginResult = .failure(AuthError.invalidCredentials)
        sut.email = "test@example.com"
        sut.password = "wrongpassword"

        // Act
        sut.login()
        try await Task.sleep(nanoseconds: 100_000_000) // Wait for async

        // Assert
        if case .error(let message) = sut.state {
            XCTAssertNotNil(message)
        } else {
            XCTFail("Expected error state")
        }
    }

    func test_login_duringLoading_setsLoadingState() {
        // Arrange
        mockLoginUseCase.loginResult = .success(AuthToken(token: "token", expiresIn: 3600))
        sut.email = "test@example.com"
        sut.password = "password123"

        // Act
        sut.login()

        // Assert (immediate check before async completes)
        if case .loading = sut.state {
            XCTAssertTrue(true)
        } else {
            XCTFail("Expected loading state")
        }
    }

    // MARK: - Reset State Tests

    func test_resetState_transitionsToIdle() async {
        // Arrange
        mockLoginUseCase.loginResult = .success(AuthToken(token: "token", expiresIn: 3600))
        sut.email = "test@example.com"
        sut.password = "password123"
        sut.login()
        try await Task.sleep(nanoseconds: 100_000_000)

        // Act
        sut.resetState()

        // Assert
        if case .idle = sut.state {
            XCTAssertTrue(true)
        } else {
            XCTFail("Expected idle state")
        }
    }

    // MARK: - Use Case Invocation Tests

    func test_login_callsLoginUseCaseWithCredentials() async {
        // Arrange
        let email = "test@example.com"
        let password = "password123"
        mockLoginUseCase.loginResult = .success(AuthToken(token: "token", expiresIn: 3600))
        sut.email = email
        sut.password = password

        // Act
        sut.login()
        try await Task.sleep(nanoseconds: 100_000_000)

        // Assert
        XCTAssertEqual(mockLoginUseCase.loginCalledWithEmail, email)
        XCTAssertEqual(mockLoginUseCase.loginCalledWithPassword, password)
    }
}

// MARK: - Mock Use Case

final class MockLoginUseCase: LoginUseCase {
    var loginResult: Result<AuthToken, Error> = .failure(NSError(domain: "", code: -1))
    var loginCalledWithEmail: String?
    var loginCalledWithPassword: String?

    func execute(email: String, password: String) async throws -> AuthToken {
        loginCalledWithEmail = email
        loginCalledWithPassword = password
        switch loginResult {
        case .success(let token):
            return token
        case .failure(let error):
            throw error
        }
    }
}
```

**Presentation Test Coverage Target:** 80%+

### UI Layer Testing

**Test Scope:**
- XCUITest for coordinator flows
- Test screen navigation paths
- Test user interaction sequences
- Focus: critical user journeys (authentication, main flows)

**Example: AuthFlowUITests**

```swift
import XCTest

final class AuthFlowUITests: XCTestCase {
    let app = XCUIApplication()

    override func setUp() {
        super.setUp()
        continueAfterFailure = false
        app.launch()
    }

    override func tearDown() {
        super.tearDown()
    }

    // MARK: - Login Flow Tests

    func test_loginFlow_withValidCredentials_navigatesToMainApp() {
        // Arrange
        let emailTextField = app.textFields["emailField"]
        let passwordTextField = app.secureTextFields["passwordField"]
        let loginButton = app.buttons["loginButton"]

        // Act
        emailTextField.tap()
        emailTextField.typeText("test@example.com")

        passwordTextField.tap()
        passwordTextField.typeText("password123")

        loginButton.tap()

        // Assert
        let homeScreen = app.navigationBars["Home"]
        XCTAssertTrue(homeScreen.waitForExistence(timeout: 5))
    }

    func test_loginFlow_withInvalidCredentials_showsError() {
        // Arrange
        let emailTextField = app.textFields["emailField"]
        let passwordTextField = app.secureTextFields["passwordField"]
        let loginButton = app.buttons["loginButton"]

        // Act
        emailTextField.tap()
        emailTextField.typeText("invalid@example.com")

        passwordTextField.tap()
        passwordTextField.typeText("wrongpass")

        loginButton.tap()

        // Assert
        let errorAlert = app.alerts.firstMatch
        XCTAssertTrue(errorAlert.waitForExistence(timeout: 5))
    }

    // MARK: - Navigation Tests

    func test_forgotPasswordLink_navigatesToForgotPasswordScreen() {
        // Act
        app.buttons["forgotPasswordButton"].tap()

        // Assert
        let forgotPasswordTitle = app.navigationBars["Forgot Password"]
        XCTAssertTrue(forgotPasswordTitle.waitForExistence(timeout: 5))
    }

    func test_signupLink_navigatesToSignupScreen() {
        // Act
        app.buttons["signupButton"].tap()

        // Assert
        let signupTitle = app.navigationBars["Sign Up"]
        XCTAssertTrue(signupTitle.waitForExistence(timeout: 5))
    }
}
```

## Test Naming Convention

**Pattern:** `test_[methodName]_[givenCondition]_[expectedResult]`

**Examples:**
```
test_fetchUser_withValidId_returnsUserData
test_login_withInvalidCredentials_showsErrorState
test_viewModel_onViewAppear_loadsUserProfile
test_repository_withNetworkError_throwsError
test_mapper_withNullField_returnsNilProperty
```

## Lint Agent Overview

The Lint Agent enforces code quality and architectural consistency through automated checking and fixing.

**Configuration Options:**
- `lint_strategy`: always | never | per-task (default: always)
- `auto_fix_enabled`: true | false (default: true)
- `escalate_on_error`: true | false (default: true)

**Execution Process:**
1. SwiftLint check → identify violations
2. SwiftFormat auto-fix → fix style issues
3. Re-lint → verify fixes
4. Layer boundary check → scan imports
5. Pass or escalate → report results

**Self-Healing:**
- Auto-fixable violations don't escalate
- Semantic violations escalate to Code Gen lead
- Layer boundary violations escalate to Architect

## SwiftLint Rules

**Key Rules Enforced:**

```yaml
# Error-level rules
force_unwrap:
  severity: error
  excluded_operators: ["??"]

trailing_whitespace:
  severity: error

# Warning-level rules
implicit_return:
  severity: warning

line_length:
  warning: 120
  error: 150

cyclomatic_complexity:
  warning: 10
  error: 15

function_parameter_list_length:
  warning: 5
  error: 8

type_name:
  severity: warning
  validates_start_with_lowercase: false

identifier_name:
  severity: warning
  min_length: 1
  max_length: 50

number_separator:
  severity: warning
```

## SwiftFormat Configuration

**Style Enforced:**

```swift
// Spacing
spaceInsideBrackets = false
spaceInsideParens = false

// Indentation
indent = 4

// Trailing commas
trailingCommas = true

// Import organization
importGrouping = alphabetized
groupedImports = testableFirst

// Line breaks
maxWidth = 120
wrapArguments = beforeFirst
wrapParameters = beforeFirst
```

## Layer Boundary Enforcement

**Lint Process:**

1. **Domain Layer Scan:**
   - Files in Domain/ must not import UIKit
   - Files in Domain/ must not import SwiftUI
   - Files in Domain/ must not import external frameworks
   - Violation: escalate to Architect

2. **Data Layer Scan:**
   - Files in Data/ must not import Presentation/
   - Files in Data/ can import Domain/
   - Violation: escalate to Architect

3. **Presentation Layer Scan:**
   - Files in Presentation/ must not import Data/ (except via Domain protocols)
   - Files in Presentation/ must import SwiftUI
   - Violation: escalate to Code Gen lead

**Violation Escalation:**

```
Layer Boundary Violation Detected
├── If Domain import violation
│   └── Route to: Architect + Domain Lead
├── If Data layer cross-contamination
│   └── Route to: Data Lead + Architect
└── If Presentation imports Data directly
    └── Route to: Presentation Lead + Code Gen
```

## Common Violations and Resolution

**Force Unwrap:**
- Violation: Using `!` operator
- Fix: Replace with guard let, if let, or nil-coalescing
- Auto-fixable: Partial (depends on context)

**Implicit Return:**
- Violation: Single expression without return keyword
- Fix: Add return keyword (or remove for implicit returns in computed properties)
- Auto-fixable: Yes

**Line Length:**
- Violation: Line exceeds 120 characters
- Fix: Break into multiple lines
- Auto-fixable: No (requires semantic understanding)

**Cyclomatic Complexity:**
- Violation: Function has too many branches
- Fix: Extract nested logic into helper functions
- Auto-fixable: No

**Type Naming:**
- Violation: Type name doesn't follow PascalCase
- Fix: Rename type to PascalCase
- Auto-fixable: No (requires all references updated)

## Parallel Execution

**Test and Lint Timeline:**
```
Code Generation Complete
│
├─ Test Agent Starts
│  ├─ Unit Test Gen (Domain, Data, Presentation)
│  └─ UI Test Gen (Coordinator flows)
│
├─ Lint Agent Starts (parallel with Test Agent)
│  ├─ SwiftLint Check
│  ├─ SwiftFormat Auto-Fix
│  ├─ Re-Lint Verification
│  └─ Layer Boundary Check
│
Quality Gates Pass?
├─ Yes → Build Stage
└─ No → Feedback to Coordinator
```

**Failure Handling:**
- Test failures: Coordinator → Code Gen (specific layer)
- Lint violations: Coordinator → affected lead
- Build errors: Coordinator → Build Agent

Both Test and Lint must pass (100% success rate) before Build stage entry.
