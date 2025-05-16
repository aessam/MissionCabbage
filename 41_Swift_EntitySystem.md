# Swift Implementation: Entity System

This document outlines the approach for reimplementing Zed's entity system in Swift, adapting the Rust design patterns to Swift's language features and memory management model.

## Hybrid SwiftUI/AppKit Integration

After analyzing GPUI's similarities to SwiftUI, we've determined that a hybrid SwiftUI/AppKit approach will best leverage Swift's capabilities while maintaining performance in critical areas. This approach affects how the Entity System needs to be designed.

### Entity System Design for Hybrid UI

1. **View Representation Protocol**: The Entity System will include protocols that allow entities to declare their UI representation in either SwiftUI or AppKit, with bridging between the two systems.

```swift
protocol ViewRepresentable {
    associatedtype ViewType
    func createView() -> ViewType
    func updateView(_ view: ViewType)
}

protocol SwiftUIRepresentable: ViewRepresentable where ViewType: View {
    // SwiftUI specific methods
}

protocol AppKitRepresentable: ViewRepresentable where ViewType: NSView {
    // AppKit specific methods
}
```

2. **UI Binding System**: Specialized bindings to efficiently update UI from entity state changes with minimal overhead:

```swift
class EntityBinding<T, V: ViewRepresentable> {
    let entity: Entity<T>
    weak var view: V.ViewType?
    
    func update() {
        if let view = view {
            entity.read { state in
                // Update view from entity state
            }
        }
    }
}
```

### Platform Integration Considerations

1. **SwiftUI/AppKit Bridging**: The Entity System will provide efficient bridging between SwiftUI and AppKit components through specialized wrapper types and protocols.

2. **Platform Events**: Unified event handling system that normalizes platform events for both SwiftUI and AppKit components.

3. **Process Lifecycle**: Lifecycle management adapted to work with both SwiftUI app structure and AppKit delegate patterns.

4. **File System Access**: Sandboxed file system access through a common API that works with both UI frameworks.

5. **XPC Communication**: Secure inter-process communication for extension isolation, especially important for editor extensions.

6. **Rendering Integration**: Efficient rendering pipeline that can leverage both SwiftUI's declarative updates and AppKit's direct drawing capabilities.

7. **Metal Integration**: GPU acceleration where appropriate, with proper abstractions to work in both UI frameworks.

This hybrid approach allows us to use SwiftUI for UI-heavy components where its declarative style shines, while using AppKit for performance-critical editing experiences.

## Overview

Zed's entity system is a core architectural pattern based on the Entity-Component-System paradigm but tailored for UI application state management. The system provides a way to:

1. Manage uniquely identifiable objects with strong and weak references
2. Control object lifecycle through reference counting
3. Provide type-safe access to entity state
4. Enable concurrency-safe entity access and updates
5. Support observation and notification of state changes

In Rust, this is implemented using a combination of slotmaps for entity storage, reference counting for lifecycle management, and a context system for entity access. In Swift, we can leverage the language's built-in reference counting, type system, and concurrency features to implement an equivalent system.

## Core Components to Implement

The key components of the entity system we need to implement in Swift are:

1. **EntityId**: A unique identifier for entities
2. **EntityStore**: Storage for all entities in the application
3. **Entity\<T>**: A strong reference to an entity of type T
4. **WeakEntity\<T>**: A weak reference to an entity of type T
5. **AnyEntity**: A type-erased reference to any entity
6. **Context\<T>**: A context for interacting with entities

These components will form the foundation of our Swift implementation, adapting the core patterns to Swift's language features and memory management model.

## Swift Implementation Strategy

### 1. Entity Identifier

```swift
/// A unique identifier for entities across the application
struct EntityId: Hashable, CustomStringConvertible {
    private let rawValue: UInt64
    
    init(rawValue: UInt64) {
        self.rawValue = rawValue
    }
    
    var description: String {
        return "\(rawValue)"
    }
}
```

### 2. Entity Storage

```swift
/// Central storage for all entities in the application
final class EntityStore {
    private var entities: [EntityId: Any] = [:]
    private var refCounts: [EntityId: Int] = [:]
    private var nextId: UInt64 = 1
    private let lock = NSRecursiveLock()
    
    func reserve<T>() -> Entity<T> {
        lock.lock()
        defer { lock.unlock() }
        
        let id = EntityId(rawValue: nextId)
        nextId += 1
        refCounts[id] = 1
        
        return Entity<T>(id: id, store: self)
    }
    
    func insert<T>(_ value: T, for id: EntityId) {
        lock.lock()
        defer { lock.unlock() }
        
        entities[id] = value
    }
    
    func read<T>(_ id: EntityId) -> T? {
        lock.lock()
        defer { lock.unlock() }
        
        return entities[id] as? T
    }
    
    func update<T, R>(_ id: EntityId, _ action: (inout T) -> R) -> R? {
        lock.lock()
        defer { lock.unlock() }
        
        guard var value = entities[id] as? T else { return nil }
        let result = action(&value)
        entities[id] = value
        return result
    }
    
    func retain(_ id: EntityId) -> Bool {
        lock.lock()
        defer { lock.unlock() }
        
        guard let count = refCounts[id] else { return false }
        refCounts[id] = count + 1
        return true
    }
    
    func release(_ id: EntityId) {
        lock.lock()
        defer { lock.unlock() }
        
        guard let count = refCounts[id] else { return }
        if count == 1 {
            entities.removeValue(forKey: id)
            refCounts.removeValue(forKey: id)
        } else {
            refCounts[id] = count - 1
        }
    }
}
```

### 3. Entity Reference

```swift
/// A strong reference to an entity of type T
final class Entity<T>: Hashable {
    let id: EntityId
    private weak var store: EntityStore?
    
    init(id: EntityId, store: EntityStore) {
        self.id = id
        self.store = store
    }
    
    deinit {
        store?.release(id)
    }
    
    func read<R>(_ action: (T) -> R) -> R? {
        guard let store = store, let value = store.read(id) as? T else { return nil }
        return action(value)
    }
    
    func update<R>(_ action: (inout T) -> R) -> R? {
        return store?.update(id, action)
    }
    
    func downgrade() -> WeakEntity<T> {
        return WeakEntity<T>(id: id, store: store)
    }
    
    static func == (lhs: Entity<T>, rhs: Entity<T>) -> Bool {
        return lhs.id == rhs.id
    }
    
    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

### 4. Weak Entity Reference

```swift
/// A weak reference to an entity of type T
final class WeakEntity<T>: Hashable {
    let id: EntityId
    private weak var store: EntityStore?
    
    init(id: EntityId, store: EntityStore?) {
        self.id = id
        self.store = store
    }
    
    func upgrade() -> Entity<T>? {
        guard let store = store, store.retain(id) else { return nil }
        return Entity<T>(id: id, store: store)
    }
    
    func read<R>(_ action: (T) -> R) -> R? {
        guard let entity = upgrade() else { return nil }
        defer { _ = entity } // Ensure entity is released
        return entity.read(action)
    }
    
    func update<R>(_ action: (inout T) -> R) -> R? {
        guard let entity = upgrade() else { return nil }
        defer { _ = entity } // Ensure entity is released
        return entity.update(action)
    }
    
    static func == (lhs: WeakEntity<T>, rhs: WeakEntity<T>) -> Bool {
        return lhs.id == rhs.id
    }
    
    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

### 5. Application Context

```swift
/// The root application context
final class App {
    let entities: EntityStore
    
    init() {
        self.entities = EntityStore()
    }
    
    func new<T>(_ initialValue: T) -> Entity<T> {
        let entity = entities.reserve()
        entities.insert(initialValue, for: entity.id)
        return entity
    }
    
    func read<T, R>(_ entity: Entity<T>, _ action: (T) -> R) -> R? {
        return entity.read(action)
    }
    
    func update<T, R>(_ entity: Entity<T>, _ action: (inout T) -> R) -> R? {
        return entity.update(action)
    }
}
```

### 6. Entity Context

```swift
/// Context for entity updates
final class Context<T> {
    private let app: App
    private let entity: Entity<T>
    
    init(app: App, entity: Entity<T>) {
        self.app = app
        self.entity = entity
    }
    
    func update<R>(_ action: (inout T) -> R) -> R? {
        return entity.update(action)
    }
    
    func new<U>(_ initialValue: U) -> Entity<U> {
        return app.new(initialValue)
    }
    
    func notify() {
        // Trigger UI update
    }
}
```

## Advanced Swift Features

### 1. Type Erasure

For a complete implementation, we'll need type-erased entities similar to `AnyEntity` in Rust:

```swift
/// A type-erased entity reference
final class AnyEntity: Hashable {
    let id: EntityId
    private let typeId: ObjectIdentifier
    private weak var store: EntityStore?
    
    init<T>(_ entity: Entity<T>) {
        self.id = entity.id
        self.typeId = ObjectIdentifier(T.self)
        self.store = entity.store
    }
    
    func downcast<T>() -> Entity<T>? {
        if typeId == ObjectIdentifier(T.self),
           let store = store,
           store.retain(id) {
            return Entity<T>(id: id, store: store)
        }
        return nil
    }
    
    static func == (lhs: AnyEntity, rhs: AnyEntity) -> Bool {
        return lhs.id == rhs.id
    }
    
    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

### 2. Actor-Based Concurrency

Swift's actor model can simplify concurrency management:

```swift
actor EntityActor {
    private var entities: [EntityId: Any] = [:]
    private var refCounts: [EntityId: Int] = [:]
    private var nextId: UInt64 = 1
    
    func reserve<T>() -> Entity<T> {
        let id = EntityId(rawValue: nextId)
        nextId += 1
        refCounts[id] = 1
        
        return Entity<T>(id: id, actor: self)
    }
    
    func insert<T>(_ value: T, for id: EntityId) {
        entities[id] = value
    }
    
    func read<T>(_ id: EntityId) -> T? {
        return entities[id] as? T
    }
    
    func update<T, R>(_ id: EntityId, _ action: (inout T) -> R) -> R? {
        guard var value = entities[id] as? T else { return nil }
        let result = action(&value)
        entities[id] = value
        return result
    }
    
    // Retain and release methods similar to EntityStore
}
```

### 3. Combine Integration

Combine framework can be used for reactive state updates:

```swift
extension Entity {
    func publisher() -> AnyPublisher<T, Never> {
        let subject = CurrentValueSubject<T?, Never>(nil)
        
        // Setup observation
        let observer = EntityObserver(entity: self) { newValue in
            subject.send(newValue)
        }
        
        // Initial value
        if let value = self.read({ $0 }) {
            subject.send(value)
        }
        
        return subject
            .compactMap { $0 }
            .handleEvents(receiveCancel: {
                // Clean up observer
                observer.invalidate()
            })
            .eraseToAnyPublisher()
    }
}
```

## Hybrid UI Framework Integration

The entity system supports integration with both SwiftUI and AppKit through specialized adapters.

### SwiftUI Integration

```swift
@propertyWrapper
struct EntityState<T>: DynamicProperty {
    @StateObject private var observer: EntityStateObserver<T>
    
    init(_ entity: Entity<T>) {
        self._observer = StateObject(wrappedValue: EntityStateObserver(entity: entity))
    }
    
    var wrappedValue: T {
        get { observer.value }
        nonmutating set {
            observer.entity.update { $0 = newValue }
        }
    }
    
    var projectedValue: Binding<T> {
        Binding(
            get: { self.wrappedValue },
            set: { self.wrappedValue = $0 }
        )
    }
}

class EntityStateObserver<T>: ObservableObject {
    @Published var value: T
    let entity: Entity<T>
    private var cancellable: AnyCancellable?
    
    init(entity: Entity<T>) {
        self.entity = entity
        self.value = entity.read { $0 }!
        
        self.cancellable = entity.publisher()
            .receive(on: DispatchQueue.main)
            .sink { [weak self] newValue in
                self?.value = newValue
            }
    }
}
```

### AppKit Integration

```swift
class EntityViewController<T>: NSViewController {
    let entity: Entity<T>
    private var observer: NSKeyValueObservation?
    
    init(entity: Entity<T>) {
        self.entity = entity
        super.init(nibName: nil, bundle: nil)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) not implemented")
    }
    
    override func loadView() {
        // Create custom view for high-performance rendering
        self.view = EntityBackedView(entity: entity)
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupObservation()
    }
    
    private func setupObservation() {
        // Set up observation system for entity changes
        // This is a custom system optimized for AppKit performance
    }
}

class EntityBackedView<T>: NSView {
    let entity: Entity<T>
    
    init(entity: Entity<T>) {
        self.entity = entity
        super.init(frame: .zero)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) not implemented")
    }
    
    override func draw(_ dirtyRect: NSRect) {
        // High-performance custom drawing based on entity state
        entity.read { state in
            // Custom drawing implementation using Core Graphics
            // for maximum performance with potentially large text buffers
        }
    }
}
```

### Bridging Between Frameworks

```swift
struct AppKitViewRepresentable<T>: NSViewRepresentable {
    let entity: Entity<T>
    
    func makeNSView(context: Context) -> EntityBackedView<T> {
        return EntityBackedView(entity: entity)
    }
    
    func updateNSView(_ nsView: EntityBackedView<T>, context: Context) {
        nsView.needsDisplay = true
    }
}

class SwiftUIHostingController<T, Content: View>: NSViewController {
    let entity: Entity<T>
    let viewBuilder: (Entity<T>) -> Content
    
    init(entity: Entity<T>, @ViewBuilder viewBuilder: @escaping (Entity<T>) -> Content) {
        self.entity = entity
        self.viewBuilder = viewBuilder
        super.init(nibName: nil, bundle: nil)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) not implemented")
    }
    
    override func loadView() {
        let hostingView = NSHostingView(rootView: viewBuilder(entity))
        self.view = hostingView
    }
}
```
```

## Handling Entity Lifecycle

In Rust, entities are automatically released when all references are dropped. In Swift, we can use the built-in reference counting system:

```swift
// Using the entity in a view
struct EntityView<T>: View {
    let entity: Entity<T>
    
    var body: some View {
        entity.read { value in
            // Render the value
            Text("\(String(describing: value))")
        }
        .onDisappear {
            // No need to explicitly release the entity - it will be
            // released when this view is deinitialized
        }
    }
}

// Creating an entity
let app = App()
let entity = app.new(MyState())

// Using weak references to avoid retain cycles
class MyComponent {
    let entity: Entity<MyState>
    weak var parent: WeakEntity<ParentState>?
    
    init(entity: Entity<MyState>, parent: Entity<ParentState>) {
        self.entity = entity
        self.parent = parent.downgrade()
    }
    
    func updateParent() {
        parent?.update { state in
            // Update parent state
        }
    }
}
```

## Key Differences from Rust Implementation

1. **Memory Management**: Swift uses ARC (Automatic Reference Counting) while Rust uses a custom reference counting system.

2. **Thread Safety**: Swift offers actors for isolated state while Rust uses locks and a single-threaded foreground model.

3. **Type System**: Swift uses generics similar to Rust but with dynamic casting through `as?` rather than Rust's static downcasting.

4. **Error Handling**: Swift's optionals and `Result` type can replace Rust's `anyhow::Result` for error handling.

5. **Concurrency Model**: Swift's structured concurrency with async/await and actors provides a more integrated approach than Rust's custom task system.

## Implementation Challenges

1. **Reference Cycles**: Careful management of strong and weak references is needed to prevent retain cycles.

2. **Thread Safety**: Ensuring thread-safe access while maintaining performance requires careful design.

3. **Type Erasure**: Swift's type system requires explicit type erasure for dynamic dispatch.

4. **Performance**: Monitoring memory overhead and performance characteristics particularly for large numbers of entities.

5. **UI Integration**: Seamless integration with SwiftUI's state management system requires additional abstractions.

## Conclusion

The Swift implementation of Zed's entity system leverages Swift's strengths in object-oriented programming, reference counting, and actor-based concurrency. While the core concepts remain the same as the Rust implementation, the Swift version benefits from tighter integration with the platform's native capabilities.

This approach allows for a cleaner, more idiomatic implementation while preserving the key strengths of the original design: strong typing, safe concurrency, and efficient memory management.