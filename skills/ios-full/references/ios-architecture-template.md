# iOS Architecture Template and Code Examples

This document provides concrete project templates and code examples for implementing the MVVM-C Clean Architecture in the ios-full-pipeline.

## Project Structure Quick Reference

```
MyApp/
├── App/
│   ├── MyApp.swift
│   ├── DI/
│   │   ├── DIContainer.swift
│   │   └── Assemblies/
│   │       ├── AuthAssembly.swift
│   │       └── SharedAssembly.swift
│   └── Coordinators/
│       └── AppCoordinator.swift
├── Presentation/
│   └── Features/
│       ├── Auth/
│       │   ├── Screens/
│       │   ├── ViewModels/
│       │   ├── Coordinators/
│       │   └── Models/
│       └── (other features)
├── Domain/
│   ├── Entities/
│   ├── UseCases/
│   └── Repositories/
└── Data/
    ├── Repositories/
    ├── DTOs/
    ├── Mappers/
    └── DataSources/
```

## App Entry Point

**File: App/MyApp.swift**

```swift
import SwiftUI

@main
struct MyApp: App {
    let container = DIContainer()

    var body: some Scene {
        WindowGroup {
            AppCoordinator(container: container)
                .start()
                .preferredColorScheme(nil) // Respect system settings
        }
    }
}
```

**Key Points:**
- DIContainer instantiated once at app launch
- AppCoordinator receives container and manages root navigation
- No SceneDelegate needed (SwiftUI 2.0+)

## Dependency Injection Container

**File: App/DI/DIContainer.swift**

```swift
import Foundation
import Domain

final class DIContainer {
    // MARK: - Repositories

    lazy var userRepository: UserRepository = {
        UserRepositoryImpl(
            apiClient: apiClient,
            mapper: userMapper
        )
    }()

    lazy var authRepository: AuthRepository = {
        AuthRepositoryImpl(
            apiClient: apiClient,
            mapper: authMapper,
            credentialsStorage: credentialsManager
        )
    }()

    lazy var productRepository: ProductRepository = {
        ProductRepositoryImpl(
            apiClient: apiClient,
            mapper: productMapper,
            cacheManager: cacheManager
        )
    }()

    // MARK: - Use Cases

    lazy var loginUseCase: LoginUseCase = {
        LoginUseCaseImpl(repository: authRepository)
    }()

    lazy var logoutUseCase: LogoutUseCase = {
        LogoutUseCaseImpl(repository: authRepository)
    }()

    lazy var fetchUserProfileUseCase: FetchUserProfileUseCase = {
        FetchUserProfileUseCaseImpl(repository: userRepository)
    }()

    lazy var updateUserProfileUseCase: UpdateUserProfileUseCase = {
        UpdateUserProfileUseCaseImpl(repository: userRepository)
    }()

    lazy var fetchProductsUseCase: FetchProductsUseCase = {
        FetchProductsUseCaseImpl(repository: productRepository)
    }()

    // MARK: - Networking

    private lazy var apiClient: APIClient = {
        HTTPClient(
            baseURL: URL(string: "https://api.example.com")!,
            session: URLSession.shared
        )
    }()

    // MARK: - Persistence

    private lazy var credentialsManager: CredentialsManager = {
        KeychainCredentialsManager()
    }()

    private lazy var cacheManager: CacheManager = {
        UserDefaultsCacheManager()
    }()

    // MARK: - Mappers

    private lazy var userMapper: UserMapper = {
        UserMapper()
    }()

    private lazy var authMapper: AuthMapper = {
        AuthMapper()
    }()

    private lazy var productMapper: ProductMapper = {
        ProductMapper()
    }()
}
```

**Container Design Principles:**
- lazy properties for deferred initialization
- Private underlying services, public repositories and use cases
- All dependencies injectable and replaceable for testing

## Domain Layer Examples

**File: Domain/Entities/User.swift**

```swift
import Foundation

struct User: Identifiable, Equatable {
    let id: String
    let name: String
    let email: String
    let profileImageURL: URL?
    let joinDate: Date
    let isActive: Bool
}
```

**Entity Design:**
- Immutable structs (no class inheritance)
- Conform to Identifiable, Equatable, Codable when needed
- No UI dependencies
- Pure Swift types only

**File: Domain/Entities/Product.swift**

```swift
import Foundation

struct Product: Identifiable, Equatable {
    let id: String
    let name: String
    let description: String
    let price: Decimal
    let imageURL: URL
    let category: String
    let inStock: Bool
}
```

## Use Case Examples

**File: Domain/UseCases/FetchUserProfileUseCase.swift**

```swift
import Foundation

protocol FetchUserProfileUseCase {
    func execute(userId: String) async throws -> User
}

final class FetchUserProfileUseCaseImpl: FetchUserProfileUseCase {
    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func execute(userId: String) async throws -> User {
        guard !userId.isEmpty else {
            throw ValidationError.emptyUserId
        }
        return try await repository.fetchUserProfile(userId: userId)
    }
}

enum ValidationError: LocalizedError {
    case emptyUserId

    var errorDescription: String? {
        switch self {
        case .emptyUserId:
            return "User ID cannot be empty"
        }
    }
}
```

**File: Domain/UseCases/LoginUseCase.swift**

```swift
import Foundation

protocol LoginUseCase {
    func execute(email: String, password: String) async throws -> AuthToken
}

final class LoginUseCaseImpl: LoginUseCase {
    private let repository: AuthRepository

    init(repository: AuthRepository) {
        self.repository = repository
    }

    func execute(email: String, password: String) async throws -> AuthToken {
        guard !email.isEmpty else {
            throw ValidationError.invalidEmail
        }
        guard password.count >= 8 else {
            throw ValidationError.weakPassword
        }

        return try await repository.login(email: email, password: password)
    }
}

enum ValidationError: LocalizedError {
    case invalidEmail
    case weakPassword

    var errorDescription: String? {
        switch self {
        case .invalidEmail:
            return "Email address is invalid"
        case .weakPassword:
            return "Password must be at least 8 characters"
        }
    }
}
```

**Use Case Design:**
- Protocol and implementation in Domain
- Receive repository protocols, not concrete implementations
- Validate inputs before delegating to repository
- Return domain entities only
- Throw domain-level errors

## Repository Pattern

**File: Domain/Repositories/UserRepository.swift**

```swift
import Foundation

protocol UserRepository {
    func fetchUserProfile(userId: String) async throws -> User
    func updateUserProfile(_ user: User) async throws
    func deleteAccount(userId: String) async throws
}
```

**File: Data/Repositories/UserRepositoryImpl.swift**

```swift
import Foundation
import Domain

final class UserRepositoryImpl: UserRepository {
    private let apiClient: APIClient
    private let mapper: UserMapper

    init(
        apiClient: APIClient,
        mapper: UserMapper
    ) {
        self.apiClient = apiClient
        self.mapper = mapper
    }

    func fetchUserProfile(userId: String) async throws -> User {
        let dto = try await apiClient.request(
            UserEndpoint.fetchProfile(userId: userId),
            as: UserDTO.self
        )
        return mapper.toDomain(dto)
    }

    func updateUserProfile(_ user: User) async throws {
        let dto = mapper.toDTO(user)
        try await apiClient.request(
            UserEndpoint.updateProfile(dto),
            as: EmptyResponse.self
        )
    }

    func deleteAccount(userId: String) async throws {
        try await apiClient.request(
            UserEndpoint.deleteAccount(userId: userId),
            as: EmptyResponse.self
        )
    }
}
```

## DTOs and Mappers

**File: Data/DTOs/UserDTO.swift**

```swift
import Foundation

struct UserDTO: Codable {
    let id: String
    let full_name: String
    let email_address: String
    let profile_image_url: String?
    let join_date: String
    let is_active: Bool

    enum CodingKeys: String, CodingKey {
        case id
        case full_name
        case email_address
        case profile_image_url
        case join_date
        case is_active
    }
}

struct UserUpdateDTO: Codable {
    let full_name: String
    let email_address: String

    enum CodingKeys: String, CodingKey {
        case full_name
        case email_address
    }
}
```

**File: Data/Mappers/UserMapper.swift**

```swift
import Foundation
import Domain

struct UserMapper {
    func toDomain(_ dto: UserDTO) -> User {
        User(
            id: dto.id,
            name: dto.full_name,
            email: dto.email_address,
            profileImageURL: dto.profile_image_url.flatMap(URL.init(string:)),
            joinDate: ISO8601DateFormatter().date(from: dto.join_date) ?? Date(),
            isActive: dto.is_active
        )
    }

    func toDTO(_ user: User) -> UserUpdateDTO {
        UserUpdateDTO(
            full_name: user.name,
            email_address: user.email
        )
    }
}
```

**Mapper Responsibilities:**
- Convert between DTOs (API response format) and domain entities
- Handle date/number format conversions
- Resolve optional fields appropriately
- Keep mappers stateless and testable

## ViewModel Examples

**File: Presentation/Features/Auth/ViewModels/LoginViewModel.swift**

```swift
import Foundation
import Combine
import Domain

@MainActor
final class LoginViewModel: ObservableObject {
    enum ViewState {
        case idle
        case loading
        case success(AuthToken)
        case error(String)
    }

    @Published var email: String = ""
    @Published var password: String = ""
    @Published var state: ViewState = .idle

    private let loginUseCase: LoginUseCase
    private var loginTask: Task<Void, Never>?

    init(loginUseCase: LoginUseCase) {
        self.loginUseCase = loginUseCase
    }

    func login() {
        loginTask?.cancel()

        loginTask = Task {
            state = .loading

            do {
                let token = try await loginUseCase.execute(
                    email: email,
                    password: password
                )
                state = .success(token)
            } catch {
                state = .error(error.localizedDescription)
            }
        }
    }

    func resetState() {
        state = .idle
    }

    deinit {
        loginTask?.cancel()
    }
}
```

**File: Presentation/Features/Profile/ViewModels/UserProfileViewModel.swift**

```swift
import Foundation
import Domain

@MainActor
final class UserProfileViewModel: ObservableObject {
    enum ViewState {
        case idle
        case loading
        case loaded(User)
        case error(String)
        case empty
    }

    @Published var state: ViewState = .idle

    private let fetchUserUseCase: FetchUserProfileUseCase
    private let updateUserUseCase: UpdateUserProfileUseCase
    private var loadTask: Task<Void, Never>?

    init(
        fetchUserUseCase: FetchUserProfileUseCase,
        updateUserUseCase: UpdateUserProfileUseCase
    ) {
        self.fetchUserUseCase = fetchUserUseCase
        self.updateUserUseCase = updateUserUseCase
    }

    func load(userId: String) {
        loadTask?.cancel()

        loadTask = Task {
            state = .loading

            do {
                let user = try await fetchUserUseCase.execute(userId: userId)
                state = .loaded(user)
            } catch {
                state = .error(error.localizedDescription)
            }
        }
    }

    func refresh(userId: String) {
        load(userId: userId)
    }

    deinit {
        loadTask?.cancel()
    }
}
```

**ViewModel Design:**
- @MainActor for thread safety
- ObservableObject for SwiftUI reactivity
- ViewState enum for screen states
- @Published properties for reactive updates
- Async/await for data loading
- Graceful task cancellation in deinit

## Coordinator Examples

**File: App/Coordinators/AppCoordinator.swift**

```swift
import SwiftUI
import Domain

@MainActor
final class AppCoordinator: View {
    @StateObject private var navigationState = AppNavigationState()
    let container: DIContainer

    init(container: DIContainer) {
        self.container = container
    }

    var body: some View {
        NavigationStack(path: $navigationState.path) {
            if navigationState.isAuthenticated {
                mainTabView()
            } else {
                AuthCoordinator(
                    container: container,
                    coordinator: self
                ).start()
            }
        }
    }

    @ViewBuilder
    private func mainTabView() -> some View {
        TabView(selection: $navigationState.selectedTab) {
            HomeCoordinator(container: container)
                .start()
                .tabItem {
                    Label("Home", systemImage: "house")
                }
                .tag(Tab.home)

            ProfileCoordinator(
                container: container,
                coordinator: self
            ).start()
                .tabItem {
                    Label("Profile", systemImage: "person")
                }
                .tag(Tab.profile)
        }
    }

    func logout() {
        navigationState.isAuthenticated = false
        navigationState.path.removeAll()
    }
}

@MainActor
final class AppNavigationState: ObservableObject {
    @Published var path = NavigationPath()
    @Published var selectedTab: AppCoordinator.Tab = .home
    @Published var isAuthenticated = false
}

extension AppCoordinator {
    enum Tab {
        case home
        case profile
    }
}
```

**File: Presentation/Features/Auth/Coordinators/AuthCoordinator.swift**

```swift
import SwiftUI
import Domain

@MainActor
final class AuthCoordinator: View {
    @StateObject private var navigationState = AuthNavigationState()
    let container: DIContainer
    weak var appCoordinator: AppCoordinator?

    init(
        container: DIContainer,
        coordinator: AppCoordinator
    ) {
        self.container = container
        self.appCoordinator = coordinator
    }

    var body: some View {
        NavigationStack(path: $navigationState.path) {
            LoginView(
                viewModel: LoginViewModel(
                    loginUseCase: container.loginUseCase
                ),
                coordinator: self
            )
            .navigationDestination(for: Route.self) { route in
                switch route {
                case .signup:
                    SignupView(
                        viewModel: SignupViewModel(
                            signupUseCase: container.signupUseCase
                        ),
                        coordinator: self
                    )
                case .forgotPassword:
                    ForgotPasswordView(coordinator: self)
                }
            }
        }
    }

    enum Route: Hashable {
        case signup
        case forgotPassword
    }

    func navigateToSignup() {
        navigationState.path.append(Route.signup)
    }

    func navigateToForgotPassword() {
        navigationState.path.append(Route.forgotPassword)
    }

    func handleLoginSuccess() {
        appCoordinator?.navigationState.isAuthenticated = true
    }
}

@MainActor
final class AuthNavigationState: ObservableObject {
    @Published var path = NavigationPath()
}
```

**Coordinator Design:**
- Owns NavigationPath and navigation logic
- Creates ViewModels with injected dependencies
- Routes represent possible navigation destinations
- Coordinator methods handle navigation transitions
- Parent coordinator notified of completion via callbacks

## SwiftUI View Examples

**File: Presentation/Features/Auth/Screens/LoginView.swift**

```swift
import SwiftUI
import Domain

struct LoginView: View {
    @StateObject private var viewModel: LoginViewModel
    let coordinator: AuthCoordinator

    init(
        viewModel: LoginViewModel,
        coordinator: AuthCoordinator
    ) {
        _viewModel = StateObject(wrappedValue: viewModel)
        self.coordinator = coordinator
    }

    var body: some View {
        VStack(spacing: 20) {
            Text("Welcome Back")
                .font(.title)
                .bold()

            VStack(spacing: 12) {
                TextField("Email", text: $viewModel.email)
                    .textInputAutocapitalization(.never)
                    .keyboardType(.emailAddress)
                    .textFieldStyle(.roundedBorder)

                SecureField("Password", text: $viewModel.password)
                    .textFieldStyle(.roundedBorder)
            }

            HStack {
                Spacer()
                Button("Forgot Password?") {
                    coordinator.navigateToForgotPassword()
                }
                .font(.caption)
            }

            Button(action: { viewModel.login() }) {
                if case .loading = viewModel.state {
                    ProgressView()
                        .tint(.white)
                } else {
                    Text("Sign In")
                }
            }
            .frame(maxWidth: .infinity)
            .padding(.vertical, 12)
            .background(Color.blue)
            .foregroundStyle(.white)
            .cornerRadius(8)
            .disabled(isLoading)

            Divider()

            HStack {
                Text("Don't have an account?")
                Button("Sign Up") {
                    coordinator.navigateToSignup()
                }
                .fontWeight(.semibold)
            }
            .font(.caption)

            Spacer()
        }
        .padding(20)
        .alert("Error", isPresented: .constant(isError), presenting: errorMessage) { _ in
            Button("OK") { viewModel.resetState() }
        } message: { message in
            Text(message)
        }
        .onChange(of: viewModel.state) { _, newState in
            if case .success = newState {
                coordinator.handleLoginSuccess()
            }
        }
    }

    private var isLoading: Bool {
        if case .loading = viewModel.state {
            return true
        }
        return false
    }

    private var isError: Bool {
        if case .error = viewModel.state {
            return true
        }
        return false
    }

    private var errorMessage: String? {
        if case .error(let message) = viewModel.state {
            return message
        }
        return nil
    }
}

#Preview {
    LoginView(
        viewModel: LoginViewModel(
            loginUseCase: PreviewLoginUseCase()
        ),
        coordinator: AuthCoordinator(
            container: DIContainer(),
            coordinator: AppCoordinator(container: DIContainer())
        )
    )
}
```

**File: Presentation/Features/Profile/Screens/UserProfileView.swift**

```swift
import SwiftUI
import Domain

struct UserProfileView: View {
    @StateObject private var viewModel: UserProfileViewModel
    let userId: String

    init(
        viewModel: UserProfileViewModel,
        userId: String
    ) {
        _viewModel = StateObject(wrappedValue: viewModel)
        self.userId = userId
    }

    var body: some View {
        switch viewModel.state {
        case .idle, .loading:
            ProgressView()
                .task {
                    await viewModel.load(userId: userId)
                }

        case .loaded(let user):
            ScrollView {
                VStack(spacing: 16) {
                    AsyncImage(url: user.profileImageURL) { phase in
                        switch phase {
                        case .empty:
                            ProgressView()
                        case .success(let image):
                            image
                                .resizable()
                                .scaledToFill()
                                .frame(width: 120, height: 120)
                                .clipShape(Circle())
                        case .failure:
                            Image(systemName: "person.crop.circle")
                                .font(.system(size: 60))
                        @unknown default:
                            EmptyView()
                        }
                    }

                    VStack(spacing: 8) {
                        Text(user.name)
                            .font(.headline)
                        Text(user.email)
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }

                    Divider()

                    VStack(alignment: .leading, spacing: 12) {
                        InfoRow(label: "Joined", value: user.joinDate.formatted())
                        InfoRow(
                            label: "Status",
                            value: user.isActive ? "Active" : "Inactive"
                        )
                    }

                    Spacer()
                }
                .padding(20)
            }
            .navigationTitle("Profile")
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button(action: { Task { await viewModel.refresh(userId: userId) } }) {
                        Image(systemName: "arrow.clockwise")
                    }
                }
            }

        case .error(let message):
            ErrorView(message: message) {
                Task { await viewModel.load(userId: userId) }
            }

        case .empty:
            EmptyStateView()
        }
    }
}

struct InfoRow: View {
    let label: String
    let value: String

    var body: some View {
        HStack {
            Text(label)
                .foregroundStyle(.secondary)
            Spacer()
            Text(value)
                .fontWeight(.semibold)
        }
    }
}
```

**View Design:**
- Receive ViewModels and Coordinators via init
- StateObject for ViewModel lifecycle management
- Switch on ViewState enum for screen states
- @Published properties reactive to changes
- Coordinator methods for navigation intent
- Minimal business logic (none in views)

## Network Endpoint Definition

**File: Data/DataSources/Remote/Endpoints/UserEndpoint.swift**

```swift
import Foundation

enum UserEndpoint {
    case fetchProfile(userId: String)
    case updateProfile(UserUpdateDTO)
    case deleteAccount(userId: String)
}

extension UserEndpoint: APIEndpoint {
    var path: String {
        switch self {
        case .fetchProfile(let userId):
            return "/users/\(userId)"
        case .updateProfile:
            return "/users/profile"
        case .deleteAccount(let userId):
            return "/users/\(userId)"
        }
    }

    var method: HTTPMethod {
        switch self {
        case .fetchProfile, .deleteAccount:
            return .get
        case .updateProfile:
            return .put
        }
    }

    var body: Encodable? {
        switch self {
        case .updateProfile(let dto):
            return dto
        default:
            return nil
        }
    }
}
```

This template provides the foundation for consistent architecture across all iOS features and modules.
