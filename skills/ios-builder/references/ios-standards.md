# iOS Development Standards for ios-builder

This document defines the comprehensive iOS development standards enforced by the ios-builder skill. All code generated, reviewed, and tested must adhere to these standards.

## Swift and SwiftUI Requirements

**Swift Version and Language Features:**
- Minimum Swift 5.9+
- Adopt Swift 5.9+ features: async/await, actors, structured concurrency
- Use modern concurrency patterns exclusively for asynchronous operations
- No legacy callback-based or GCD-heavy code in new implementations

**UI Framework Mandate:**
- SwiftUI only for new code
- UIKit wrapping allowed only for legacy components under active migration
- Declarative UI composition over imperative view manipulation
- View hierarchy must reflect domain structure
- No dependency on UIKit for new feature development

## Architecture: MVVM-C with Clean Architecture

The codebase follows a three-layer Clean Architecture with Coordinator pattern for navigation.

**Three-Layer Design:**

1. **Presentation Layer**
   - SwiftUI Views (pure presentation logic)
   - ViewModels (state management, business logic coordination)
   - Coordinators (navigation orchestration)
   - ViewState enums (loading, loaded, error, empty states)

2. **Domain Layer**
   - Entities (immutable, pure Swift structs)
   - Use Case protocols and implementations
   - Repository protocols (data access contracts)
   - Business rules enforcement
   - No external framework dependencies

3. **Data Layer**
   - DTOs (Data Transfer Objects, Codable structs)
   - Mappers (DTO ↔ Domain entity conversion)
   - Repository implementations
   - APIClient and network handling
   - Persistence layer (CoreData, UserDefaults, etc.)

**App/DI Layer:**
- DIContainer and assemblies
- Dependency injection wiring at app launch
- Manual container implementation
- No third-party DI frameworks (Swinject prohibited)

## Dependency Injection Strategy

**Protocol-Driven Manual DI:**
- All services exposed via protocols
- DIContainer instantiates and wires dependencies at app launch
- Constructor injection via init()
- No service locator pattern
- No runtime property injection
- All dependencies injectable for testability

**Container Structure:**
```swift
// App/DI/DIContainer.swift
final class DIContainer {
    // Repositories
    lazy var userRepository: UserRepository = { ... }()
    lazy var authRepository: AuthRepository = { ... }()

    // Use Cases
    lazy var fetchUserUseCase: FetchUserUseCase = { ... }()
    lazy var loginUseCase: LoginUseCase = { ... }()

    // Networking
    private lazy var apiClient: APIClient = { ... }()

    // Persistence
    private lazy var persistenceManager: PersistenceManager = { ... }()
}
```

## Navigation: MVVM-C Coordinator Pattern

**Coordinator Responsibilities:**
- Own all navigation logic for feature
- Maintain NavigationPath for SwiftUI navigation
- Manage parent-child coordinator hierarchy
- Enable deep linking via coordinator chain
- Create and inject ViewModels with dependencies

**Coordinator Hierarchy Example:**
```
AppCoordinator
├── AuthCoordinator
│   ├── LoginCoordinator
│   └── SignupCoordinator
└── MainTabCoordinator
    ├── HomeCoordinator
    ├── ProfileCoordinator
    └── SettingsCoordinator
```

**Deep Linking:**
- Coordinators support URL-based navigation
- Each coordinator handles its feature routes
- Chain of responsibility for unhandled routes
- Example: deeplink://user/profile/123 routes through AppCoordinator → MainTabCoordinator → ProfileCoordinator

## State Management

**ViewModel State Pattern:**
- ViewModels are ObservableObject with @Published properties
- State encapsulated in ViewState enums (not scattered properties)
- @MainActor annotation on all ViewModels
- Immutable state transitions

**ViewState Enum Pattern:**
```swift
enum ViewState {
    case idle
    case loading
    case loaded(T)
    case error(String)
    case empty
}
```

**Async Data Loading:**
- Use async/await in ViewModels exclusively
- Load data in task() in SwiftUI views
- Handle loading/error/empty states via switch on @Published state
- No Combine operators in new code (prefer AsyncSequence for streams)

**Observable Updates:**
- @Published properties trigger view updates
- State changes atomic and predictable
- No direct property manipulation from views
- View models expose intent-based methods (load(), refresh(), deleteItem(id:))

## Naming Conventions (Swift Style)

**Type Names (PascalCase):**
- ViewModels: `[Feature]ViewModel` (e.g., UserProfileViewModel, LoginViewModel)
- Views: `[Feature]View` (e.g., SettingsView, ProfileView)
- Coordinators: `[Feature]Coordinator` (e.g., AuthCoordinator, HomeCoordinator)
- Entities: `[Entity]` (e.g., User, Product, Order)
- DTOs: `[Name]DTO` or `[Name]Response` (e.g., UserDTO, LoginResponse)

**Protocol Names:**
- Ability/capability: `-able`, `-ible` (e.g., Fetchable, Encodable)
- Type: noun form (e.g., UserRepository, APIClient, DataPersistence)
- No "Delegate" or "Manager" suffixes unless justified

**Method/Property Names (camelCase):**
- Action methods: verb phrases (fetchUser, deleteAccount, sendMessage)
- Query methods: verb phrases (getUserProfile, hasPermission)
- Properties: noun phrases (isLoading, currentUser, errorMessage)

**Use Case Names:**
- Pattern: `[Verb][Noun]UseCase` (e.g., FetchUserProfileUseCase, DeleteAccountUseCase)
- Implementation: `[Verb][Noun]UseCaseImpl` (e.g., FetchUserProfileUseCaseImpl)
- Protocol lives in Domain, implementation in Data layer

**Mapper Names:**
- Pattern: `[Entity]Mapper` (e.g., UserMapper, ProductMapper)
- Methods: `toDomain(_ dto:)`, `toDTO(_ entity:)`

## File Organization Structure

**Complete Project Structure Tree:**

```
MyApp/
├── App/
│   ├── MyApp.swift (@main entry point)
│   ├── DI/
│   │   ├── DIContainer.swift
│   │   ├── Assemblies/
│   │   │   ├── AuthAssembly.swift
│   │   │   ├── HomeAssembly.swift
│   │   │   └── SharedAssembly.swift
│   │   └── Extensions/
│   │       └── DIContainer+Extensions.swift
│   └── Coordinators/
│       ├── AppCoordinator.swift
│       └── (feature coordinators in feature folders)
│
├── Presentation/
│   ├── Features/
│   │   ├── Auth/
│   │   │   ├── Screens/
│   │   │   │   ├── LoginView.swift
│   │   │   │   └── SignupView.swift
│   │   │   ├── ViewModels/
│   │   │   │   ├── LoginViewModel.swift
│   │   │   │   └── SignupViewModel.swift
│   │   │   ├── Coordinators/
│   │   │   │   └── AuthCoordinator.swift
│   │   │   └── Models/
│   │   │       └── AuthViewState.swift
│   │   ├── Home/
│   │   │   ├── Screens/
│   │   │   ├── ViewModels/
│   │   │   ├── Coordinators/
│   │   │   └── Models/
│   │   └── Profile/
│   │       └── (same structure)
│   └── Shared/
│       ├── Components/
│       │   ├── LoadingView.swift
│       │   ├── ErrorView.swift
│       │   └── EmptyStateView.swift
│       ├── Extensions/
│       │   └── View+Extensions.swift
│       └── Utilities/
│           └── DateFormatter+Helpers.swift
│
├── Domain/
│   ├── Entities/
│   │   ├── User.swift
│   │   ├── Product.swift
│   │   └── Order.swift
│   ├── UseCases/
│   │   ├── Auth/
│   │   │   ├── LoginUseCase.swift
│   │   │   └── LogoutUseCase.swift
│   │   ├── User/
│   │   │   ├── FetchUserProfileUseCase.swift
│   │   │   └── UpdateUserProfileUseCase.swift
│   │   └── Product/
│   │       ├── FetchProductsUseCase.swift
│   │       └── SearchProductsUseCase.swift
│   └── Repositories/
│       ├── UserRepository.swift
│       ├── AuthRepository.swift
│       └── ProductRepository.swift
│
├── Data/
│   ├── Repositories/
│   │   ├── UserRepositoryImpl.swift
│   │   ├── AuthRepositoryImpl.swift
│   │   └── ProductRepositoryImpl.swift
│   ├── DataSources/
│   │   ├── Remote/
│   │   │   ├── APIClient.swift
│   │   │   ├── Endpoints/
│   │   │   │   ├── AuthEndpoint.swift
│   │   │   │   ├── UserEndpoint.swift
│   │   │   │   └── ProductEndpoint.swift
│   │   │   └── NetworkService.swift
│   │   └── Local/
│   │       ├── UserDefaults/
│   │       │   └── PreferencesManager.swift
│   │       └── CoreData/
│   │           ├── CoreDataStack.swift
│   │           └── CoreDataModels/
│   ├── DTOs/
│   │   ├── UserDTO.swift
│   │   ├── ProductDTO.swift
│   │   └── LoginResponseDTO.swift
│   ├── Mappers/
│   │   ├── UserMapper.swift
│   │   ├── ProductMapper.swift
│   │   └── AuthMapper.swift
│   └── Networking/
│       ├── HTTPClient.swift
│       ├── Request.swift
│       └── Response.swift
│
└── Resources/
    ├── Localizable.strings
    ├── Assets.xcassets/
    └── Generated/
        └── Strings.swift (generated)
```

## Layer Import Rules

**Domain Layer:**
- Import: Foundation only
- Prohibited: UIKit, SwiftUI, any external frameworks
- Result: Pure Swift, framework-agnostic business logic

**Data Layer:**
- Import: Domain + Foundation + networking/persistence frameworks
- Can import: Alamofire, URLSession, CoreData, etc.
- Cannot import: Presentation, SwiftUI, UIKit (except for legacy persistence)

**Presentation Layer:**
- Import: Domain + SwiftUI + Foundation
- Cannot import: Data layer classes/implementations
- Only import Data layer protocol definitions from Domain
- Exception: Can use SwiftUI system frameworks

**App/DI Layer:**
- Import: All layers (Domain, Data, Presentation)
- Responsibility: Wire entire dependency graph
- Located: App/ folder, accessed only at app initialization

**Import Violation Checks:**
```swift
// ✅ CORRECT - Domain imports only Foundation
import Foundation
struct User { ... }

// ❌ WRONG - Domain imports SwiftUI
import SwiftUI
struct User { ... }

// ✅ CORRECT - Presentation imports Domain and SwiftUI
import SwiftUI
import Domain
struct UserView: View { ... }

// ❌ WRONG - Presentation imports Data directly
import SwiftUI
import Data  // VIOLATION
struct UserView: View { ... }
```

## Lint Rules and Code Quality

**SwiftLint Configuration:**
- Custom rules enforce layer boundaries
- Naming convention rules enforce Swift style guide
- Complexity rules limit function length and cyclomatic complexity
- File organization rules enforce folder structure adherence

**Key SwiftLint Rules:**
- `force_unwrap`: Error (no !), exception: tests
- `implicit_return`: Warning (use single-line returns)
- `line_length`: 120 characters
- `cyclomatic_complexity`: max 10
- `function_parameter_list_length`: max 5 parameters
- `type_name`: PascalCase
- `identifier_name`: camelCase
- `trailing_whitespace`: Error
- `vertical_whitespace`: max 1 blank line

**SwiftFormat Configuration:**
- Auto-format on every lint run
- Consistent spacing (2 spaces)
- Trailing commas in multi-line collections
- Organized imports (Foundation, then third-party, then local)

**Lint Execution:**
- Runs in Integration phase (Lite pipeline) or Lint stage (Full pipeline)
- Auto-fixes style violations
- Escalates semantic violations to Code Gen lead
- Must pass before Build stage

## Testing Patterns

**Unit Test Strategy by Layer:**

**Domain Layer Tests:**
- Test use cases directly with pure Swift
- No mocks needed (use cases receive protocol-based repositories)
- Mock repositories return hardcoded test data
- Focus: business logic correctness
- Example: FetchUserProfileUseCaseTests

**Data Layer Tests:**
- Mock APIClient/network calls
- Test DTO parsing and mapping
- Test repository implementations
- Focus: data flow and transformation
- Example: UserRepositoryImplTests mocks APIClient

**Presentation Layer Tests:**
- Mock all use cases (passed to ViewModels)
- Test ViewModel state transitions
- Test published property changes
- Focus: state management and error handling
- Example: LoginViewModelTests mocks LoginUseCase

**UI Layer Tests:**
- XCUITest for coordinator flows
- Test screen navigation paths
- Test user interaction flows
- Focus: navigation correctness
- Example: AuthFlowUITests

**Test Naming Convention:**
- Pattern: `test_[methodName]_[givenCondition]_[expectedResult]`
- Examples:
  - `test_fetchUser_withValidId_returnsUserData`
  - `test_login_withInvalidCredentials_showsErrorState`
  - `test_viewModel_onViewAppear_loadsUserProfile`

**Protocol-Based Mocking:**
```swift
// Mock repository for testing use case
final class MockUserRepository: UserRepository {
    var fetchUserResult: Result<User, Error> = .failure(NSError(domain: "", code: -1))

    func fetchUser(id: String) async throws -> User {
        switch fetchUserResult {
        case .success(let user): return user
        case .failure(let error): throw error
        }
    }
}
```

**Test Coverage Expectations:**
- Domain layer: 90%+ coverage
- Data layer: 85%+ coverage
- Presentation layer: 80%+ coverage (focus on state transitions)
- UI layer: critical flows only (authentication, core navigation)

## Code Quality Gates

All code must pass before merge:

1. **Lint Stage**: SwiftLint checks + SwiftFormat auto-fix
2. **Test Stage**: Unit tests + UI tests must pass
3. **Build Stage**: Compilation must succeed
4. **Architecture Review**: Layer boundaries verified
5. **Manual Code Review**: Developer reviews before merge

No exceptions to these gates for production code.
