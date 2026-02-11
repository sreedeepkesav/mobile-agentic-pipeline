# 4-Phase Coder for iOS Lite

## Overview

The Coder is a team of AI specialists that generates your entire iOS app in 4 sequential phases. Each phase has a specific focus and builds on the outputs of prior phases.

**Why 4 phases?** AI writes better code when it thinks in layers. Each phase can focus on one concern (architecture, domain logic, data, UI) without getting distracted by others.

**Total duration**: ~15-30 minutes depending on feature complexity

**Inputs**: Spec.md, screens.md, design-tokens.json, edge-cases.md (all from Product+Design)

**Output**: Complete Swift/SwiftUI project ready to build

---

## Phase 1: Architect (Principal Level)

The Architect is a senior-level AI that reads the spec and designs the entire solution architecture before any code is written.

### Input
- Feature spec with acceptance criteria
- Screen list and navigation flows
- API contracts
- Design tokens
- iOS standards reference (MVVM-C + Clean Architecture)

### Task

**Read & Understand**:
- What problem are we solving?
- What screens are involved?
- What data do we need?
- What APIs do we call?
- What edge cases must we handle?

**Research Swift/iOS Best Practices**:
- How do other apps structure similar features?
- What Swift patterns are recommended?
- What SwiftUI approaches work best?
- Error handling patterns

**Design Protocol Contracts**:
- What Use Case protocols do we need?
- What Repository protocols for data access?
- What domain errors should we define?
- Example:
  ```swift
  // Domain layer contracts (defined now, implemented in Phase 2 & 3)
  protocol GetTimersUseCase {
      func execute() async throws -> [Timer]
  }

  protocol TimerRepository {
      func fetchAll() async throws -> [Timer]
      func save(_ timer: Timer) async throws
      func delete(_ id: UUID) async throws
  }

  enum DomainError: Error {
      case timerNotFound
      case invalidDuration
      case networkFailure
  }
  ```

**Plan Dependency Graph**:
- What components depend on what?
- How does data flow from repository → use case → view model → view?
- What gets injected at app startup?
- Draw it out:
  ```
  View (SwiftUI)
    ↓
  ViewModel (ObservableObject)
    ↓
  UseCase (protocol + impl)
    ↓
  Repository (protocol + impl)
    ↓
  APIClient (network) + LocalStorage (persistence)
  ```

**Decide File Structure**:
- What files go in Domain/?
- What files go in Data/?
- What files go in Presentation/?
- What goes in App/?
- Example (from plan, not created yet):
  ```
  App/
  ├── DI/
  │   └── Container.swift
  └── main.swift

  Domain/
  ├── Entities/
  │   └── Timer.swift
  ├── UseCases/
  │   ├── GetTimersUseCase.swift
  │   └── StartTimerUseCase.swift
  ├── Repositories/
  │   └── TimerRepository.swift
  └── Errors/
      └── DomainError.swift

  Data/
  ├── DTOs/
  │   └── TimerDTO.swift
  ├── Mappers/
  │   └── TimerMapper.swift
  ├── Repositories/
  │   └── DefaultTimerRepository.swift
  └── Network/
      └── APIClient.swift

  Presentation/
  ├── Views/
  │   ├── TimerListView.swift
  │   └── TimerDetailView.swift
  ├── ViewModels/
  │   ├── TimerListViewModel.swift
  │   └── TimerDetailViewModel.swift
  └── Coordinators/
      └── TimerCoordinator.swift
  ```

**Design Navigation with Coordinators**:
- Which Coordinator class (or classes)?
- What NavigationPath routes?
- How does ViewModel tell Coordinator to navigate?
- Example:
  ```swift
  // Navigation routes
  enum TimerRoute: Hashable {
      case detail(UUID)
      case create
  }

  // Coordinator holds navigation state
  class TimerCoordinator: NSObject, ObservableObject {
      @Published var navigationPath = NavigationPath()

      func showDetail(_ id: UUID) {
          navigationPath.append(TimerRoute.detail(id))
      }
  }
  ```

**Plan Error Handling**:
- What domain errors are possible?
- How do we map API errors to domain errors?
- How does UI display errors to user?
- What's the retry logic?

**Write Architecture Decision Record (ADR)**:
A text file explaining all decisions made:

```markdown
# Architecture Decision Record: Timer Feature

## Decision: Use MVVM-C with Clean Architecture

### Context
We need to add a timer feature with list, detail, create, and delete screens.

### Decision
Structure as 3-layer (Domain, Data, Presentation) with MVVM-C navigation using NavigationPath.

### Rationale
- Domain layer stays pure Swift (testable, framework-agnostic)
- Data layer handles all I/O (network, local storage)
- Presentation layer owns UI and navigation
- Coordinator pattern provides clear navigation contracts

### Consequences
- More files (protocol definitions, implementations, view models)
- Clear layer boundaries (no cross-layer imports)
- Easier to test domain logic (mock repositories)
- Reusable UseCase protocols

## Decision: Protocol-Based Dependency Injection

### Context
Lite pipeline has no DI framework (Swinject, Factory, etc.).

### Decision
Manual DI: Domain defines protocols, Data implements, App creates container at startup.

### Rationale
- Zero dependencies
- Clear to read (explicit wiring)
- No runtime magic or reflection

### Consequences
- More boilerplate in Container.swift
- Easier to debug (no hidden initializers)

## Decision: NavigationPath for Routing

### Context
Need to route between Timer List → Timer Detail → Create Timer.

### Decision
Use NavigationPath with Coordinator holding state.

### Rationale
- Native SwiftUI approach
- Works with @StateObject and @Published
- No custom navigation frameworks

### Consequences
- Limited deep linking (NavigationPath doesn't support URLs easily)
- Works well for simple routing (which this feature is)

## Decision: ViewState Enum for Loading/Error/Success

### Context
Need to handle loading, success, error, and empty states.

### Decision
Use ViewState enum in ViewModel, switch on it in View.

### Rationale
- Exhaustive switch (compiler catches missing cases)
- Clear state machine (only one state at a time)
- ViewModel controls state, View only displays

### Consequences
- More boilerplate (switch statements)
- Safer code (no showing both loading + error simultaneously)

## File Ownership
- Architect: Defines what goes where
- Domain Lead: Implements Domain/ files
- Data Lead: Implements Data/ files
- Presentation Lead: Implements Presentation/ files
- Integration: Wires DI container and App/
```

### Output

**ADR file** (text, not code):
- Architecture decisions
- Protocol contracts (Use Cases, Repositories, domain errors)
- Dependency graph diagram
- Navigation plan
- Error handling strategy
- File structure tree

### CRITICAL: No Code Yet

The Architect defines what classes/protocols exist and how they relate. It does NOT write code. Phase 2 and 3 implement these protocols.

---

## Phase 2: Domain Lead (Senior Level)

The Domain Lead implements the Domain layer: pure Swift, zero framework imports, focusing on business logic.

### Input
- ADR from Phase 1
- Feature spec
- iOS standards reference

### Task

**Create Entity Structs**:
- Pure Swift structs (no Codable, no framework imports)
- Represent core business data
- Use value semantics (struct, not class)

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
    let createdAt: Date

    func start() -> Timer {
        var updated = self
        updated.isRunning = true
        return updated
    }

    func pause() -> Timer {
        var updated = self
        updated.isRunning = false
        return updated
    }
}
```

**Define Use Case Protocols**:
- Describe what actions the feature can perform
- Business logic contracts
- What throws, what returns
- Example:
```swift
// Domain/UseCases/GetTimersUseCase.swift
protocol GetTimersUseCase {
    func execute() async throws -> [Timer]
}

// Domain/UseCases/StartTimerUseCase.swift
protocol StartTimerUseCase {
    func execute(timerId: UUID) async throws -> Timer
}

// Domain/UseCases/DeleteTimerUseCase.swift
protocol DeleteTimerUseCase {
    func execute(timerId: UUID) async throws
}
```

**Implement Use Cases**:
- Implement each Use Case protocol
- Pure business logic (no UI, no network calls)
- Use injected Repository protocols (defined in Phase 3, implemented in Data)
- Example:
```swift
// Domain/UseCases/DefaultGetTimersUseCase.swift
class DefaultGetTimersUseCase: GetTimersUseCase {
    let repository: TimerRepository

    init(repository: TimerRepository) {
        self.repository = repository
    }

    func execute() async throws -> [Timer] {
        let timers = try await repository.fetchAll()
        // Could add business logic here (e.g., sort, filter)
        return timers.sorted { $0.createdAt > $1.createdAt }
    }
}
```

**Define Repository Protocols**:
- Abstractions for data sources (network, local storage)
- Domain layer doesn't know how data is fetched
- Example:
```swift
// Domain/Repositories/TimerRepository.swift
protocol TimerRepository {
    func fetchAll() async throws -> [Timer]
    func fetch(_ id: UUID) async throws -> Timer
    func save(_ timer: Timer) async throws -> Timer
    func delete(_ id: UUID) async throws
}
```

**Define Domain Errors**:
- Errors that make sense to business logic
- Example:
```swift
// Domain/Errors/DomainError.swift
enum DomainError: Error, LocalizedError {
    case timerNotFound
    case invalidDuration(message: String)
    case operationFailed(message: String)

    var errorDescription: String? {
        switch self {
        case .timerNotFound:
            return "Timer not found"
        case .invalidDuration(let msg):
            return "Invalid duration: \(msg)"
        case .operationFailed(let msg):
            return "Operation failed: \(msg)"
        }
    }
}
```

### Output

**Swift files** in Domain/:
- `Domain/Entities/Timer.swift` (and other entities)
- `Domain/UseCases/GetTimersUseCase.swift` (protocol + implementation)
- `Domain/UseCases/StartTimerUseCase.swift` (and others)
- `Domain/Repositories/TimerRepository.swift` (protocol only, no impl)
- `Domain/Errors/DomainError.swift`

### CRITICAL: Zero Imports Except Foundation

```swift
// ✅ OK
import Foundation

// ❌ NOT OK
import UIKit
import SwiftUI
import Combine
import Alamofire
```

Domain is pure business logic. No framework dependencies.

---

## Phase 3: Parallel Development

### Phase 3a: Data Lead (Senior Level)

The Data Lead implements data fetching, storage, and mapping.

#### Input
- ADR from Phase 1
- Domain code from Phase 2
- API contracts from spec

#### Task

**Create DTOs (Data Transfer Objects)**:
- Structures matching API response shape
- Codable for JSON deserialization
- Example:
```swift
// Data/DTOs/TimerDTO.swift
import Foundation

struct TimerDTO: Codable {
    let id: String
    let name: String
    let duration: Double
    let remainingTime: Double
    let isRunning: Bool
    let createdAt: String

    enum CodingKeys: String, CodingKey {
        case id, name, duration, remainingTime, isRunning, createdAt = "created_at"
    }
}
```

**Create Mappers**:
- Transform DTO ↔ Entity
- Handle type conversions (String UUID, Double TimeInterval, String Date)
- Example:
```swift
// Data/Mappers/TimerMapper.swift
import Foundation

struct TimerMapper {
    static func toDomain(_ dto: TimerDTO) -> Timer {
        Timer(
            id: UUID(uuidString: dto.id) ?? UUID(),
            name: dto.name,
            duration: dto.duration,
            remainingTime: dto.remainingTime,
            isRunning: dto.isRunning,
            createdAt: ISO8601DateFormatter().date(from: dto.createdAt) ?? Date()
        )
    }

    static func toDTO(_ entity: Timer) -> TimerDTO {
        TimerDTO(
            id: entity.id.uuidString,
            name: entity.name,
            duration: entity.duration,
            remainingTime: entity.remainingTime,
            isRunning: entity.isRunning,
            createdAt: ISO8601DateFormatter().string(from: entity.createdAt)
        )
    }
}
```

**Implement APIClient**:
- Network requests to backend
- Handles encoding/decoding
- Maps errors to domain errors
- Example:
```swift
// Data/Network/APIClient.swift
import Foundation

class APIClient {
    private let baseURL = URL(string: "https://api.example.com")!

    func getTimers() async throws -> [TimerDTO] {
        let url = baseURL.appendingPathComponent("/timers")
        let (data, _) = try await URLSession.shared.data(from: url)
        let response = try JSONDecoder().decode([TimerDTO].self, from: data)
        return response
    }

    func saveTimer(_ dto: TimerDTO) async throws -> TimerDTO {
        let url = baseURL.appendingPathComponent("/timers")
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try JSONEncoder().encode(dto)

        let (data, _) = try await URLSession.shared.data(for: request)
        return try JSONDecoder().decode(TimerDTO.self, from: data)
    }
}
```

**Implement Repository Protocols**:
- Conform to Domain repository protocols
- Use APIClient and local storage
- Map DTOs to Entities
- Example:
```swift
// Data/Repositories/DefaultTimerRepository.swift
import Foundation

class DefaultTimerRepository: TimerRepository {
    let apiClient: APIClient
    let userDefaults: UserDefaults

    init(apiClient: APIClient, userDefaults: UserDefaults = .standard) {
        self.apiClient = apiClient
        self.userDefaults = userDefaults
    }

    func fetchAll() async throws -> [Timer] {
        do {
            let dtos = try await apiClient.getTimers()
            return dtos.map { TimerMapper.toDomain($0) }
        } catch {
            // Fallback to cached data
            return getCachedTimers()
        }
    }

    func save(_ timer: Timer) async throws -> Timer {
        let dto = TimerMapper.toDTO(timer)
        let savedDTO = try await apiClient.saveTimer(dto)
        let result = TimerMapper.toDomain(savedDTO)
        cacheTimer(result)
        return result
    }

    private func getCachedTimers() -> [Timer] {
        // Read from UserDefaults
        []
    }

    private func cacheTimer(_ timer: Timer) {
        // Save to UserDefaults
    }
}
```

#### Output

**Swift files** in Data/:
- `Data/DTOs/TimerDTO.swift` (and other DTOs)
- `Data/Mappers/TimerMapper.swift`
- `Data/Network/APIClient.swift`
- `Data/Repositories/DefaultTimerRepository.swift` (implements Domain protocols)
- `Data/LocalStorage/TimerCache.swift` (if using persistent storage)

#### CRITICAL: Imports Only Domain + Foundation

```swift
import Foundation
import Domain  // ✅ Import Domain to implement protocols
import UIKit   // ❌ Not allowed
import SwiftUI // ❌ Not allowed
```

---

### Phase 3b: Presentation Lead (Parallel with Data Lead)

The Presentation Lead implements ViewModels, Views, and Coordinators.

#### Input
- ADR from Phase 1
- Domain code from Phase 2
- Design tokens from spec
- Screen descriptions from spec

#### Task

**Create ViewModels**:
- ObservableObject with @Published state
- ViewState enum (loading, success, error, empty)
- Async load functions
- Delegate user interactions to UseCases

**Example**:
```swift
// Presentation/ViewModels/TimerListViewModel.swift
import Foundation
import Combine

@MainActor
class TimerListViewModel: ObservableObject {
    @Published var state: ViewState = .loading

    enum ViewState {
        case loading
        case success([TimerViewModel])
        case empty
        case error(String)
    }

    let getTimersUseCase: GetTimersUseCase
    let deleteTimerUseCase: DeleteTimerUseCase
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

    func loadTimers() async {
        state = .loading
        do {
            let timers = try await getTimersUseCase.execute()
            if timers.isEmpty {
                state = .empty
            } else {
                state = .success(timers.map { TimerViewModel($0) })
            }
        } catch {
            state = .error(error.localizedDescription)
        }
    }

    func deleteTimer(_ id: UUID) async {
        do {
            try await deleteTimerUseCase.execute(timerId: id)
            await loadTimers()  // Reload list
        } catch {
            state = .error("Delete failed: \(error.localizedDescription)")
        }
    }

    func showDetail(_ id: UUID) {
        coordinator.showDetail(id)
    }
}
```

**Create Coordinators**:
- Holds NavigationPath
- Provides navigation methods
- Tied to ViewController/View

**Example**:
```swift
// Presentation/Coordinators/TimerCoordinator.swift
import Foundation
import SwiftUI

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

enum TimerRoute: Hashable {
    case detail(UUID)
    case create
}
```

**Create SwiftUI Views**:
- Switch on ViewState
- Render UI per design tokens
- Handle user interactions

**Example**:
```swift
// Presentation/Views/TimerListView.swift
import SwiftUI

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
                        ForEach(timers) { timer in
                            NavigationLink(value: TimerRoute.detail(timer.id)) {
                                TimerRowView(timer: timer)
                            }
                        }
                        .onDelete { indexSet in
                            for index in indexSet {
                                Task {
                                    await viewModel.deleteTimer(timers[index].id)
                                }
                            }
                        }
                    }

                case .empty:
                    VStack {
                        Text("No timers yet")
                            .font(.body)
                        Button("Create one") {
                            viewModel.showCreate()
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
                    CreateTimerView(coordinator: coordinator)
                }
            }
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Button(action: { viewModel.showCreate() }) {
                        Image(systemName: "plus")
                    }
                }
            }
        }
        .task {
            await viewModel.loadTimers()
        }
    }
}

struct TimerRowView: View {
    let timer: TimerViewModel

    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(timer.name)
                    .font(.headline)
                Text("\(Int(timer.remainingTime)) seconds")
                    .font(.caption)
                    .foregroundColor(.gray)
            }
            Spacer()
            if timer.isRunning {
                Image(systemName: "play.circle.fill")
                    .foregroundColor(.green)
            } else {
                Image(systemName: "pause.circle")
                    .foregroundColor(.gray)
            }
        }
    }
}
```

#### Output

**Swift files** in Presentation/:
- `Presentation/ViewModels/TimerListViewModel.swift` (and others)
- `Presentation/Views/TimerListView.swift` (and other views)
- `Presentation/Coordinators/TimerCoordinator.swift`

#### CRITICAL: Imports Only Domain + SwiftUI

```swift
import SwiftUI
import Combine
import Foundation
import Domain  // ✅ OK to import Domain

import Data    // ❌ NOT ALLOWED - don't import Data layer directly
import UIKit   // ❌ Not allowed
```

---

## Phase 4: Integration (Staff Level)

The Integration specialist wires everything together and audits the result.

### Input
- All code from Phases 1-3
- iOS standards reference

### Task

**Create DI Container**:
- Instantiate all repositories
- Instantiate all use cases
- Make container available to App

**Example**:
```swift
// App/DI/Container.swift
import Foundation
import Domain

struct DIContainer {
    let timerRepository: TimerRepository
    let getTimersUseCase: GetTimersUseCase
    let startTimerUseCase: StartTimerUseCase
    let deleteTimerUseCase: DeleteTimerUseCase

    static func make() -> DIContainer {
        let apiClient = APIClient()
        let repository = DefaultTimerRepository(apiClient: apiClient)

        let getTimersUseCase = DefaultGetTimersUseCase(repository: repository)
        let startTimerUseCase = DefaultStartTimerUseCase(repository: repository)
        let deleteTimerUseCase = DefaultDeleteTimerUseCase(repository: repository)

        return DIContainer(
            timerRepository: repository,
            getTimersUseCase: getTimersUseCase,
            startTimerUseCase: startTimerUseCase,
            deleteTimerUseCase: deleteTimerUseCase
        )
    }
}
```

**Create App Entry Point**:
- @main struct
- Create container
- Create root view with injected dependencies

**Example**:
```swift
// App/main.swift
import SwiftUI

@main
struct TimerApp: App {
    let container = DIContainer.make()
    @StateObject var coordinator = TimerCoordinator()

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

**Run SwiftLint**:
- Check code style violations
- Auto-fixable violations (via SwiftFormat):
  - Trailing whitespace
  - Indentation
  - Formatting
- Non-auto-fixable violations (flag for PR):
  - Force unwraps (!)
  - Implicitly unwrapped optionals (?)
  - Long methods
  - High complexity

**Run SwiftFormat**:
- Auto-fix formatting issues
- 4-space indentation
- Trailing commas in collections
- Blank lines between functions

**Audit Layer Separation**:
- Domain imports only Foundation ✅
- Data imports Domain + Foundation (no SwiftUI) ✅
- Presentation imports Domain + SwiftUI (no Data) ✅
- App imports all layers ✅

**Audit Protocol Conformance**:
- Check all Domain protocols are implemented
- Check all Use Cases are wired in container
- Check all Repositories conform to protocols

**Output**:
- Full project ready to build
- Integration audit log (pass/fail for each check)
- SwiftLint warnings (if any) for PR review

### CRITICAL: No Changes to Phase 1-3 Output

Integration does not rewrite Domain, Data, or Presentation code. It wires, audits, and flags issues. If issues can't be auto-fixed, they're flagged for user review in the PR.

---

## Example: Full 4-Phase Output

After all 4 phases complete, the project structure looks like:

```
TimerApp/
├── App/
│   ├── DI/
│   │   └── Container.swift (created Phase 4)
│   └── main.swift (created Phase 4)
├── Domain/
│   ├── Entities/
│   │   └── Timer.swift (Phase 2)
│   ├── UseCases/
│   │   ├── GetTimersUseCase.swift (Phase 2)
│   │   ├── StartTimerUseCase.swift (Phase 2)
│   │   └── DeleteTimerUseCase.swift (Phase 2)
│   ├── Repositories/
│   │   └── TimerRepository.swift (Phase 2, protocol only)
│   └── Errors/
│       └── DomainError.swift (Phase 2)
├── Data/
│   ├── DTOs/
│   │   └── TimerDTO.swift (Phase 3a)
│   ├── Mappers/
│   │   └── TimerMapper.swift (Phase 3a)
│   ├── Repositories/
│   │   └── DefaultTimerRepository.swift (Phase 3a)
│   └── Network/
│       └── APIClient.swift (Phase 3a)
└── Presentation/
    ├── Views/
    │   ├── TimerListView.swift (Phase 3b)
    │   ├── TimerDetailView.swift (Phase 3b)
    │   └── CreateTimerView.swift (Phase 3b)
    ├── ViewModels/
    │   ├── TimerListViewModel.swift (Phase 3b)
    │   ├── TimerDetailViewModel.swift (Phase 3b)
    │   └── CreateTimerViewModel.swift (Phase 3b)
    └── Coordinators/
        └── TimerCoordinator.swift (Phase 3b)

ADR.txt (Phase 1)
Integration Audit Log (Phase 4)
```

---

## See Also

- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-builder-lite/SKILL.md` — Full pipeline overview
- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-builder-lite/references/ios-standards.md` — Standards each phase follows
- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-builder-lite/references/ios-architecture-template.md` — Code templates
