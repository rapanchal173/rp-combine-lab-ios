# Apple Combine Framework — Complete Overview

> A practical, senior-level guide to Apple's Combine framework with concepts, operators, architecture guidance, and real-project examples.

---

## 1. What is Combine?

**Combine** is Apple's declarative framework for processing **asynchronous values over time**.

The core mental model is:

```text
Publisher
   │
   ▼
Operators
   │
   ▼
Subscriber
```

A **Publisher** emits values, an **Operator** transforms or coordinates those values, and a **Subscriber** consumes them.

Example:

```swift
import Combine

let cancellable = [1, 2, 3, 4, 5].publisher
    .filter { $0 % 2 == 0 }
    .map { $0 * 10 }
    .sink { value in
        print(value)
    }

// 20
// 40
```

Conceptually:

```text
[1,2,3,4,5]
     │
   filter
     │
   [2,4]
     │
    map
     │
 [20,40]
     │
    sink
```

---

## 2. The Three Core Pieces

### 2.1 Publisher

A publisher defines two associated types:

```swift
protocol Publisher {
    associatedtype Output
    associatedtype Failure: Error
}
```

Example:

```swift
AnyPublisher<User, Error>
```

This means:

- It emits `User`
- It can fail with `Error`

Whereas:

```swift
AnyPublisher<String, Never>
```

means:

- It emits `String`
- It never fails

---

### 2.2 Subscriber

A subscriber receives values and completion events.

The most commonly used subscriber is `sink`.

```swift
publisher
    .sink(
        receiveCompletion: { completion in
            print(completion)
        },
        receiveValue: { value in
            print(value)
        }
    )
```

A stream can emit:

```text
value
value
value
completion
```

Completion is either:

```swift
.finished
```

or:

```swift
.failure(error)
```

Once completed, the publisher cannot emit additional values.

---

### 2.3 Subscription

A subscription connects a Publisher to a Subscriber.

```text
Publisher
   │
Subscription
   │
Subscriber
```

In day-to-day Combine code, subscriptions are usually held using:

```swift
AnyCancellable
```

Example:

```swift
private var cancellables = Set<AnyCancellable>()

publisher
    .sink { value in
        print(value)
    }
    .store(in: &cancellables)
```

When the `AnyCancellable` is released, the subscription is cancelled.

---

## 3. Subjects

A `Subject` is both:

- A Publisher
- Something into which you can manually send values

Combine provides two important subjects.

### 3.1 PassthroughSubject

Does not retain the latest value.

```swift
let subject = PassthroughSubject<String, Never>()

subject
    .sink { value in
        print(value)
    }

subject.send("Hello")
subject.send("Combine")
```

Best for:

- User actions
- Commands
- One-time events
- Notifications
- Event buses

---

### 3.2 CurrentValueSubject

Stores the latest value.

```swift
let subject = CurrentValueSubject<Int, Never>(0)

subject.send(1)
subject.send(2)

print(subject.value) // 2
```

A new subscriber immediately receives the current value.

```text
Current value = 2

New Subscriber
      │
      ▼
Immediately receives 2
```

Best for:

- Current application state
- Session state
- Connectivity state
- Feature configuration

---

## 4. @Published

`@Published` is one of the most common Combine integrations with SwiftUI.

```swift
final class UserViewModel: ObservableObject {
    @Published var username = ""
    @Published var isLoading = false
}
```

A publisher for a property is accessed using `$`.

```swift
viewModel.$username
```

Example:

```swift
viewModel.$username
    .sink { value in
        print("Username:", value)
    }
    .store(in: &cancellables)
```

Flow:

```text
username changes
      │
      ▼
   @Published
      │
      ▼
   Publisher
      │
      ▼
   Subscriber
```

---

# 5. Important Operators

Operators are the heart of Combine.

They can:

- Transform
- Filter
- Combine
- Delay
- Retry
- Recover
- Switch threads
- Coordinate multiple streams

---

## 5.1 map

Transforms values.

```swift
[1, 2, 3].publisher
    .map { $0 * 10 }
```

Output:

```text
10
20
30
```

---

## 5.2 compactMap

Transforms and removes `nil`.

```swift
["1", "abc", "3"].publisher
    .compactMap(Int.init)
```

Output:

```text
1
3
```

---

## 5.3 filter

Allows only matching values.

```swift
[1, 2, 3, 4].publisher
    .filter { $0.isMultiple(of: 2) }
```

Output:

```text
2
4
```

---

## 5.4 removeDuplicates

Suppresses consecutive duplicate values.

```swift
$searchText
    .removeDuplicates()
```

Useful for:

- Search
- State streams
- Network status
- Form values

---

## 5.5 debounce

Waits for a quiet period before emitting.

Ideal for search boxes.

```swift
$searchText
    .debounce(
        for: .milliseconds(400),
        scheduler: RunLoop.main
    )
```

Flow:

```text
User types:
s
sw
swi
swif
swift
   │
   └── waits 400 ms
          │
          ▼
      emits "swift"
```

This avoids firing an API request for every keystroke.

---

## 5.6 throttle

Limits how frequently values can be emitted. It limits the rate at which a publisher emits elements.

```swift
    let subject = PassthroughSubject<String, Never>()
    subject
        .throttle(for: .seconds(1), scheduler: DispatchQueue.main, latest: true)
        .sink { value in
            print("Throttle result: \(value)")
        }
        .store(in: &cancellables)
    
    Task {
        subject.send("A") // Emitted immediately (first value in window)
        
        try? await Task.sleep(for: .milliseconds(100))
        subject.send("B")
        
        try? await Task.sleep(for: .milliseconds(100))
        subject.send("C") // Since latest: true, "C" will be emitted when the timer fires
        
        
        // Wait for the 1-second window to pass
        try? await Task.sleep(for: .seconds(1))
        subject.send("D") // Emitted immediately
        
        // Give time for the final output to print before the task finishes
        try? await Task.sleep(for: .seconds(1))
    }
```

Good use cases:

- Scroll events
- Location updates
- Analytics
- Rapid UI actions

---

## 5.7 combineLatest

Combines the latest values from multiple publishers.

Example: login validation.

```swift
    let username = CurrentValueSubject<String, Never>("")
    let password = CurrentValueSubject<String, Never>("")

    // Combine the latest value of both fields
    username.combineLatest(password)
        .map { user, pass in
            return !user.isEmpty && pass.count >= 6
        }
        .sink { isValid in
            print("Form valid: \(isValid)")
        }
        .store(in: &cancellables)

    username.send("alex_dev") // Output: Form valid: false
    password.send("123456")   // Output: Form valid: true   ("alex_dev" + "123456")
    password.send("123")      // Output: Form valid: false  ("alex_dev" + "123")
```

Flow:

```text
email ───────┐
             ├── combineLatest ──> validation ──> button enabled
password ────┘
```

Use when current values from multiple streams together define state.

---

## 5.8 zip

Pairs matching emissions in sequence.

```text
Publisher A: A1 A2 A3
Publisher B: B1 B2 B3

zip:
(A1,B1)
(A2,B2)
(A3,B3)
```

Use when corresponding events must be paired.

---

## 5.9 merge

Combines publishers with the same output type.

```swift
Publishers.Merge(
    manualRefreshPublisher,
    autoRefreshPublisher
)
```

Good for multiple triggers feeding the same workflow.

---

## 5.10 flatMap

Transforms each value into another publisher and flattens the result.

```swift
authService.login()
    .flatMap { token in
        userService.fetchProfile(token: token)
    }
```

Flow:

```text
Login
  │
  ▼
Token
  │
flatMap
  │
  ▼
Fetch Profile
```

This is essential for chaining asynchronous publisher-based operations.

---

# 6. Error Handling

Combine models errors as part of the publisher type.

```swift
enum APIError: Error {
    case invalidResponse
    case decodingFailed
}
```

Example:

```swift
AnyPublisher<User, APIError>
```

Useful error operators:

- `mapError`
- `catch`
- `retry`
- `replaceError`
- `tryMap`

---

## 6.1 retry

```swift
apiPublisher
    .retry(3)
```

Use only for retryable failures.

Good candidates:

- Temporary network failure
- Timeout
- Service unavailable

Avoid blindly retrying:

- 400 Bad Request
- 401 Unauthorized
- Business validation errors
- Payment failures

---

## 6.2 catch

Switches to another publisher when an error occurs.

```swift
apiPublisher
    .catch { _ in
        Just([])
    }
```

---

# 7. Networking with URLSession

Combine integrates directly with `URLSession`.

```swift
struct User: Decodable {
    let id: Int
    let name: String
}

func fetchUsers() -> AnyPublisher<[User], Error> {
    let url = URL(string: "https://example.com/users")!

    return URLSession.shared.dataTaskPublisher(for: url)
        .map(\.data)
        .decode(type: [User].self, decoder: JSONDecoder())
        .eraseToAnyPublisher()
}
```

Usage:

```swift
fetchUsers()
    .receive(on: DispatchQueue.main)
    .sink(
        receiveCompletion: { completion in
            print(completion)
        },
        receiveValue: { users in
            print(users)
        }
    )
    .store(in: &cancellables)
```

---

# 8. eraseToAnyPublisher

Publisher chains often produce very complex concrete types.

Instead of exposing implementation details:

```swift
Publishers.Map<URLSession.DataTaskPublisher, User>
```

prefer:

```swift
func fetchUser() -> AnyPublisher<User, Error>
```

and end the pipeline with:

```swift
.eraseToAnyPublisher()
```

Benefits:

- Hides internal implementation
- Cleaner public APIs
- Easier refactoring
- Better abstraction at architecture boundaries

---

# 9. Schedulers and Threading

Two important operators are:

```swift
subscribe(on:)
receive(on:)
```

## subscribe(on:)

Controls where upstream subscription work begins.

## receive(on:)

Controls where downstream values are delivered.

Typical UI example:

```swift
apiService.fetchUsers()
    .receive(on: DispatchQueue.main)
    .sink { users in
        // Update UI state
    }
    .store(in: &cancellables)
```

The important mental model is:

```text
subscribe(on:)
    affects upstream work

receive(on:)
    affects downstream delivery
```

---

# 10. Real Project Example — Product Search

A realistic e-commerce search flow:

```text
Search Text
    │
 @Published
    │
removeDuplicates
    │
 debounce
    │
 filter
    │
 flatMap
    │
 API
    │
Products
    │
SwiftUI
```

Example ViewModel:

```swift
final class ProductSearchViewModel: ObservableObject {

    @Published var searchText = ""
    @Published var products: [Product] = []
    @Published var isLoading = false

    private let service: ProductService
    private var cancellables = Set<AnyCancellable>()

    init(service: ProductService) {
        self.service = service
        setupSearch()
    }

    private func setupSearch() {
        $searchText
            .removeDuplicates()
            .debounce(
                for: .milliseconds(400),
                scheduler: DispatchQueue.main
            )
            .filter { !$0.isEmpty }
            .handleEvents(
                receiveOutput: { [weak self] _ in
                    self?.isLoading = true
                }
            )
            .flatMap { [service] query in
                service.searchProducts(query: query)
                    .catch { _ in Just([]) }
            }
            .receive(on: DispatchQueue.main)
            .sink { [weak self] products in
                self?.products = products
                self?.isLoading = false
            }
            .store(in: &cancellables)
    }
}
```

SwiftUI:

```swift
struct ProductSearchView: View {

    @StateObject
    private var viewModel: ProductSearchViewModel

    var body: some View {
        VStack {
            TextField("Search", text: $viewModel.searchText)

            if viewModel.isLoading {
                ProgressView()
            }

            List(viewModel.products) { product in
                Text(product.name)
            }
        }
    }
}
```

---

# 11. Real Project Example — Form Validation

Combine is very useful when multiple pieces of state together determine whether a form is valid.

```swift
final class RegistrationViewModel: ObservableObject {

    @Published var email = ""
    @Published var password = ""
    @Published var confirmPassword = ""

    @Published var isFormValid = false

    private var cancellables = Set<AnyCancellable>()

    init() {
        Publishers.CombineLatest3(
            $email,
            $password,
            $confirmPassword
        )
        .map { email, password, confirmation in
            email.contains("@") &&
            password.count >= 8 &&
            password == confirmation
        }
        .assign(to: &$isFormValid)
    }
}
```

SwiftUI:

```swift
Button("Register") {
    // Register
}
.disabled(!viewModel.isFormValid)
```

---

# 12. Real Project Example — Network Connectivity

```swift
networkMonitor.statusPublisher
    .removeDuplicates()
    .sink { status in
        print("Network:", status)
    }
    .store(in: &cancellables)
```

Multiple consumers can subscribe:

```text
Network Monitor
      │
      ▼
   Publisher
  ┌────┼───────────┐
  │    │           │
 UI   Sync      Analytics
```

This is a strong use case for a reactive stream.

---

# 13. Real Project Example — NotificationCenter

Combine integrates with NotificationCenter.

```swift
NotificationCenter.default
    .publisher(for: UIApplication.didEnterBackgroundNotification)
    .sink { _ in
        print("App entered background")
    }
    .store(in: &cancellables)
```

This reduces selector-based observer boilerplate.

---

# 14. Memory Management

One of the most common Combine problems is an accidental retain cycle.

Potential cycle:

```text
self
 │
 ▼
cancellables
 │
 ▼
subscription
 │
 ▼
closure
 │
 ▼
self
```

Example:

```swift
publisher
    .sink { value in
        self.handle(value)
    }
    .store(in: &cancellables)
```

Safer when appropriate:

```swift
publisher
    .sink { [weak self] value in
        self?.handle(value)
    }
    .store(in: &cancellables)
```

Do not use `[weak self]` mechanically. Decide based on expected ownership and subscription lifetime.

---

# 15. Backpressure

Combine supports the concept of **demand**.

A subscriber can conceptually say:

```text
"I am ready for N values."
```

This is known as backpressure.

Most application code using `sink` does not explicitly manage demand, but the concept is important for:

- Custom subscribers
- High-frequency streams
- Large data flows
- Reactive framework understanding

---

# 16. Cold vs Hot Publishers

## Cold Publisher

Work is generally started per subscriber.

Conceptually:

```text
Subscriber A ──> request/work
Subscriber B ──> separate request/work
```

Network publishers are often reasoned about this way.

## Hot Publisher

Values exist independently of an individual subscription.

Examples:

- `PassthroughSubject`
- `CurrentValueSubject`
- Notifications
- Sensor/event streams
- WebSocket streams

---

# 17. share()

If multiple subscribers observe expensive upstream work, `share()` can allow them to share that subscription.

```swift
let sharedPublisher = apiPublisher.share()
```

Conceptually:

```text
             ┌── Subscriber A
API Work ────┤
             └── Subscriber B
```

Without sharing, two subscribers may trigger two separate upstream operations.

---

# 18. combineLatest vs zip vs merge

## combineLatest

Use when the latest values from several streams together determine state.

Best examples:

- Form validation
- Button enabled state
- UI state composition

## zip

Use when emissions must be paired one-by-one.

```text
A1 + B1
A2 + B2
A3 + B3
```

## merge

Use when several publishers emit the same type and should feed one stream.

Example:

```text
Manual Refresh ──┐
                 ├──> Refresh Pipeline
Push Refresh ────┘
```

---

# 19. Combine in MVVM

A traditional Combine-based SwiftUI architecture often looks like:

```text
SwiftUI View
     │
     ▼
@Published State
     │
     ▼
ViewModel
     │
Combine Pipeline
     │
     ▼
Repository
     │
AnyPublisher
     │
     ▼
API / Database
```

Repository:

```swift
protocol ProductRepository {
    func products() -> AnyPublisher<[Product], Error>
}
```

ViewModel:

```swift
final class ProductsViewModel: ObservableObject {

    @Published
    private(set) var products: [Product] = []

    private let repository: ProductRepository
    private var cancellables = Set<AnyCancellable>()

    init(repository: ProductRepository) {
        self.repository = repository
    }

    func load() {
        repository.products()
            .receive(on: DispatchQueue.main)
            .sink(
                receiveCompletion: { completion in
                    print(completion)
                },
                receiveValue: { [weak self] products in
                    self?.products = products
                }
            )
            .store(in: &cancellables)
    }
}
```

---

# 20. Combine vs async/await

For a **single asynchronous result**, modern Swift usually favors `async/await`.

```swift
let users = try await service.fetchUsers()
```

Instead of:

```swift
service.fetchUsers()
    .sink(...)
    .store(in: &cancellables)
```

Use this mental model:

```text
Single asynchronous result
        │
        ▼
   async / await


Values changing over time
        │
        ▼
      Combine
```

Combine remains highly useful for:

- Search text streams
- Notification streams
- Reachability
- Location updates
- Form validation
- WebSocket streams
- Event aggregation
- State composition

---

# 21. Combining Combine with async/await

You do not need to choose only one.

A practical modern architecture is:

```text
UI event/state streams
        │
      Combine
        │
        ▼
    ViewModel
        │
   async/await
        │
        ▼
 Repository / API
```

Example:

```swift
@MainActor
final class ProductViewModel: ObservableObject {

    @Published var searchText = ""
    @Published var products: [Product] = []

    private let repository: ProductRepository
    private var cancellables = Set<AnyCancellable>()

    init(repository: ProductRepository) {
        self.repository = repository

        $searchText
            .debounce(
                for: .milliseconds(400),
                scheduler: DispatchQueue.main
            )
            .removeDuplicates()
            .sink { [weak self] query in
                Task {
                    await self?.search(query)
                }
            }
            .store(in: &cancellables)
    }

    private func search(_ query: String) async {
        do {
            products = try await repository.search(query)
        } catch {
            products = []
        }
    }
}
```

This design uses:

- Combine for reacting to continuously changing UI input
- Swift Concurrency for one-shot asynchronous work

---

# 22. Combine vs Observation

Modern SwiftUI supports the Observation framework:

```swift
@Observable
final class ViewModel {
    var name = ""
}
```

For basic view-state observation, Observation often removes the need for:

```text
ObservableObject
@Published
Combine
```

However, Observation does not replace all reactive-stream behavior.

Combine is still useful for:

- `debounce`
- `throttle`
- `merge`
- `zip`
- `combineLatest`
- event pipelines
- system publishers
- complex reactive transformations

---

# 23. Enterprise / Real-Project Usage

Combine is commonly found in production projects in these areas:

## Existing SwiftUI MVVM codebases

```text
ObservableObject
@Published
AnyPublisher
sink
```

## Search and Autocomplete

```text
Input
→ debounce
→ removeDuplicates
→ API
```

## Networking

Especially in projects built before Swift Concurrency became dominant.

## Event Streams

Examples:

- NotificationCenter
- Connectivity
- Bluetooth
- Location
- WebSocket
- MDM/MAM SDK events

## State Aggregation

```text
Authentication
+
Connectivity
+
Feature Flags
+
Permissions
```

## SDK Design

An SDK can expose:

```swift
var statusPublisher: AnyPublisher<SDKStatus, Never>
```

This works well for clients already using reactive architecture.

---

# 24. Common Mistakes

## Using Combine for everything

Do not turn simple command-style operations into unnecessarily complex pipelines.

Sometimes:

```swift
Task {
    await save()
}
```

is much clearer.

## Nested sink

Avoid this pattern:

```swift
publisher1
    .sink { value in
        publisher2
            .sink { ... }
    }
```

Prefer composition using operators such as `flatMap`.

## Forgetting to retain cancellables

If the returned `AnyCancellable` is not retained, the subscription may be cancelled immediately.

## Retain cycles

Long-lived subscriptions can accidentally retain their owner.

## Overusing AnyPublisher

Type erasure is useful at API and architecture boundaries, but not every internal pipeline needs it.

---

# 25. Recommended Modern Architecture

For a new SwiftUI application:

```text
                 SwiftUI
                    │
                    ▼
          Observation / View State
                    │
                    ▼
                ViewModel
             ┌──────┴──────┐
             │             │
             ▼             ▼
          Combine       async/await
       Value Streams    Commands
             │             │
             │             ▼
             │          Use Case
             │             │
             └──────┬──────┘
                    ▼
                Repository
             ┌──────┴──────┐
             ▼             ▼
            API         Database
```

Use Combine when the problem is:

> "I need to react to values changing over time."

Use `async/await` when the problem is:

> "Perform this async operation and return its result."

---

# 26. Senior-Level Interview Checklist

You should be comfortable explaining:

- Publisher
- Subscriber
- Subscription
- `AnyCancellable`
- Subjects
- `@Published`
- `AnyPublisher`
- Type erasure
- Error typing
- `map`
- `compactMap`
- `flatMap`
- `filter`
- `debounce`
- `throttle`
- `combineLatest`
- `zip`
- `merge`
- `share`
- `receive(on:)`
- `subscribe(on:)`
- Memory management
- Backpressure
- Hot vs cold publishers
- Combine vs async/await
- Combine + Observation
- Practical search / form-validation / event-stream use cases

---

# 27. Final Mental Model

```text
Combine solves:
"Values keep changing over time and I need to react."

Swift Concurrency solves:
"I need to perform asynchronous work."

Observation solves:
"My UI needs to react to state changes."
```

In modern Swift projects, all three can coexist. The best architecture uses each one where it is strongest instead of forcing a single technology across every layer.
