# Swift Concurrency Model for Zed

This document describes how to adapt Zed's concurrency model to Swift, focusing on how to implement the task scheduling system, asynchronous contexts, and thread management.

## Overview of Zed's Concurrency Model

Zed's concurrency model is built around these key components:

1. **Task**: The fundamental unit of concurrency, representing an asynchronous operation.
2. **ForegroundExecutor**: Runs tasks on the main thread (UI thread).
3. **BackgroundExecutor**: Runs tasks on background threads.
4. **AsyncApp/AsyncWindowContext**: Async-friendly context types that allow interacting with application state from async code.

The model enforces a strict separation between UI and background work, with appropriate error handling and cancellation support.

## Swift Implementation

### 1. Task System

In Swift, we'll leverage the native structured concurrency model introduced in Swift 5.5, which provides similar capabilities to Zed's task system.

```swift
/// A task that can be cancelled and awaited upon.
/// If dropped, the task will be automatically cancelled.
final class ZedTask<T> {
    private var task: Task<T, Error>?
    
    /// Creates a task that is already completed with the given value
    static func ready(_ value: T) -> ZedTask<T> {
        let result = ZedTask<T>()
        result.task = Task { value }
        return result
    }
    
    /// Detaches the task to run in the background
    func detach() {
        task = nil
    }
    
    /// Run the task to completion in the background and log any errors
    func detachAndLogError() {
        guard let task = task else { return }
        Task { [task] in
            do {
                _ = try await task.value
            } catch {
                log.error("Detached task failed: \(error)")
            }
        }
        self.task = nil
    }
    
    deinit {
        // Automatically cancel the task if it's dropped
        task?.cancel()
    }
}

// Make ZedTask awaitable
extension ZedTask: AsyncSequence {
    typealias Element = T
    
    struct AsyncIterator: AsyncIteratorProtocol {
        private let task: Task<T, Error>?
        private var consumed = false
        
        init(task: Task<T, Error>?) {
            self.task = task
        }
        
        mutating func next() async throws -> T? {
            if consumed || task == nil {
                return nil
            }
            consumed = true
            return try await task!.value
        }
    }
    
    func makeAsyncIterator() -> AsyncIterator {
        return AsyncIterator(task: task)
    }
}
```

### 2. Executors

In Swift, we'll implement executors to manage task scheduling on appropriate threads:

```swift
/// Runs tasks on background threads
final class BackgroundExecutor {
    private let queue: DispatchQueue
    
    init(label: String = "com.zed.background", qos: DispatchQoS = .userInitiated) {
        self.queue = DispatchQueue(label: label, qos: qos, attributes: .concurrent)
    }
    
    /// Enqueues the given operation to be run to completion on a background thread
    func spawn<T>(_ operation: @escaping () async throws -> T) -> ZedTask<T> {
        let task = ZedTask<T>()
        task.task = Task.detached {
            try await operation()
        }
        return task
    }
    
    /// Block the current thread until the given operation completes
    func block<T>(_ operation: @escaping () async throws -> T) throws -> T {
        let semaphore = DispatchSemaphore(value: 0)
        var result: Result<T, Error>!
        
        let task = Task.detached {
            do {
                let value = try await operation()
                result = .success(value)
            } catch {
                result = .failure(error)
            }
            semaphore.signal()
        }
        
        semaphore.wait()
        task.cancel() // Clean up the task
        
        return try result.get()
    }
    
    /// Returns a task that will complete after the given duration
    func timer(duration: TimeInterval) -> ZedTask<Void> {
        let task = ZedTask<Void>()
        task.task = Task {
            try await Task.sleep(nanoseconds: UInt64(duration * 1_000_000_000))
        }
        return task
    }
    
    /// How many CPUs are available to the executor
    var numCPUs: Int {
        return ProcessInfo.processInfo.activeProcessorCount
    }
}

/// Runs tasks on the main thread
final class ForegroundExecutor {
    /// Enqueues the given operation to be run on the main thread
    func spawn<T>(_ operation: @escaping () async throws -> T) -> ZedTask<T> {
        let task = ZedTask<T>()
        task.task = Task { @MainActor in
            try await operation()
        }
        return task
    }
}
```

### 3. Async Contexts

In Swift, we'll use the actor model to provide safe access to shared state:

```swift
/// An async-friendly version of App with a static lifetime
/// so it can be held across await points
actor AsyncApp {
    private weak var app: App?
    private let backgroundExecutor: BackgroundExecutor
    private let foregroundExecutor: ForegroundExecutor
    
    init(app: App, backgroundExecutor: BackgroundExecutor, foregroundExecutor: ForegroundExecutor) {
        self.app = app
        self.backgroundExecutor = backgroundExecutor
        self.foregroundExecutor = foregroundExecutor
    }
    
    /// Ensure the app is still available
    private func ensureApp() throws -> App {
        guard let app = app else {
            throw ZedError.appReleased
        }
        return app
    }
    
    /// Create a new entity
    func new<T: AnyObject>(_ build: @escaping (EntityContext<T>) -> T) throws -> Entity<T> {
        let app = try ensureApp()
        return try app.new(build)
    }
    
    /// Update an entity
    func updateEntity<T: AnyObject, R>(_ handle: Entity<T>, update: @escaping (inout T, EntityContext<T>) -> R) throws -> R {
        let app = try ensureApp()
        return try app.updateEntity(handle, update: update)
    }
    
    /// Read an entity
    func readEntity<T: AnyObject, R>(_ handle: Entity<T>, read: @escaping (T, App) -> R) throws -> R {
        let app = try ensureApp()
        return try app.readEntity(handle, read: read)
    }
    
    /// Update a window
    func updateWindow<T>(_ window: WindowHandle, update: @escaping (AnyView, inout Window, inout App) -> T) throws -> T {
        let app = try ensureApp()
        return try app.updateWindow(window, update: update)
    }
    
    /// Schedule a task to run in the background
    func backgroundSpawn<R>(_ operation: @escaping () async throws -> R) -> ZedTask<R> where R: Sendable {
        return backgroundExecutor.spawn(operation)
    }
    
    /// Schedule a task to run on the main thread
    func spawn<R>(_ operation: @escaping (AsyncApp) async throws -> R) -> ZedTask<R> {
        return foregroundExecutor.spawn { [weak self] in
            guard let self = self else {
                throw ZedError.appReleased
            }
            return try await operation(self)
        }
    }
    
    /// Get the background executor
    func backgroundExecutor() -> BackgroundExecutor {
        return backgroundExecutor
    }
    
    /// Get the foreground executor
    func foregroundExecutor() -> ForegroundExecutor {
        return foregroundExecutor
    }
    
    /// Subscribe to entity events
    func subscribe<T: EventEmitter, Event>(_ entity: Entity<T>, onEvent: @escaping (Entity<T>, Event, inout App) -> Void) throws -> Subscription where T: AnyObject {
        let app = try ensureApp()
        return app.subscribe(entity, onEvent: onEvent)
    }
}

/// A cloneable, owned handle to the application context,
/// composed with the window associated with the current task
actor AsyncWindowContext {
    private let app: AsyncApp
    private let window: WindowHandle
    
    init(app: AsyncApp, window: WindowHandle) {
        self.app = app
        self.window = window
    }
    
    /// Get the window handle
    func windowHandle() -> WindowHandle {
        return window
    }
    
    /// Update the window
    func update<R>(_ update: @escaping (inout Window, inout App) -> R) async throws -> R {
        return try await app.updateWindow(window) { _, window, app in
            update(&window, &app)
        }
    }
    
    /// Update the root view
    func updateRoot<R>(_ update: @escaping (AnyView, inout Window, inout App) -> R) async throws -> R {
        return try await app.updateWindow(window, update: update)
    }
    
    /// Schedule a task to run on the next frame
    func onNextFrame(_ operation: @escaping (inout Window, inout App) -> Void) {
        Task { [weak self] in
            guard let self = self else { return }
            do {
                try await self.update { window, app in
                    window.onNextFrame(operation)
                }
            } catch {
                // Ignore errors when the window is gone
            }
        }
    }
    
    /// Spawn a task in the context of this window
    func spawn<R>(_ operation: @escaping (AsyncWindowContext) async throws -> R) -> ZedTask<R> {
        let app = app
        return app.foregroundExecutor().spawn { [weak self] in
            guard let self = self else {
                throw ZedError.windowReleased
            }
            return try await operation(self)
        }
    }
}
```

### 4. Background/UI Thread Coordination

In Swift, we can leverage Swift's structured concurrency for coordination between background and UI work:

```swift
extension AsyncApp {
    /// Perform work in the background and then update the UI
    func runInBackgroundAndUpdateUI<T>(
        window: WindowHandle,
        backgroundWork: @escaping () async throws -> T,
        updateUI: @escaping (T, inout Window, inout App) throws -> Void
    ) -> ZedTask<Void> {
        // Start work on background thread
        return backgroundSpawn { [weak self] in
            guard let self = self else { return }
            
            do {
                // Do CPU-intensive work
                let result = try await backgroundWork()
                
                // Update UI on foreground thread
                let _ = try await self.foregroundExecutor().spawn {
                    try await self.updateWindow(window) { _, window, app in
                        try updateUI(result, &window, &app)
                    }
                }
            } catch {
                // Log the error
                log.error("Background work failed: \(error)")
            }
        }
    }
}
```

### 5. Task Cancellation

Swift has built-in task cancellation which we can use to implement Zed's cancellation model:

```swift
extension ZedTask {
    /// Run a task that checks for cancellation
    static func run<T>(priority: TaskPriority? = nil, operation: @escaping () async throws -> T) -> ZedTask<T> {
        let task = ZedTask<T>()
        task.task = Task(priority: priority) {
            try Task.checkCancellation()
            let result = try await operation()
            try Task.checkCancellation()
            return result
        }
        return task
    }
    
    /// Run a computation with a timeout
    static func withTimeout<T>(seconds: Double, operation: @escaping () async throws -> T) -> ZedTask<T?> {
        let task = ZedTask<T?>()
        task.task = Task {
            try await withThrowingTaskGroup(of: T?.self) { group in
                // Add the actual work
                group.addTask {
                    try await operation()
                }
                
                // Add a timeout task
                group.addTask {
                    try await Task.sleep(nanoseconds: UInt64(seconds * 1_000_000_000))
                    return nil
                }
                
                // Return the first completed task
                let result = try await group.next()
                
                // Cancel any remaining tasks
                group.cancelAll()
                
                return result
            }
        }
        return task
    }
}
```

### 6. Error Handling

Swift's try/catch system combined with Result types provides good error handling:

```swift
enum ZedError: Error {
    case appReleased
    case windowReleased
    case entityReleased
    case taskCancelled
    // ... other error types
}

extension ZedTask {
    /// Handle errors in a task
    func mapError(_ handler: @escaping (Error) -> Error) -> ZedTask<T> {
        guard let task = self.task else {
            return ZedTask.ready(try! Result<T, Error>.failure(ZedError.taskCancelled).get())
        }
        
        let newTask = ZedTask<T>()
        newTask.task = Task {
            do {
                return try await task.value
            } catch {
                throw handler(error)
            }
        }
        return newTask
    }
    
    /// Provide a fallback value if the task fails
    func fallback(_ defaultValue: T) -> ZedTask<T> {
        guard let task = self.task else {
            return ZedTask.ready(defaultValue)
        }
        
        let newTask = ZedTask<T>()
        newTask.task = Task {
            do {
                return try await task.value
            } catch {
                return defaultValue
            }
        }
        return newTask
    }
}
```

### 7. Thread Safety with Actors

Swift's actor model enforces thread safety:

```swift
actor EntityManager {
    private var entities: [EntityId: WeakEntityBox] = [:]
    
    func getEntity<T: AnyObject>(_ id: EntityId) -> Entity<T>? {
        guard let box = entities[id] as? WeakEntityBox<T>,
              let instance = box.entity else {
            return nil
        }
        return Entity(id: id, instance: instance)
    }
    
    func addEntity<T: AnyObject>(_ entity: T) -> Entity<T> {
        let id = EntityId.next()
        let box = WeakEntityBox(entity: entity)
        entities[id] = box
        return Entity(id: id, instance: entity)
    }
    
    func updateEntity<T: AnyObject, R>(_ handle: Entity<T>, update: (inout T) -> R) throws -> R {
        guard let entity = getEntity(handle.id)?.instance else {
            throw ZedError.entityReleased
        }
        
        var instance = entity
        let result = update(&instance)
        return result
    }
}

class WeakEntityBox<T: AnyObject> {
    weak var entity: T?
    
    init(entity: T) {
        self.entity = entity
    }
}
```

## Adaptation Considerations

1. **Swift's Native Concurrency** - Leverage Swift's async/await and actor model rather than reimplementing Zed's concurrency primitives entirely.

2. **MainActor** - Use Swift's `@MainActor` attribute to annotate UI code instead of manually dispatching to the main thread.

3. **Structured Concurrency** - Swift's task groups and child tasks provide structured concurrency features that can replace some of Zed's manual task management.

4. **Cancellation Handling** - Swift's cooperative cancellation system works similarly to Zed's. Use `withTaskCancellationHandler` for cleanup.

5. **Thread Safety** - Swift's actors provide thread safety guarantees similar to what Zed achieves with its mutex locking in the original Rust code.

6. **Memory Management** - Use Swift's weak references to prevent retain cycles in subscription callbacks and entity references.

7. **Isolation** - Swift's sendable checking helps enforce the thread isolation that Zed maintains with its `!Send` marker.

## Performance Considerations

1. **Actor Contention** - Actors are great for thread safety but can become bottlenecks. Consider using actor isolation only where thread safety is necessary.

2. **Task Priority** - Use Swift's task priorities to match Zed's priority system.

3. **Fine-Grained Actors** - Break up large actors into smaller, more focused ones to reduce contention.

4. **Thread Pool Size** - Configure the background executor's thread pool size based on the available CPU cores.

5. **Continuation Passing** - For long-running operations, consider using continuations to avoid blocking actor threads.

6. **Swift's Task System** - The Swift runtime optimizes task scheduling, but for critical paths, consider manual thread management.

## Example: Background Work with UI Update

Here's a complete example of performing an expensive operation in the background and updating the UI:

```swift
extension Editor {
    func scrollToPosition(_ position: Point<Float>, window: Window, cx: Context<Editor>) {
        hide_hover(self, cx: cx)
        
        // Store workspace for persistence
        let workspaceId = self.workspace?.1
        let displayMap = self.displayMap.update(cx) { map, cx in map.snapshot(cx) }
        
        // Position adjustment logic
        let adjustedPosition = self.scrollManager.forbidVerticalScroll 
            ? Point(x: position.x, y: self.scrollManager.anchor.scrollPosition(displayMap).y)
            : position
        
        // Update scroll position
        self.scrollManager.setScrollPosition(
            adjustedPosition,
            displayMap: displayMap,
            local: true,
            autoscroll: false,
            workspaceId: workspaceId,
            window: window,
            cx: cx
        )
        
        // Refresh UI elements
        self.refreshInlayHints(reason: .newLinesShown, cx: cx)
        
        // Save scroll position in the background
        if let workspaceId = workspaceId {
            let itemId = cx.entity().entityId.asUInt64
            
            // Run in background and persist
            cx.foregroundExecutor().spawn { [backgroundExecutor = cx.backgroundExecutor()] in
                backgroundExecutor.spawn {
                    log.debug("Saving scroll position for item \(itemId) in workspace \(workspaceId)")
                    do {
                        try await Database.shared.saveScrollPosition(
                            itemId: itemId,
                            workspaceId: workspaceId,
                            topRow: self.anchor.topRow(displayMap.bufferSnapshot),
                            offsetX: self.anchor.offset.x,
                            offsetY: self.anchor.offset.y
                        )
                    } catch {
                        log.error("Failed to save scroll position: \(error)")
                    }
                }.detach()
            }.detach()
        }
        
        // Notify listeners
        cx.notify()
    }
}
```

## Conclusion

Adapting Zed's concurrency model to Swift involves leveraging Swift's native concurrency features rather than directly translating Rust's approach. By using Swift's actors, tasks, and async/await pattern, we can achieve equivalent functionality with better integration into the Swift ecosystem.

The key is to maintain the separation between UI and background work while ensuring proper state management, cancellation handling, and thread safety. Swift's actor model provides excellent isolation guarantees that align well with Zed's design goals.