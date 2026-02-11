# Code Gen Phases Guide

Complete reference for the Code Gen team in the ios-full-pipeline skill. Covers the team structure, 4 phases, critical rules, and feedback routing.

## Team Structure

Code Gen is a team-based agent composed of specialized senior engineers.

### Team Composition

**Full Team (3+ developers on project):**
- **Principal Architect** (Senior/Lead): Researches, plans, writes blueprint and ADRs
- **Domain Lead** (Senior): Implements Domain layer (entities, use cases, errors)
- **Data Lead** (Senior): Implements Data layer (DTOs, repositories, API client, storage)
- **Presentation Lead** (Senior): Implements Presentation layer (ViewModels, Views, Coordinators)
- **Staff Integration** (Staff/Principal, 1 per team): Integrates layers, wires DI, audits separation

**Pair (2 developers):**
- **Principal Architect** + **Specialist** (handles Data + Presentation)
- OR **Principal Architect** + **Specialist** (handles Domain + Data)
- (Requires close coordination; less parallelization)

**Solo (1 developer):**
- Single developer executes all phases sequentially
- Maintains phase discipline (blueprint first, no code until blueprint complete)

## Phase 1: Architect (Principal Architect)

**Duration:** 1-2 hours for large features; 30min for bug fixes

**Responsibility:** Research, plan, and produce blueprint before any code is written.

**Critical Rule:** No code written until blueprint is complete and approved.

### Research Step

Architect researches before reading the spec. This prevents bias and ensures fresh perspective.

**Research targets:**

1. **Swift/iOS Patterns**
   - Async/await usage (when to use, error handling)
   - Property wrappers (@State, @Published, @Bindable)
   - Protocol-oriented design
   - Memory management (weak self, ARC)

2. **API Documentation**
   - API contract details (request/response schemas)
   - Rate limiting, authentication requirements
   - Error codes and handling
   - Pagination, filtering strategies

3. **Similar Implementations**
   - Codebase history (has this been built before? How?)
   - Pipeline Memory (patterns used in past features)
   - Design patterns from team (DI strategy, error handling)

4. **HIG and iOS Best Practices**
   - iOS 17+ deprecations and new APIs
   - Navigation patterns (NavigationStack vs NavigationView)
   - Accessibility requirements (VoiceOver, Dynamic Type)
   - Performance guidelines (memory, battery, network)

5. **Frameworks and Libraries**
   - SwiftUI version compatibility
   - Alamofire/URLSession choices
   - Combine vs async/await
   - CoreData vs Realm vs UserDefaults

**Research Output:**
- Notes in `.pipeline/research/` for future reference
- Decisions logged to Pipeline Memory

### Blueprint Production

After research, Architect reads the spec (Product Agent output or user input) and produces:

**1. Protocol Contracts**

Defines all protocols (interfaces) that will connect the layers. Domain protocols are abstract; other layers implement.

Example: User Profile feature

```swift
// Domain layer — User protocols (no frameworks)
protocol UserRepository {
    func fetchUser(id: String) async throws -> User
    func updateUser(id: String, name: String, email: String) async throws -> User
}

protocol UserUseCase {
    func getUser(id: String) async throws -> User
    func updateProfile(name: String, email: String) async throws -> User
}

// Presentation layer — ViewModel protocols
protocol UserProfileViewModelDelegate {
    func viewModelDidUpdateUser(_ vm: UserProfileViewModel)
    func viewModel(_ vm: UserProfileViewModel, didFailWithError error: Error)
}

protocol UserProfileViewControlling {
    var viewModel: UserProfileViewModel { get }
    func displayLoading()
    func displayUserProfile(_ user: User)
    func displayError(_ message: String)
}
```

**2. Dependency Graph**

Maps which layers depend on which. Should show: Domain has no dependencies; Data depends on Domain; Presentation depends on Domain and Data (abstract).

```
    Presentation
      ↓        ↓
    Data    Coordinator
      ↓        ↓
    Domain  Domain

    No circular dependencies allowed
```

**3. File Plan**

Specifies files to be created and their responsibilities.

```
Domain/
  ├─ Entities/
  │  ├─ User.swift                 (Codable? no. Equatable? yes)
  │  └─ UserErrors.swift           (custom error enum)
  ├─ UseCases/
  │  ├─ FetchUserUseCase.swift     (protocol + business logic)
  │  └─ UpdateUserUseCase.swift    (protocol + business logic)
  └─ Repositories/
     └─ UserRepository.swift       (protocol, no implementation)

Data/
  ├─ DTOs/
  │  ├─ UserDTO.swift             (Codable, maps from API)
  │  └─ UserMapper.swift          (DTO ↔ Entity conversion)
  ├─ Repositories/
  │  └─ UserRepositoryImpl.swift   (implements UserRepository)
  ├─ Network/
  │  ├─ APIClient.swift           (URLSession wrapper)
  │  └─ UserAPIEndpoint.swift     (endpoint definitions)
  └─ Storage/
     └─ UserLocalStorage.swift    (UserDefaults/CoreData access)

Presentation/
  ├─ ViewModels/
  │  ├─ UserProfileViewModel.swift (loads user, exposes @Published state)
  │  └─ UserProfileViewState.swift (enum for UI states)
  ├─ Views/
  │  ├─ UserProfileView.swift     (SwiftUI view)
  │  ├─ UserProfileEditView.swift (edit form)
  │  └─ Components/
  │     └─ UserAvatarView.swift   (reusable component)
  └─ Coordinators/
     └─ UserProfileCoordinator.swift (navigation, MVVM-C pattern)
```

**4. Error Types**

Defines custom error enums that will be thrown and caught.

```swift
// Domain/UserErrors.swift
enum UserError: Error, Equatable {
    case notFound
    case unauthorized
    case validationFailed(String)
    case networkError(NetworkError)
    case unknown
}

// Data/NetworkError.swift (can be shared across features)
enum NetworkError: Error {
    case invalidURL
    case timeout
    case noInternetConnection
    case serverError(statusCode: Int)
}
```

**5. Coordinator Navigation Plan**

For features involving navigation, defines routes and transitions.

```swift
// Presentation/Coordinators/NavigationRoute.swift
enum UserProfileRoute {
    case profile(userId: String)
    case edit(userId: String)
    case settings
}

// Coordinator flow:
// 1. App launches with userId
// 2. Show UserProfileView
// 3. User taps "Edit" → show UserProfileEditView
// 4. User saves → back to UserProfileView (refreshed)
// 5. User taps "Settings" → navigate to settings (outside profile flow)
```

### ADR (Architecture Decision Record)

Architect writes an ADR documenting key decisions and tradeoffs.

**File:** `docs/decisions/ADR-NNN-[title].md`

**Example:**

```markdown
# ADR-005: Use Async/Await Instead of Combine for User Profile

## Context
We need to fetch and display user profiles. The team uses both Combine publishers and async/await.

## Decision
Use async/await throughout User Profile feature (domain, data, presentation).

## Rationale
- Async/await is more readable (sequential code flow vs publisher chains)
- Async/await is the Swift standard (iOS 13.0+, our min target is 14.0)
- Async/await has better error handling (try/catch vs .mapError)
- Team expertise is growing in async/await; Combine usage is declining

## Consequences
- Positive: Cleaner code, easier to test, less boilerplate
- Negative: No existing Combine operators (map, filter chains) available
- Neutral: Both coexist in codebase (other features use Combine)

## Alternatives Considered
1. Pure Combine: More consistent with legacy code, but harder to read
2. RxSwift: Overkill for this feature, external dependency
3. Callbacks/Closures: Callback hell; hard to manage multiple async operations

## References
- https://developer.apple.com/videos/play/wwdc2021/10132/
- [Internal async/await guide: docs/patterns/async-await.md]
```

### Blueprint Approval

Architect submits blueprint (protocol contracts, dependency graph, file plan, error types, ADR).

**Review Questions:**
- Are all dependencies unidirectional (no cycles)?
- Do protocols define all needed operations?
- Is the Domain layer free of framework imports?
- Are error types clear and exhaustive?
- Is navigation plan feasible?

**Approval:** Reviewed by dev lead or team; approved = proceed to Phase 2.

## Phase 2: Domain Lead (Senior)

**Duration:** 1-3 hours (depends on feature complexity)

**Responsibility:** Implement the Domain layer using the blueprint.

**Critical Rule:** ZERO imports of UIKit, SwiftUI, Combine, Alamofire, or any external framework. Pure Swift only.

### Domain Layer Scope

Domain layer contains:

1. **Entities** (struct, Equatable, NOT Codable)
2. **Use Cases** (business logic, protocols)
3. **Repository Protocols** (abstract data access)
4. **Error Types** (custom errors)

### Implementation Process

**Step 1: Entities**

Create Swift structs representing domain objects. No Codable, no framework dependencies.

```swift
// Domain/Entities/User.swift
import Foundation

struct User: Equatable {
    let id: String
    let name: String
    let email: String
    let avatarURL: URL?
    let createdAt: Date

    // No Codable conformance; DTOs handle JSON
    // No @Published; ViewModels add that
}

struct Post: Equatable {
    let id: String
    let userId: String
    let title: String
    let body: String
    let imageURLs: [URL]
    let createdAt: Date
}
```

**Step 2: Use Cases (Business Logic)**

Implement protocols that define business operations.

```swift
// Domain/UseCases/FetchUserUseCase.swift
protocol FetchUserUseCase {
    func execute(userId: String) async throws -> User
}

// Domain/UseCases/UpdateUserUseCase.swift
protocol UpdateUserUseCase {
    func execute(userId: String, name: String, email: String) async throws -> User
}

// Implementations (in Domain, not Data) — pure logic
class FetchUserUseCaseImpl: FetchUserUseCase {
    let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func execute(userId: String) async throws -> User {
        guard !userId.isEmpty else {
            throw UserError.invalidInput("User ID cannot be empty")
        }

        let user = try await repository.fetchUser(id: userId)

        // Validation: business logic, not API logic
        guard !user.name.isEmpty else {
            throw UserError.validationFailed("User name is required")
        }

        return user
    }
}

class UpdateUserUseCaseImpl: UpdateUserUseCase {
    let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func execute(userId: String, name: String, email: String) async throws -> User {
        // Validate input
        guard !name.isEmpty else {
            throw UserError.validationFailed("Name cannot be empty")
        }

        guard email.contains("@") else {
            throw UserError.validationFailed("Invalid email format")
        }

        // Call repository
        let updatedUser = try await repository.updateUser(id: userId, name: name, email: email)

        return updatedUser
    }
}
```

**Step 3: Repository Protocols**

Define data access interfaces (no implementation; Data layer implements).

```swift
// Domain/Repositories/UserRepository.swift
protocol UserRepository {
    func fetchUser(id: String) async throws -> User
    func updateUser(id: String, name: String, email: String) async throws -> User
    func deleteUser(id: String) async throws
}
```

**Step 4: Error Types**

Define exhaustive error enums for this feature.

```swift
// Domain/UserErrors.swift
enum UserError: Error, Equatable {
    case notFound
    case unauthorized
    case invalidInput(String)
    case validationFailed(String)
    case networkError(NetworkError)
    case unknown
}

// Domain/NetworkErrors.swift (shared across domain)
enum NetworkError: Error, Equatable {
    case invalidURL
    case timeout
    case noInternetConnection
    case serverError(statusCode: Int, message: String)
}
```

### Domain Tests

Domain Lead also writes tests for use cases.

```swift
// Tests/Domain/UseCases/FetchUserUseCaseTests.swift
import XCTest
@testable import Domain

final class FetchUserUseCaseTests: XCTestCase {
    var useCase: FetchUserUseCase!
    var mockRepository: MockUserRepository!

    override func setUp() {
        super.setUp()
        mockRepository = MockUserRepository()
        useCase = FetchUserUseCaseImpl(repository: mockRepository)
    }

    func testExecute_WithValidUserId_ReturnsFetchedUser() async throws {
        // Arrange
        let userId = "user-123"
        let expectedUser = User(id: userId, name: "John", email: "john@example.com", avatarURL: nil, createdAt: Date())
        mockRepository.fetchUserResult = .success(expectedUser)

        // Act
        let result = try await useCase.execute(userId: userId)

        // Assert
        XCTAssertEqual(result, expectedUser)
    }

    func testExecute_WithEmptyUserId_ThrowsError() async {
        // Arrange
        let userId = ""

        // Act & Assert
        do {
            _ = try await useCase.execute(userId: userId)
            XCTFail("Expected error")
        } catch let error as UserError {
            XCTAssertEqual(error, UserError.invalidInput("User ID cannot be empty"))
        }
    }
}
```

## Phase 3: Data Lead || Presentation Lead (Parallel)

These two leads work simultaneously. They depend only on Domain layer (which is complete), not on each other.

### Data Lead Responsibilities

Implements the Data layer: DTOs, mappers, repository implementations, API client, local storage.

**Files created:**

```
Data/DTOs/UserDTO.swift
Data/Mappers/UserMapper.swift
Data/Repositories/UserRepositoryImpl.swift
Data/Network/APIClient.swift
Data/Network/UserAPIEndpoint.swift
Data/Storage/UserLocalStorage.swift
```

**Example: DTOs**

```swift
// Data/DTOs/UserDTO.swift
import Foundation

struct UserDTO: Codable {
    let id: String
    let name: String
    let email: String
    let avatarUrl: String?
    let createdAt: String // ISO 8601 from API

    enum CodingKeys: String, CodingKey {
        case id
        case name
        case email
        case avatarUrl = "avatar_url"
        case createdAt = "created_at"
    }
}
```

**Example: Mappers**

```swift
// Data/Mappers/UserMapper.swift
import Foundation

final class UserMapper {
    static func map(_ dto: UserDTO) -> User {
        return User(
            id: dto.id,
            name: dto.name,
            email: dto.email,
            avatarURL: dto.avatarUrl.flatMap(URL.init(string:)),
            createdAt: ISO8601DateFormatter().date(from: dto.createdAt) ?? Date()
        )
    }

    static func mapDTO(_ user: User) -> [String: Any] {
        return [
            "id": user.id,
            "name": user.name,
            "email": user.email,
            "avatar_url": user.avatarURL?.absoluteString ?? ""
        ]
    }
}
```

**Example: Repository Implementation**

```swift
// Data/Repositories/UserRepositoryImpl.swift
import Foundation

final class UserRepositoryImpl: UserRepository {
    let apiClient: APIClient
    let localStorage: UserLocalStorage

    init(apiClient: APIClient, localStorage: UserLocalStorage) {
        self.apiClient = apiClient
        self.localStorage = localStorage
    }

    func fetchUser(id: String) async throws -> User {
        do {
            let dto = try await apiClient.fetch(UserDTO.self, from: .user(id))
            let user = UserMapper.map(dto)
            try localStorage.save(user: user)
            return user
        } catch {
            // Fallback to local storage if network fails
            if let cached = try localStorage.fetch(id: id) {
                return cached
            }
            throw error
        }
    }

    func updateUser(id: String, name: String, email: String) async throws -> User {
        let dto = try await apiClient.request(
            method: .PATCH,
            endpoint: .user(id),
            body: ["name": name, "email": email],
            responseType: UserDTO.self
        )
        let user = UserMapper.map(dto)
        try localStorage.save(user: user)
        return user
    }

    func deleteUser(id: String) async throws {
        try await apiClient.request(method: .DELETE, endpoint: .user(id), responseType: EmptyResponse.self)
        try localStorage.delete(id: id)
    }
}
```

**Data Lead also writes tests:**

```swift
// Tests/Data/Repositories/UserRepositoryImplTests.swift
final class UserRepositoryImplTests: XCTestCase {
    var repository: UserRepository!
    var mockAPIClient: MockAPIClient!
    var mockStorage: MockUserLocalStorage!

    func testFetchUser_WithValidId_ReturnsMappedUser() async throws {
        // Arrange
        let dto = UserDTO(id: "123", name: "John", email: "john@example.com", ...)
        mockAPIClient.fetchResult = .success(dto)

        // Act
        let user = try await repository.fetchUser(id: "123")

        // Assert
        XCTAssertEqual(user.name, "John")
        XCTAssertTrue(mockStorage.saveCalled)
    }

    func testFetchUser_OnNetworkError_FallsBackToLocalStorage() async throws {
        // Arrange
        mockAPIClient.fetchResult = .failure(NetworkError.timeout)
        mockStorage.cachedUser = User(...) // cached

        // Act
        let user = try await repository.fetchUser(id: "123")

        // Assert
        XCTAssertNotNil(user)
    }
}
```

### Presentation Lead Responsibilities

Implements the Presentation layer: ViewModels, Views, Coordinators.

**Files created:**

```
Presentation/ViewModels/UserProfileViewModel.swift
Presentation/ViewModels/UserProfileViewState.swift
Presentation/Views/UserProfileView.swift
Presentation/Views/UserProfileEditView.swift
Presentation/Coordinators/UserProfileCoordinator.swift
```

**Example: ViewModel**

```swift
// Presentation/ViewModels/UserProfileViewModel.swift
import Foundation
import Combine

@MainActor
final class UserProfileViewModel: ObservableObject {
    @Published var state: UserProfileViewState = .loading

    private let userId: String
    private let fetchUseCase: FetchUserUseCase
    private let updateUseCase: UpdateUserUseCase

    init(userId: String, fetchUseCase: FetchUserUseCase, updateUseCase: UpdateUserUseCase) {
        self.userId = userId
        self.fetchUseCase = fetchUseCase
        self.updateUseCase = updateUseCase

        Task {
            await loadUser()
        }
    }

    func loadUser() async {
        state = .loading
        do {
            let user = try await fetchUseCase.execute(userId: userId)
            state = .loaded(user)
        } catch {
            state = .error(error.localizedDescription)
        }
    }

    func updateProfile(name: String, email: String) async {
        do {
            let updatedUser = try await updateUseCase.execute(userId: userId, name: name, email: email)
            state = .loaded(updatedUser)
        } catch {
            state = .error(error.localizedDescription)
        }
    }
}

enum UserProfileViewState: Equatable {
    case loading
    case loaded(User)
    case error(String)
}
```

**Example: SwiftUI View**

```swift
// Presentation/Views/UserProfileView.swift
import SwiftUI

struct UserProfileView: View {
    @StateObject private var viewModel: UserProfileViewModel
    @State private var isEditing = false

    init(userId: String, viewModel: UserProfileViewModel) {
        _viewModel = StateObject(wrappedValue: viewModel)
    }

    var body: some View {
        switch viewModel.state {
        case .loading:
            ProgressView()
                .task {
                    await viewModel.loadUser()
                }

        case .loaded(let user):
            VStack {
                // Profile header
                AsyncImage(url: user.avatarURL) { phase in
                    switch phase {
                    case .success(let image):
                        image.resizable().scaledToFill()
                    default:
                        Color.gray
                    }
                }
                .frame(width: 100, height: 100)
                .clipShape(Circle())

                // Profile details
                Text(user.name).font(.headline)
                Text(user.email).font(.caption)

                // Edit button
                Button("Edit Profile") {
                    isEditing = true
                }
                .sheet(isPresented: $isEditing) {
                    UserProfileEditView(user: user, viewModel: viewModel)
                }

                Spacer()
            }
            .padding()

        case .error(let message):
            VStack {
                Text("Error: \(message)").foregroundColor(.red)
                Button("Retry") {
                    Task {
                        await viewModel.loadUser()
                    }
                }
            }
            .padding()
        }
    }
}
```

**Example: Coordinator**

```swift
// Presentation/Coordinators/UserProfileCoordinator.swift
import SwiftUI

@MainActor
class UserProfileCoordinator: NSObject, UINavigationControllerDelegate {
    let navigationController: UINavigationController
    let dependencies: UserProfileDependencies

    init(navigationController: UINavigationController, dependencies: UserProfileDependencies) {
        self.navigationController = navigationController
        self.dependencies = dependencies
    }

    func start(with userId: String) {
        let viewModel = UserProfileViewModel(
            userId: userId,
            fetchUseCase: dependencies.fetchUserUseCase,
            updateUseCase: dependencies.updateUserUseCase
        )
        let view = UserProfileView(userId: userId, viewModel: viewModel)
        let hostingController = UIHostingController(rootView: view)
        navigationController.pushViewController(hostingController, animated: true)
    }
}

struct UserProfileDependencies {
    let fetchUserUseCase: FetchUserUseCase
    let updateUserUseCase: UpdateUserUseCase
}
```

**Presentation Lead writes tests:**

```swift
// Tests/Presentation/ViewModels/UserProfileViewModelTests.swift
@MainActor
final class UserProfileViewModelTests: XCTestCase {
    var viewModel: UserProfileViewModel!
    var mockFetchUseCase: MockFetchUserUseCase!
    var mockUpdateUseCase: MockUpdateUserUseCase!

    func testLoadUser_WithValidUserId_UpdatesStateToLoaded() async {
        // Arrange
        let expectedUser = User(id: "123", name: "John", ...)
        mockFetchUseCase.executeResult = .success(expectedUser)

        // Act
        viewModel = UserProfileViewModel(
            userId: "123",
            fetchUseCase: mockFetchUseCase,
            updateUseCase: mockUpdateUseCase
        )
        await viewModel.loadUser()

        // Assert
        if case .loaded(let user) = viewModel.state {
            XCTAssertEqual(user, expectedUser)
        } else {
            XCTFail("Expected loaded state")
        }
    }
}
```

## Phase 4: Integration (Staff)

**Duration:** 1 hour

**Responsibility:** Wire the DI container, ensure all dependencies resolve correctly, audit layer separation.

### Integration Tasks

**1. DI Container (Assembly Pattern)**

```swift
// App/DependencyContainer.swift
final class DependencyContainer {
    // Singletons
    private lazy var apiClient: APIClient = {
        return APIClientImpl(baseURL: URL(string: "https://api.example.com")!)
    }()

    private lazy var localStorage: UserLocalStorage = {
        return UserLocalStorageImpl()
    }()

    // Domain layer
    private lazy var userRepository: UserRepository = {
        return UserRepositoryImpl(apiClient: apiClient, localStorage: localStorage)
    }()

    private lazy var fetchUserUseCase: FetchUserUseCase = {
        return FetchUserUseCaseImpl(repository: userRepository)
    }()

    private lazy var updateUserUseCase: UpdateUserUseCase = {
        return UpdateUserUseCaseImpl(repository: userRepository)
    }()

    // Presentation layer
    func userProfileViewModel(userId: String) -> UserProfileViewModel {
        return UserProfileViewModel(
            userId: userId,
            fetchUseCase: fetchUserUseCase,
            updateUseCase: updateUserUseCase
        )
    }

    func userProfileCoordinator(navigationController: UINavigationController) -> UserProfileCoordinator {
        let dependencies = UserProfileDependencies(
            fetchUserUseCase: fetchUserUseCase,
            updateUserUseCase: updateUserUseCase
        )
        return UserProfileCoordinator(navigationController: navigationController, dependencies: dependencies)
    }
}
```

**2. Conformance Check**

Verify all protocols are implemented:

```swift
// Check: APIClient protocol is implemented ✓
// Check: UserRepository protocol is implemented ✓
// Check: FetchUserUseCase protocol is implemented ✓
// Check: UpdateUserUseCase protocol is implemented ✓
// All use cases are injectable ✓
```

**3. Separation Audit**

```
Audit Results:
  Domain layer:
    ✓ Zero UIKit imports
    ✓ Zero SwiftUI imports
    ✓ Zero Combine imports
    ✓ Zero Alamofire imports
    ✓ No external dependencies

  Data layer:
    ✓ Does not import Presentation
    ✓ Imports Domain (protocols)
    ✓ Can import Alamofire, URLSession
    ✓ Can import Codable, Foundation

  Presentation layer:
    ✓ Does not import Data (imports Domain only)
    ✓ Imports Domain (protocols, entities)
    ✓ Imports SwiftUI, Combine
    ✓ DependencyContainer injects Data layer via protocols

  Dependency flow:
    Presentation → (Domain, DI) → Data → Domain ✓ (valid, no cycles)
```

**4. Testability Audit**

```
Testability Checklist:
  ✓ All dependencies are injectable (constructor injection)
  ✓ All dependencies are protocols (not concrete classes)
  ✓ Mocks can be easily created (protocol + mock implementations)
  ✓ No static dependencies
  ✓ No singletons outside DI container
  ✓ ViewModels are observable (@Published)
  ✓ Coordinators are injectable
```

### Integration Tests

Integration writes tests that verify layers work together:

```swift
// Tests/Integration/UserProfileIntegrationTests.swift
final class UserProfileIntegrationTests: XCTestCase {
    var container: DependencyContainer!

    override func setUp() {
        super.setUp()
        container = DependencyContainer()
    }

    @MainActor
    func testUserProfileFlow_FetchThenUpdate() async throws {
        // Use real implementations (or mocks for network)
        let viewModel = container.userProfileViewModel(userId: "123")

        // Load user
        await viewModel.loadUser()
        guard case .loaded(let user) = viewModel.state else {
            XCTFail("Expected loaded state")
            return
        }

        // Update user
        await viewModel.updateProfile(name: "Jane", email: "jane@example.com")
        guard case .loaded(let updatedUser) = viewModel.state else {
            XCTFail("Expected loaded state after update")
            return
        }

        XCTAssertEqual(updatedUser.name, "Jane")
    }
}
```

## Feedback Routes

When compilation fails, tests fail, or architectural issues arise, Coordinator routes feedback to the appropriate agent.

**ViewModel Test Failure → Presentation Lead**
```
Test: "testLoadUser_UpdatesStateToLoaded" fails
Error: "Type 'UserProfileViewModel' has no member 'state'"
Route: Presentation Lead
Fix: Add @Published var state property; ensure init loads user
```

**Repository Test Failure → Data Lead**
```
Test: "testFetchUser_WithNetworkError_FallsBackToLocalStorage" fails
Error: "APIClient does not conform to protocol 'APIClientProtocol'"
Route: Data Lead
Fix: Ensure APIClientImpl conforms to protocol; implement missing methods
```

**Use Case Test Failure → Domain Lead**
```
Test: "testExecute_WithEmptyUserId_ThrowsError" fails
Error: "UserError.invalidInput(_:) requires argument"
Route: Domain Lead
Fix: Fix error enum or use case logic
```

**Layer Boundary Violation → Architect + Affected Lead**
```
Error: "Presentation imports from Data (should import Domain only)"
Route: Architect + Presentation Lead
Fix: Refactor to use protocols; move concrete classes to Data layer
```

**Compile Error (General) → Integration or Layer Lead**
```
Error: "Cannot find type 'UserRepository' in scope"
Route: Integration (if DI issue) or Domain Lead (if protocol missing)
Fix: Wire DI container; ensure protocol is exported from Domain module
```

**DI Audit Failure → Architect + Integration**
```
Error: "UserLocalStorage not wired in DependencyContainer"
Route: Integration
Fix: Add lazy property and wiring logic; audit dependency tree
```
