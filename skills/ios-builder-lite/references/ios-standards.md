# iOS Standards Reference

## Language & Framework Versions

- **Swift**: 5.9+
- **SwiftUI**: iOS 15+ minimum
- **Concurrency**: async/await (no callbacks or DispatchQueue)
- **No UIKit**: SwiftUI only

---

## Architecture: MVVM-C + Clean Architecture

The iOS app follows a 3-layer architecture with MVVM-C navigation pattern.

### Layer 1: Domain (Pure Business Logic)

**Location**: `Domain/` folder

**Responsibility**:
- Define business entities
- Define use case protocols
- Define repository protocols
- Define domain errors

**Import rules**: Foundation only. Zero framework dependencies.

**Files**:
- `Entities/` — Pure structs representing business data
- `UseCases/` — Protocols and implementations of business actions
- `Repositories/` — Protocols for data access (no implementation)
- `Errors/` — Domain error enums

**Example**:
```swift
// Domain/Entities/Timer.swift
import Foundation

struct Timer: Equatable {
    let id: UUID
    let name: String
    let duration: TimeInterval
    var remainingTime: TimeInterval
    var isRunning: Bool

    mutating func start() {
        isRunning = true
    }

    mutating func pause() {
        isRunning = false
    }
}

// Domain/UseCases/GetTimersUseCase.swift
protocol GetTimersUseCase {
    func execute() async throws -> [Timer]
}

class DefaultGetTimersUseCase: GetTimersUseCase {
    let repository: TimerRepository

    init(repository: TimerRepository) {
        self.repository = repository
    }

    func execute() async throws -> [Timer] {
        try await repository.fetchAll()
    }
}

// Domain/Repositories/TimerRepository.swift
protocol TimerRepository {
    func fetchAll() async throws -> [Timer]
    func save(_ timer: Timer) async throws -> Timer
    func delete(_ id: UUID) async throws
}

// Domain/Errors/DomainError.swift
enum DomainError: Error, LocalizedError {
    case timerNotFound
    case invalidDuration
    case networkFailure

    var errorDescription: String? {
        switch self {
        case .timerNotFound:
            return "Timer not found"
        case .invalidDuration:
            return "Invalid duration"
        case .networkFailure:
            return "Network request failed"
        }
    }
}
```

### Layer 2: Data (Persistence & Networking)

**Location**: `Data/` folder

**Responsibility**:
- Fetch data from APIs
- Persist data locally
- Map API data to domain entities
- Implement repository protocols

**Import rules**: Domain + Foundation + networking frameworks (URLSession, Codable). No SwiftUI, no UIKit.

**Files**:
- `DTOs/` — Data Transfer Objects (API response shapes)
- `Mappers/` — DTO ↔ Entity transformations
- `Repositories/` — Repository protocol implementations
- `Network/` — API clients and network logic
- `LocalStorage/` — UserDefaults, CoreData, file-based storage

**Example**:
```swift
// Data/DTOs/TimerDTO.swift
import Foundation

struct TimerDTO: Codable {
    let id: String
    let name: String
    let duration: Double
    let remainingTime: Double
    let isRunning: Bool
}

// Data/Mappers/TimerMapper.swift
import Foundation
import Domain

struct TimerMapper {
    static func toDomain(_ dto: TimerDTO) -> Timer {
        Timer(
            id: UUID(uuidString: dto.id) ?? UUID(),
            name: dto.name,
            duration: dto.duration,
            remainingTime: dto.remainingTime,
            isRunning: dto.isRunning
        )
    }

    static func toDTO(_ entity: Timer) -> TimerDTO {
        TimerDTO(
            id: entity.id.uuidString,
            name: entity.name,
            duration: entity.duration,
            remainingTime: entity.remainingTime,
            isRunning: entity.isRunning
        )
    }
}

// Data/Network/APIClient.swift
import Foundation

class APIClient {
    private let baseURL = URL(string: "https://api.example.com")!

    func getTimers() async throws -> [TimerDTO] {
        let url = baseURL.appendingPathComponent("/timers")
        let (data, response) = try await URLSession.shared.data(from: url)

        guard let httpResponse = response as? HTTPURLResponse, httpResponse.statusCode == 200 else {
            throw NSError(domain: "HTTP", code: -1)
        }

        return try JSONDecoder().decode([TimerDTO].self, from: data)
    }
}

// Data/Repositories/DefaultTimerRepository.swift
import Foundation
import Domain

class DefaultTimerRepository: TimerRepository {
    let apiClient: APIClient

    init(apiClient: APIClient) {
        self.apiClient = apiClient
    }

    func fetchAll() async throws -> [Timer] {
        let dtos = try await apiClient.getTimers()
        return dtos.map { TimerMapper.toDomain($0) }
    }

    func save(_ timer: Timer) async throws -> Timer {
        // POST to API
        timer
    }

    func delete(_ id: UUID) async throws {
        // DELETE from API
    }
}
```

### Layer 3: Presentation (UI & Navigation)

**Location**: `Presentation/` folder

**Responsibility**:
- SwiftUI views and layouts
- ViewModels with state management
- Navigation and routing
- User interaction handling

**Import rules**: Domain + SwiftUI + Combine. No Data layer imports. No UIKit.

**Files**:
- `Views/` — SwiftUI View structs
- `ViewModels/` — ObservableObject state machines
- `Coordinators/` — MVVM-C navigation coordinators

**Example**:
```swift
// Presentation/ViewModels/TimerListViewModel.swift
import Foundation
import Combine
import Domain

@MainActor
class TimerListViewModel: ObservableObject {
    @Published var state: ViewState = .loading

    enum ViewState {
        case loading
        case success([Timer])
        case empty
        case error(String)
    }

    let getTimersUseCase: GetTimersUseCase
    let coordinator: TimerCoordinator

    init(
        getTimersUseCase: GetTimersUseCase,
        coordinator: TimerCoordinator
    ) {
        self.getTimersUseCase = getTimersUseCase
        self.coordinator = coordinator
    }

    func loadTimers() async {
        state = .loading
        do {
            let timers = try await getTimersUseCase.execute()
            state = timers.isEmpty ? .empty : .success(timers)
        } catch {
            state = .error(error.localizedDescription)
        }
    }

    func showDetail(_ id: UUID) {
        coordinator.showDetail(id)
    }
}

// Presentation/Coordinators/TimerCoordinator.swift
import SwiftUI

class TimerCoordinator: NSObject, ObservableObject {
    @Published var navigationPath = NavigationPath()

    func showDetail(_ id: UUID) {
        navigationPath.append(TimerRoute.detail(id))
    }

    func pop() {
        navigationPath.removeLast()
    }
}

enum TimerRoute: Hashable {
    case detail(UUID)
    case create
}

// Presentation/Views/TimerListView.swift
import SwiftUI
import Domain

struct TimerListView: View {
    @StateObject var viewModel: TimerListViewModel
    @ObservedObject var coordinator: TimerCoordinator

    var body: some View {
        NavigationStack(path: $coordinator.navigationPath) {
            ZStack {
                switch viewModel.state {
                case .loading:
                    ProgressView()

                case .success(let timers):
                    List {
                        ForEach(timers, id: \.id) { timer in
                            NavigationLink(value: TimerRoute.detail(timer.id)) {
                                VStack(alignment: .leading) {
                                    Text(timer.name)
                                        .font(.headline)
                                    Text("\(Int(timer.remainingTime))s")
                                        .font(.caption)
                                        .foregroundColor(.gray)
                                }
                            }
                        }
                    }

                case .empty:
                    VStack {
                        Text("No timers yet")
                        Button("Create one") {
                            viewModel.showDetail(UUID())
                        }
                    }

                case .error(let message):
                    VStack {
                        Text("Error: \(message)")
                            .foregroundColor(.red)
                        Button("Retry") {
                            Task { await viewModel.loadTimers() }
                        }
                    }
                }
            }
            .navigationTitle("Timers")
            .navigationDestination(for: TimerRoute.self) { route in
                switch route {
                case .detail(let id):
                    TimerDetailView(timerId: id, coordinator: coordinator)
                case .create:
                    Text("Create Timer")
                }
            }
            .task {
                await viewModel.loadTimers()
            }
        }
    }
}
```

---

## Dependency Injection: Protocol-Driven Manual

No DI frameworks. Explicit manual wiring.

**Pattern**:
1. Domain defines protocols (UseCase, Repository)
2. Data implements protocols (DefaultRepository)
3. Presentation depends on protocols, not implementations
4. App container wires everything at startup

**Container Example**:
```swift
// App/DI/Container.swift
import Foundation
import Domain

struct DIContainer {
    let getTimersUseCase: GetTimersUseCase
    let startTimerUseCase: StartTimerUseCase

    static func make() -> DIContainer {
        let apiClient = APIClient()
        let repository = DefaultTimerRepository(apiClient: apiClient)

        let getTimersUseCase = DefaultGetTimersUseCase(repository: repository)
        let startTimerUseCase = DefaultStartTimerUseCase(repository: repository)

        return DIContainer(
            getTimersUseCase: getTimersUseCase,
            startTimerUseCase: startTimerUseCase
        )
    }
}

// App/main.swift
@main
struct TimerApp: App {
    let container = DIContainer.make()

    var body: some Scene {
        WindowGroup {
            TimerListView(
                viewModel: TimerListViewModel(
                    getTimersUseCase: container.getTimersUseCase,
                    coordinator: TimerCoordinator()
                ),
                coordinator: TimerCoordinator()
            )
        }
    }
}
```

---

## Navigation: MVVM-C with NavigationPath

**Pattern**: Coordinator holds `@Published var navigationPath = NavigationPath()`. ViewModel asks coordinator to navigate. NavigationStack listens to navigationPath.

**Routes Enum**:
```swift
enum TimerRoute: Hashable {
    case detail(UUID)
    case create
}
```

**Coordinator**:
```swift
class TimerCoordinator: NSObject, ObservableObject {
    @Published var navigationPath = NavigationPath()

    func showDetail(_ id: UUID) {
        navigationPath.append(TimerRoute.detail(id))
    }

    func showCreate() {
        navigationPath.append(TimerRoute.create)
    }

    func pop() {
        navigationPath.removeLast()
    }
}
```

**View Integration**:
```swift
NavigationStack(path: $coordinator.navigationPath) {
    // Root content
    .navigationDestination(for: TimerRoute.self) { route in
        switch route {
        case .detail(let id):
            TimerDetailView(id: id)
        case .create:
            CreateTimerView()
        }
    }
}
```

---

## State Management: ViewState Enums + @Published

**Pattern**: ViewModel has `@Published var state: ViewState`. View switches on state and renders accordingly.

**ViewState Design**:
```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var state: ViewState = .loading

    enum ViewState {
        case loading
        case success(Data)
        case empty
        case error(String)
    }

    func load() async {
        state = .loading
        do {
            let data = try await fetch()
            state = data.isEmpty ? .empty : .success(data)
        } catch {
            state = .error(error.localizedDescription)
        }
    }
}
```

**View Pattern**:
```swift
switch viewModel.state {
case .loading:
    ProgressView()

case .success(let data):
    List(data) { item in
        ItemRow(item: item)
    }

case .empty:
    Text("No data")

case .error(let message):
    VStack {
        Text("Error: \(message)")
        Button("Retry") { Task { await viewModel.load() } }
    }
}
```

**Benefits**:
- Exhaustive switch (compiler catches missing cases)
- Only one state at a time (no loading + success simultaneously)
- Clear state transitions

---

## Naming Conventions

### Types (PascalCase)
- `Timer`, `TimerViewModel`, `TimerRepository`
- `GetTimersUseCase`, `StartTimerUseCase`
- `TimerListView`, `TimerDetailView`
- `TimerCoordinator`

### Methods & Properties (camelCase)
- `loadTimers()`, `startTimer()`, `deleteTimer()`
- `remainingTime`, `isRunning`, `createdAt`

### Use Cases
- Format: `VerbNounUseCase`
- Examples: `GetTimersUseCase`, `StartTimerUseCase`, `DeleteTimerUseCase`

### ViewModels
- Format: `ScreenNameViewModel`
- Examples: `TimerListViewModel`, `TimerDetailViewModel`

### Repositories
- Format: `NounRepository` (protocol) or `DefaultNounRepository` (impl)
- Examples: `TimerRepository`, `DefaultTimerRepository`

### Views
- Format: `ScreenNameView`
- Examples: `TimerListView`, `TimerDetailView`, `TimerRowView`

### Coordinators
- Format: `FlowNameCoordinator`
- Examples: `TimerCoordinator`, `AuthCoordinator`

---

## File Structure

Canonical directory layout:

```
App.xcodeproj
├── App/
│   ├── DI/
│   │   └── Container.swift (dependency injection container)
│   └── main.swift (@main entry point)
│
├── Domain/
│   ├── Entities/
│   │   ├── Timer.swift
│   │   └── User.swift
│   ├── UseCases/
│   │   ├── GetTimersUseCase.swift (protocol + impl)
│   │   ├── StartTimerUseCase.swift
│   │   └── DeleteTimerUseCase.swift
│   ├── Repositories/
│   │   └── TimerRepository.swift (protocol only)
│   └── Errors/
│       └── DomainError.swift
│
├── Data/
│   ├── DTOs/
│   │   └── TimerDTO.swift
│   ├── Mappers/
│   │   └── TimerMapper.swift
│   ├── Repositories/
│   │   └── DefaultTimerRepository.swift (impl)
│   └── Network/
│       ├── APIClient.swift
│       └── URLSessionConfiguration.swift
│
└── Presentation/
    ├── Views/
    │   ├── TimerListView.swift
    │   ├── TimerDetailView.swift
    │   ├── TimerRowView.swift
    │   └── CreateTimerView.swift
    ├── ViewModels/
    │   ├── TimerListViewModel.swift
    │   ├── TimerDetailViewModel.swift
    │   └── CreateTimerViewModel.swift
    └── Coordinators/
        └── TimerCoordinator.swift
```

---

## Layer Import Rules

### Domain Layer
```swift
import Foundation  // ✅ Only

// ❌ NOT ALLOWED
import UIKit
import SwiftUI
import Combine
import Domain
```

### Data Layer
```swift
import Foundation  // ✅
import Domain      // ✅
// Plus: URLSession, Codable, SQLite, etc.

// ❌ NOT ALLOWED
import SwiftUI
import UIKit
```

### Presentation Layer
```swift
import SwiftUI    // ✅
import Combine    // ✅
import Foundation // ✅
import Domain     // ✅

// ❌ NOT ALLOWED
import Data       // Use Domain protocols only
import UIKit
```

### App Layer
```swift
import Foundation  // ✅
import Domain      // ✅
import Data        // ✅ (for DI)
import Presentation // ✅ (for root view)
```

---

## Async/Await Patterns

**Never use DispatchQueue or Combine operators**. Always use async/await.

**ViewModel loading**:
```swift
@MainActor
class ViewModel: ObservableObject {
    func load() async {
        state = .loading
        do {
            let data = try await useCase.execute()
            state = .success(data)
        } catch {
            state = .error(error.localizedDescription)
        }
    }
}
```

**View task**:
```swift
.task {
    await viewModel.load()
}
```

**No Combine operators**:
```swift
// ❌ Don't do this
$searchText
    .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
    .flatMap { ... }

// ✅ Do this
searchText.onSubmit {
    Task { await search() }
}
```

---

## Code Quality

### SwiftLint Rules

**Enabled**:
- No force unwrap (!) in code (constants allowed)
- No implicitly unwrapped optionals (!)
- Max line length: 120 characters
- Proper naming (camelCase methods, PascalCase types)
- No TODO without assignee
- No empty catch blocks

**Example .swiftlint.yml**:
```yaml
rules:
  - force_unwrap
  - implicitly_unwrapped_optional
  - line_length
  - naming
  - todos

line_length:
  max_length: 120

disabled_rules:
  - trailing_comma  # SwiftFormat handles this
```

### SwiftFormat Rules

**Indentation**: 4 spaces
**Trailing commas**: Add in multiline collections
**Blank lines**: Between function definitions
**Brackets**: Opening bracket on same line

**Example formatting**:
```swift
// ✅ Correct
struct Timer {
    let id: UUID
    let name: String
    let duration: TimeInterval

    func start() {
        // ...
    }

    func pause() {
        // ...
    }
}

// ❌ Incorrect
struct Timer
{
  let id: UUID
  let name: String
  let duration: TimeInterval
  func start() { }
  func pause() { }
}
```

---

## Testing

### Unit Testing Pattern

```swift
// TimerRepositoryMock.swift
class TimerRepositoryMock: TimerRepository {
    var fetchAllCalled = false
    var fetchAllResult: [Timer] = []
    var fetchAllError: Error?

    func fetchAll() async throws -> [Timer] {
        fetchAllCalled = true
        if let error = fetchAllError {
            throw error
        }
        return fetchAllResult
    }
    // ...
}

// GetTimersUseCaseTests.swift
class GetTimersUseCaseTests: XCTestCase {
    func test_execute_returnsTimersFromRepository() async {
        // Given
        let mock = TimerRepositoryMock()
        mock.fetchAllResult = [Timer(id: UUID(), name: "Test", duration: 60, ...)]
        let useCase = DefaultGetTimersUseCase(repository: mock)

        // When
        let result = try await useCase.execute()

        // Then
        XCTAssertTrue(mock.fetchAllCalled)
        XCTAssertEqual(result.count, 1)
        XCTAssertEqual(result[0].name, "Test")
    }
}
```

### Test Naming Convention

`test_<scenario>_<expectedResult>`

- `test_loadTimers_returnsEmptyList`
- `test_deleteTimer_callsRepository`
- `test_load_showsErrorState`

---

## Accessibility

**Always include**:
- `.accessibilityLabel()` on interactive elements
- `.accessibilityIdentifier()` for testing
- `.accessibilityHint()` for complex actions

**Example**:
```swift
Button(action: { /* delete */ }) {
    Image(systemName: "trash")
}
.accessibilityLabel("Delete timer")
.accessibilityIdentifier("deleteButton")
.accessibilityHint("Permanently removes the timer")
```

---

## See Also

- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-builder-lite/references/code-gen-phases.md` — How standards are applied in each phase
- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-builder-lite/references/ios-architecture-template.md` — Code templates
