# iOS Architecture Code Templates

## Project Structure Tree

```
TimerApp/
├── App/
│   ├── DI/
│   │   └── Container.swift
│   └── main.swift
├── Domain/
│   ├── Entities/
│   │   └── Timer.swift
│   ├── UseCases/
│   │   ├── GetTimersUseCase.swift
│   │   ├── StartTimerUseCase.swift
│   │   ├── PauseTimerUseCase.swift
│   │   ├── ResetTimerUseCase.swift
│   │   └── DeleteTimerUseCase.swift
│   ├── Repositories/
│   │   └── TimerRepository.swift
│   └── Errors/
│       └── DomainError.swift
├── Data/
│   ├── DTOs/
│   │   └── TimerDTO.swift
│   ├── Mappers/
│   │   └── TimerMapper.swift
│   ├── Repositories/
│   │   └── DefaultTimerRepository.swift
│   └── Network/
│       ├── APIClient.swift
│       └── URLSessionConfiguration.swift
└── Presentation/
    ├── Views/
    │   ├── TimerListView.swift
    │   ├── TimerDetailView.swift
    │   └── CreateTimerView.swift
    ├── ViewModels/
    │   ├── TimerListViewModel.swift
    │   ├── TimerDetailViewModel.swift
    │   └── CreateTimerViewModel.swift
    └── Coordinators/
        └── TimerCoordinator.swift
```

---

## App Entry Point

### main.swift

```swift
import SwiftUI

@main
struct TimerApp: App {
    // Create DI container at app launch
    let container = DIContainer.make()

    // Create coordinator for navigation
    @StateObject private var coordinator = TimerCoordinator()

    var body: some Scene {
        WindowGroup {
            TimerListView(
                viewModel: TimerListViewModel(
                    getTimersUseCase: container.getTimersUseCase,
                    deleteTimerUseCase: container.deleteTimerUseCase,
                    coordinator: coordinator
                ),
                coordinator: coordinator
            )
        }
    }
}
```

---

## DI Container

### Container.swift

```swift
import Foundation
import Domain

// Single entry point for dependency injection
struct DIContainer {
    // Use Cases
    let getTimersUseCase: GetTimersUseCase
    let startTimerUseCase: StartTimerUseCase
    let pauseTimerUseCase: PauseTimerUseCase
    let resetTimerUseCase: ResetTimerUseCase
    let deleteTimerUseCase: DeleteTimerUseCase

    // Repositories (optional - expose if needed by UI)
    let timerRepository: TimerRepository

    // Factory method - creates fully wired container
    static func make() -> DIContainer {
        // 1. Create infrastructure (APIClient, etc.)
        let apiClient = APIClient()

        // 2. Create repositories (implement Domain protocols)
        let timerRepository = DefaultTimerRepository(apiClient: apiClient)

        // 3. Create use cases (implement Domain protocols, use repositories)
        let getTimersUseCase = DefaultGetTimersUseCase(repository: timerRepository)
        let startTimerUseCase = DefaultStartTimerUseCase(repository: timerRepository)
        let pauseTimerUseCase = DefaultPauseTimerUseCase(repository: timerRepository)
        let resetTimerUseCase = DefaultResetTimerUseCase(repository: timerRepository)
        let deleteTimerUseCase = DefaultDeleteTimerUseCase(repository: timerRepository)

        // 4. Return container with all wired dependencies
        return DIContainer(
            getTimersUseCase: getTimersUseCase,
            startTimerUseCase: startTimerUseCase,
            pauseTimerUseCase: pauseTimerUseCase,
            resetTimerUseCase: resetTimerUseCase,
            deleteTimerUseCase: deleteTimerUseCase,
            timerRepository: timerRepository
        )
    }
}
```

---

## Domain Layer

### Entity Example: Timer.swift

```swift
import Foundation

// Pure business data model - no imports except Foundation
struct Timer: Equatable, Identifiable {
    let id: UUID
    let name: String
    let duration: TimeInterval
    var remainingTime: TimeInterval
    var isRunning: Bool
    let createdAt: Date

    // Business logic methods
    mutating func start() {
        isRunning = true
    }

    mutating func pause() {
        isRunning = false
    }

    mutating func reset() {
        remainingTime = duration
        isRunning = false
    }

    mutating func tick(by delta: TimeInterval = 1.0) {
        if isRunning {
            remainingTime = max(0, remainingTime - delta)
            if remainingTime == 0 {
                isRunning = false
            }
        }
    }

    var isFinished: Bool {
        remainingTime <= 0
    }
}
```

### Use Case Protocol Example: GetTimersUseCase.swift

```swift
import Foundation

// Protocol defining the use case contract
protocol GetTimersUseCase {
    func execute() async throws -> [Timer]
}

// Implementation (Domain layer, but can use injected Repository protocol)
class DefaultGetTimersUseCase: GetTimersUseCase {
    let repository: TimerRepository

    init(repository: TimerRepository) {
        self.repository = repository
    }

    func execute() async throws -> [Timer] {
        // Fetch from repository
        let timers = try await repository.fetchAll()

        // Optional: Apply domain-level filtering/sorting
        return timers.sorted { $0.createdAt > $1.createdAt }
    }
}
```

### Repository Protocol Example: TimerRepository.swift

```swift
import Foundation

// Protocol for data access - no implementation in Domain
protocol TimerRepository {
    func fetchAll() async throws -> [Timer]
    func fetch(_ id: UUID) async throws -> Timer
    func save(_ timer: Timer) async throws -> Timer
    func delete(_ id: UUID) async throws
}
```

### Domain Error Example: DomainError.swift

```swift
import Foundation

enum DomainError: Error, LocalizedError {
    case timerNotFound
    case invalidDuration(message: String)
    case networkFailure
    case persistenceFailed(message: String)

    var errorDescription: String? {
        switch self {
        case .timerNotFound:
            return "Timer not found"
        case .invalidDuration(let message):
            return "Invalid duration: \(message)"
        case .networkFailure:
            return "Network request failed"
        case .persistenceFailed(let message):
            return "Failed to save: \(message)"
        }
    }

    var recoverySuggestion: String? {
        switch self {
        case .networkFailure:
            return "Check your internet connection and try again"
        case .persistenceFailed:
            return "Try again or restart the app"
        default:
            return nil
        }
    }
}
```

---

## Data Layer

### DTO Example: TimerDTO.swift

```swift
import Foundation

// Data Transfer Object - matches API response shape exactly
struct TimerDTO: Codable {
    let id: String
    let name: String
    let duration: Double
    let remainingTime: Double
    let isRunning: Bool
    let createdAt: String

    enum CodingKeys: String, CodingKey {
        case id, name, duration, remainingTime, isRunning
        case createdAt = "created_at"  // Map API field name
    }
}
```

### Mapper Example: TimerMapper.swift

```swift
import Foundation
import Domain

// Transforms between DTO (API data) and Entity (domain data)
struct TimerMapper {
    private static let dateFormatter: ISO8601DateFormatter = {
        let formatter = ISO8601DateFormatter()
        formatter.formatOptions = [.withInternetDateTime, .withFractionalSeconds]
        return formatter
    }()

    // DTO → Domain Entity
    static func toDomain(_ dto: TimerDTO) -> Timer {
        Timer(
            id: UUID(uuidString: dto.id) ?? UUID(),
            name: dto.name,
            duration: dto.duration,
            remainingTime: dto.remainingTime,
            isRunning: dto.isRunning,
            createdAt: dateFormatter.date(from: dto.createdAt) ?? Date()
        )
    }

    // Domain Entity → DTO
    static func toDTO(_ entity: Timer) -> TimerDTO {
        TimerDTO(
            id: entity.id.uuidString,
            name: entity.name,
            duration: entity.duration,
            remainingTime: entity.remainingTime,
            isRunning: entity.isRunning,
            createdAt: dateFormatter.string(from: entity.createdAt)
        )
    }

    // Array transformation helper
    static func toDomainArray(_ dtos: [TimerDTO]) -> [Timer] {
        dtos.map { toDomain($0) }
    }
}
```

### API Client Example: APIClient.swift

```swift
import Foundation

// Handles all network communication
class APIClient {
    private let baseURL: URL
    private let session: URLSession

    init(baseURL: URL = URL(string: "https://api.example.com")!, session: URLSession = .shared) {
        self.baseURL = baseURL
        self.session = session
    }

    // GET /timers
    func getTimers() async throws -> [TimerDTO] {
        let url = baseURL.appendingPathComponent("/timers")
        let request = URLRequest(url: url)
        return try await request(decodingTo: [TimerDTO].self)
    }

    // GET /timers/{id}
    func getTimer(_ id: String) async throws -> TimerDTO {
        let url = baseURL.appendingPathComponent("/timers/\(id)")
        let request = URLRequest(url: url)
        return try await request(decodingTo: TimerDTO.self)
    }

    // POST /timers
    func createTimer(_ dto: TimerDTO) async throws -> TimerDTO {
        let url = baseURL.appendingPathComponent("/timers")
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try JSONEncoder().encode(dto)
        return try await request(decodingTo: TimerDTO.self)
    }

    // DELETE /timers/{id}
    func deleteTimer(_ id: String) async throws {
        let url = baseURL.appendingPathComponent("/timers/\(id)")
        var request = URLRequest(url: url)
        request.httpMethod = "DELETE"
        let (_, response) = try await session.data(for: request)
        try validateHTTPResponse(response)
    }

    // MARK: - Helpers

    private func request<T: Decodable>(decodingTo type: T.Type, _ request: URLRequest) async throws -> T {
        let (data, response) = try await session.data(for: request)
        try validateHTTPResponse(response)
        return try JSONDecoder().decode(T.self, from: data)
    }

    private func validateHTTPResponse(_ response: URLResponse) throws {
        guard let httpResponse = response as? HTTPURLResponse else {
            throw URLError(.badServerResponse)
        }

        guard (200..<300).contains(httpResponse.statusCode) else {
            throw URLError(.badServerResponse)
        }
    }
}
```

### Repository Implementation Example: DefaultTimerRepository.swift

```swift
import Foundation
import Domain

// Implements Domain's TimerRepository protocol
class DefaultTimerRepository: TimerRepository {
    let apiClient: APIClient
    let userDefaults: UserDefaults

    init(apiClient: APIClient, userDefaults: UserDefaults = .standard) {
        self.apiClient = apiClient
        self.userDefaults = userDefaults
    }

    func fetchAll() async throws -> [Timer] {
        do {
            // Try to fetch from API
            let dtos = try await apiClient.getTimers()
            // Cache successful result
            cacheTimers(dtos)
            return TimerMapper.toDomainArray(dtos)
        } catch {
            // Fall back to cached data if network fails
            if let cachedDTOs = getCachedTimers() {
                return TimerMapper.toDomainArray(cachedDTOs)
            }
            throw DomainError.networkFailure
        }
    }

    func fetch(_ id: UUID) async throws -> Timer {
        let dto = try await apiClient.getTimer(id.uuidString)
        return TimerMapper.toDomain(dto)
    }

    func save(_ timer: Timer) async throws -> Timer {
        let dto = TimerMapper.toDTO(timer)
        let savedDTO = try await apiClient.createTimer(dto)
        let savedEntity = TimerMapper.toDomain(savedDTO)
        // Update cache
        var cached = getCachedTimers() ?? []
        cached.append(savedDTO)
        cacheTimers(cached)
        return savedEntity
    }

    func delete(_ id: UUID) async throws {
        try await apiClient.deleteTimer(id.uuidString)
        // Remove from cache
        if var cached = getCachedTimers() {
            cached.removeAll { $0.id == id.uuidString }
            cacheTimers(cached)
        }
    }

    // MARK: - Caching

    private func cacheTimers(_ dtos: [TimerDTO]) {
        if let data = try? JSONEncoder().encode(dtos) {
            userDefaults.set(data, forKey: "cached_timers")
        }
    }

    private func getCachedTimers() -> [TimerDTO]? {
        guard let data = userDefaults.data(forKey: "cached_timers") else { return nil }
        return try? JSONDecoder().decode([TimerDTO].self, from: data)
    }
}
```

---

## Presentation Layer

### ViewModel Example: TimerListViewModel.swift

```swift
import Foundation
import Combine
import Domain

// Observable object holding state and business logic for TimerListView
@MainActor
class TimerListViewModel: ObservableObject {
    @Published var state: ViewState = .loading
    @Published var searchText: String = ""

    enum ViewState {
        case loading
        case success([TimerViewModel])
        case empty
        case error(String)
    }

    private let getTimersUseCase: GetTimersUseCase
    private let deleteTimerUseCase: DeleteTimerUseCase
    let coordinator: TimerCoordinator

    init(
        getTimersUseCase: GetTimersUseCase,
        deleteTimerUseCase: DeleteTimerUseCase,
        coordinator: TimerCoordinator
    ) {
        self.getTimersUseCase = getTimersUseCase
        self.deleteTimerUseCase = deleteTimerUseCase
        self.coordinator = coordinator
    }

    // Public actions triggered by View
    func load() async {
        state = .loading
        do {
            let timers = try await getTimersUseCase.execute()
            let viewModels = timers.map { TimerViewModel($0) }
            state = viewModels.isEmpty ? .empty : .success(viewModels)
        } catch {
            state = .error(error.localizedDescription)
        }
    }

    func deleteTimer(_ id: UUID) async {
        do {
            try await deleteTimerUseCase.execute(timerId: id)
            // Reload after deletion
            await load()
        } catch {
            state = .error("Failed to delete: \(error.localizedDescription)")
        }
    }

    func refresh() async {
        await load()
    }

    func showTimerDetail(_ id: UUID) {
        coordinator.showDetail(id)
    }

    func showCreateTimer() {
        coordinator.showCreate()
    }
}

// Helper: wraps Timer for display
struct TimerViewModel: Identifiable {
    let timer: Timer

    var id: UUID { timer.id }
    var name: String { timer.name }
    var duration: TimeInterval { timer.duration }
    var remainingTime: TimeInterval { timer.remainingTime }
    var isRunning: Bool { timer.isRunning }
    var formattedTime: String {
        let minutes = Int(remainingTime) / 60
        let seconds = Int(remainingTime) % 60
        return String(format: "%02d:%02d", minutes, seconds)
    }
}
```

### Coordinator Example: TimerCoordinator.swift

```swift
import SwiftUI

// Navigation coordinator using NavigationPath
class TimerCoordinator: NSObject, ObservableObject {
    @Published var navigationPath = NavigationPath()

    // Navigation actions
    func showDetail(_ id: UUID) {
        navigationPath.append(TimerRoute.detail(id))
    }

    func showCreate() {
        navigationPath.append(TimerRoute.create)
    }

    func pop() {
        if !navigationPath.isEmpty {
            navigationPath.removeLast()
        }
    }

    func popToRoot() {
        navigationPath.removeLast(navigationPath.count)
    }
}

// Routes supported by TimerCoordinator
enum TimerRoute: Hashable {
    case detail(UUID)
    case create
}
```

### View Example: TimerListView.swift

```swift
import SwiftUI
import Domain

// Main timer list view - demonstrates all ViewState cases
struct TimerListView: View {
    @StateObject var viewModel: TimerListViewModel
    @ObservedObject var coordinator: TimerCoordinator

    var body: some View {
        NavigationStack(path: $coordinator.navigationPath) {
            ZStack {
                // Switch on state to show appropriate content
                switch viewModel.state {
                case .loading:
                    ProgressView()
                        .accessibilityLabel("Loading timers")

                case .success(let timers):
                    List {
                        ForEach(timers, id: \.id) { timer in
                            NavigationLink(value: TimerRoute.detail(timer.id)) {
                                TimerRowView(timer: timer)
                            }
                            .accessibilityIdentifier("timerRow_\(timer.id)")
                        }
                        .onDelete { indexSet in
                            for index in indexSet {
                                Task {
                                    await viewModel.deleteTimer(timers[index].id)
                                }
                            }
                        }
                    }
                    .listStyle(.plain)

                case .empty:
                    VStack(spacing: 16) {
                        Image(systemName: "hourglass")
                            .font(.system(size: 48))
                            .foregroundColor(.gray)
                        Text("No timers yet")
                            .font(.headline)
                        Button(action: { viewModel.showCreateTimer() }) {
                            Text("Create your first timer")
                                .frame(maxWidth: .infinity)
                                .padding()
                                .background(Color.blue)
                                .foregroundColor(.white)
                                .cornerRadius(8)
                        }
                        .padding()
                    }
                    .frame(maxWidth: .infinity, maxHeight: .infinity)
                    .background(Color(.systemGray6))

                case .error(let message):
                    VStack(spacing: 16) {
                        Image(systemName: "exclamationmark.triangle.fill")
                            .font(.system(size: 48))
                            .foregroundColor(.red)
                        Text("Error")
                            .font(.headline)
                        Text(message)
                            .font(.body)
                            .foregroundColor(.gray)
                        Button(action: { Task { await viewModel.load() } }) {
                            Text("Retry")
                                .frame(maxWidth: .infinity)
                                .padding()
                                .background(Color.blue)
                                .foregroundColor(.white)
                                .cornerRadius(8)
                        }
                        .padding()
                    }
                    .frame(maxWidth: .infinity, maxHeight: .infinity)
                    .background(Color(.systemGray6))
                }
            }
            .navigationTitle("Timers")
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Button(action: { viewModel.showCreateTimer() }) {
                        Image(systemName: "plus")
                    }
                    .accessibilityLabel("Create new timer")
                    .accessibilityIdentifier("createButton")
                }
            }
            .navigationDestination(for: TimerRoute.self) { route in
                switch route {
                case .detail(let id):
                    TimerDetailView(timerId: id, coordinator: coordinator)
                case .create:
                    CreateTimerView(coordinator: coordinator)
                }
            }
        }
        .task {
            await viewModel.load()
        }
    }
}

// Helper: renders a single timer row
struct TimerRowView: View {
    let timer: TimerViewModel

    var body: some View {
        HStack(spacing: 12) {
            VStack(alignment: .leading, spacing: 4) {
                Text(timer.name)
                    .font(.headline)
                Text(timer.formattedTime)
                    .font(.caption)
                    .foregroundColor(.gray)
            }

            Spacer()

            if timer.isRunning {
                Image(systemName: "play.circle.fill")
                    .foregroundColor(.green)
                    .font(.title3)
            } else {
                Image(systemName: "pause.circle")
                    .foregroundColor(.gray)
                    .font(.title3)
            }
        }
        .padding(.vertical, 8)
    }
}

#Preview {
    TimerListView(
        viewModel: TimerListViewModel(
            getTimersUseCase: PreviewGetTimersUseCase(),
            deleteTimerUseCase: PreviewDeleteTimerUseCase(),
            coordinator: TimerCoordinator()
        ),
        coordinator: TimerCoordinator()
    )
}
```

---

## Testing Helpers

### Mock Repository for Testing

```swift
import Domain

class TimerRepositoryMock: TimerRepository {
    var fetchAllCalled = false
    var fetchAllResult: [Timer] = []
    var fetchAllError: Error?

    var fetchCalled = false
    var fetchResult: Timer?
    var fetchError: Error?

    var saveCalled = false
    var saveResult: Timer?
    var saveError: Error?

    var deleteCalled = false
    var deleteError: Error?

    func fetchAll() async throws -> [Timer] {
        fetchAllCalled = true
        if let error = fetchAllError {
            throw error
        }
        return fetchAllResult
    }

    func fetch(_ id: UUID) async throws -> Timer {
        fetchCalled = true
        if let error = fetchError {
            throw error
        }
        return fetchResult ?? Timer(id: id, name: "Test", duration: 60, ...)
    }

    func save(_ timer: Timer) async throws -> Timer {
        saveCalled = true
        if let error = saveError {
            throw error
        }
        return saveResult ?? timer
    }

    func delete(_ id: UUID) async throws {
        deleteCalled = true
        if let error = deleteError {
            throw error
        }
    }
}
```

---

## See Also

- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-lite/references/ios-standards.md` — Standards and conventions
- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-lite/references/code-gen-phases.md` — How phases create these templates
