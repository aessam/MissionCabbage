# Swift Implementation: ReactiveUI

This document outlines the approach for adapting Zed's reactive UI system (GPUI) to Swift, leveraging Swift's language features and UI frameworks to implement a similar architecture.

## Hybrid UI Approach

After analyzing GPUI's architecture and comparing it to SwiftUI, we've determined that a hybrid approach combining SwiftUI and AppKit will deliver the best results:

### UI Framework Distribution

- **SwiftUI**: For declarative, state-driven UI components where performance is not critical
- **AppKit**: For performance-critical components, especially those involving text editing
- **Custom Rendering**: For specialized, high-performance rendering needs (similar to GPUI's current approach)

```swift
// Example of hybrid approach declaration at the application level
class ZedApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.[editor], TextEditor.self) // Registers AppKit text editor
                .environment(\.[navigationPanel], NavigationPanel.self) // SwiftUI navigation
                .environment(\.[completions], CompletionsPanel.self) // SwiftUI completions panel
        }
    }
}

// Content view mixing SwiftUI and AppKit components
struct ContentView: View {
    @Environment(\.editor) var editor
    
    var body: some View {
        HSplitView {
            NavigationPanelView() // Pure SwiftUI
            
            EditorView() // AppKit-backed NSViewRepresentable
            
            CompletionsPanelView() // Pure SwiftUI
        }
    }
}
```

## Overview

Zed's UI framework, GPUI, combines elements of Entity-Component-System (ECS) architecture with a declarative UI approach. The system provides:

1. A declarative view definition syntax with composition
2. Efficient diffing and rendering of UI state changes
3. Flexbox-based layout system
4. Component-based architecture for reusability
5. Integration with the entity system for state management

The Swift implementation will adapt these concepts to work with Swift's native capabilities while maintaining the core architectural principles that make Zed's UI system powerful and flexible.

## Core Components to Implement

### 1. Elements

Elements in a Swift UI system would be the building blocks of the user interface. They would handle layout, painting, and event handling. We need to implement protocols that define the core requirements for UI elements:

- Unique identification
- Layout calculation
- Rendering to screen
- Event handling

### 2. Views and Rendering

Views would be entities that implement rendering logic, converting state into a tree of elements. In Swift, this would be represented as a protocol that defines how to render a view's state into UI elements.

### 3. Component System

The Swift implementation should include a component system for creating reusable UI patterns, allowing components to be composed together while maintaining clean separation of concerns.

## Swift Implementation Strategy

### 1. View Protocol

```swift
/// Protocol for defining UI components that can render to the screen
protocol View {
    /// Render this view into a view hierarchy
    func render() -> some SwiftUI.View
    
    /// Get a unique identifier for this view, if it has one
    var id: ViewID? { get }
    
    /// Called when this view needs to be updated due to state changes
    func update()
}
```

### 2. Element System

```swift
/// The basic building block of the UI system
protocol Element {
    /// Layout phase - determine size and position
    func layout(in rect: CGRect) -> CGSize
    
    /// Update phase - process pending state changes
    func update()
    
    /// Render phase - draw the element to the screen
    func render(in context: GraphicsContext)
    
    /// Handle input events directed at this element
    func handleEvent(_ event: InputEvent) -> Bool
}

/// A concrete implementation of an element
class BoxElement: Element {
    var children: [AnyElement] = []
    var style: Style
    
    init(style: Style = .default) {
        self.style = style
    }
    
    func layout(in rect: CGRect) -> CGSize {
        // Implement flexbox layout algorithm
        // Position children and return resulting size
    }
    
    func update() {
        children.forEach { $0.update() }
    }
    
    func render(in context: GraphicsContext) {
        // Draw background, border, etc.
        children.forEach { $0.render(in: context) }
    }
    
    func handleEvent(_ event: InputEvent) -> Bool {
        // Delegate to children if appropriate
        for child in children.reversed() {
            if child.handleEvent(event) {
                return true
            }
        }
        return false
    }
    
    func add(child: AnyElement) {
        children.append(child)
    }
}
```

### 3. Type Erasure

```swift
/// Type-erased element wrapper
struct AnyElement {
    private let _layout: (CGRect) -> CGSize
    private let _update: () -> Void
    private let _render: (GraphicsContext) -> Void
    private let _handleEvent: (InputEvent) -> Bool
    
    init<E: Element>(_ element: E) {
        _layout = { element.layout(in: $0) }
        _update = { element.update() }
        _render = { element.render(in: $0) }
        _handleEvent = { element.handleEvent($0) }
    }
    
    func layout(in rect: CGRect) -> CGSize {
        _layout(rect)
    }
    
    func update() {
        _update()
    }
    
    func render(in context: GraphicsContext) {
        _render(context)
    }
    
    func handleEvent(_ event: InputEvent) -> Bool {
        _handleEvent(event)
    }
}
```

### 4. Integration with Swift UI

Although we could build our own UI system from scratch, it's more efficient to leverage SwiftUI's capabilities while adapting the GPUI concepts:

```swift
struct EntityView<T: EntityState>: SwiftUI.View {
    @ObservedObject var state: EntityStateObserver<T>
    
    init(_ entity: Entity<T>) {
        _state = ObservedObject(wrappedValue: EntityStateObserver(entity: entity))
    }
    
    var body: some SwiftUI.View {
        state.value.render()
    }
}
```

### 5. Layout System

Implementing a flexbox layout system:

```swift
class FlexboxLayout {
    enum Direction {
        case row, column
    }
    
    enum Justify {
        case start, center, end, spaceBetween, spaceAround
    }
    
    enum Align {
        case start, center, end, stretch
    }
    
    struct Node {
        var size: CGSize
        var children: [Node]
        var direction: Direction
        var justify: Justify
        var align: Align
        var padding: EdgeInsets
        var margin: EdgeInsets
        var grow: CGFloat
        var shrink: CGFloat
        
        func layout(in rect: CGRect) -> [CGRect] {
            // Implement flexbox layout algorithm
            // Return list of child rectangles
        }
    }
}
```

## Core Features Implementation

### 1. Element Tree Construction

```swift
// Building an element tree
func buildElementTree() -> AnyElement {
    let root = BoxElement(style: .init(direction: .column))
    
    let header = BoxElement(style: .init(
        direction: .row,
        justify: .spaceBetween,
        padding: EdgeInsets(top: 10, leading: 10, bottom: 10, trailing: 10)
    ))
    
    let content = BoxElement(style: .init(
        direction: .column,
        grow: 1
    ))
    
    let footer = BoxElement(style: .init(
        direction: .row,
        padding: EdgeInsets(top: 10, leading: 10, bottom: 10, trailing: 10)
    ))
    
    // Add text, images, and other elements to the sections
    header.add(child: AnyElement(TextElement(text: "Header")))
    content.add(child: AnyElement(TextElement(text: "Content")))
    footer.add(child: AnyElement(TextElement(text: "Footer")))
    
    // Assemble the tree
    root.add(child: AnyElement(header))
    root.add(child: AnyElement(content))
    root.add(child: AnyElement(footer))
    
    return AnyElement(root)
}
```

### 2. Rendering Cycle

```swift
class Renderer {
    var rootElement: AnyElement
    var lastUpdateTime: TimeInterval = 0
    var needsLayout: Bool = true
    var needsRender: Bool = true
    
    init(rootElement: AnyElement) {
        self.rootElement = rootElement
    }
    
    func update() {
        rootElement.update()
        needsRender = true
    }
    
    func layout(size: CGSize) {
        if needsLayout {
            _ = rootElement.layout(in: CGRect(origin: .zero, size: size))
            needsLayout = false
            needsRender = true
        }
    }
    
    func render(in context: GraphicsContext) {
        if needsRender {
            rootElement.render(in: context)
            needsRender = false
        }
    }
    
    func handleEvent(_ event: InputEvent) -> Bool {
        return rootElement.handleEvent(event)
    }
}
```

### 3. State Management Integration

```swift
class ElementContext<T: EntityState> {
    let entity: Entity<T>
    let app: App
    
    init(entity: Entity<T>, app: App) {
        self.entity = entity
        self.app = app
    }
    
    func update<R>(_ action: (inout T) -> R) -> R? {
        return entity.update(action)
    }
    
    func notify() {
        // Mark the UI for re-render
        app.requestRender()
    }
    
    func createEntity<U>(_ initialValue: U) -> Entity<U> {
        return app.createEntity(initialValue)
    }
}
```

## Advanced Features

### 1. Hybrid Component System

Our component system supports both custom elements and native SwiftUI/AppKit components:

```swift
// Base protocol for all UI components
protocol Component {
    associatedtype Props
    associatedtype ResultView
    
    func render(props: Props) -> ResultView
}

// Custom Element Component (similar to GPUI)
protocol ElementComponent: Component where ResultView == AnyElement {
    func render(props: Props) -> AnyElement
}

// SwiftUI Component
protocol SwiftUIComponent: Component where ResultView: View {
    func render(props: Props) -> ResultView
}

// AppKit Component
protocol AppKitComponent: Component where ResultView: NSView {
    func render(props: Props) -> ResultView
}

// Implementation examples

// Custom element implementation
struct CustomButton: ElementComponent {
    struct Props {
        let label: String
        let action: () -> Void
        let style: ButtonStyle
    }
    
    func render(props: Props) -> AnyElement {
        let element = BoxElement(style: props.style.toElementStyle())
        element.add(child: AnyElement(TextElement(text: props.label)))
        element.onTap(props.action)
        return AnyElement(element)
    }
}

// SwiftUI implementation
struct SwiftUIButton: SwiftUIComponent {
    struct Props {
        let label: String
        let action: () -> Void
        let style: ButtonStyle
    }
    
    func render(props: Props) -> some View {
        Button(action: props.action) {
            Text(props.label)
        }
        .buttonStyle(props.style.toSwiftUIStyle())
    }
}

// AppKit implementation
struct AppKitButton: AppKitComponent {
    struct Props {
        let label: String
        let action: () -> Void
        let style: ButtonStyle
    }
    
    func render(props: Props) -> NSButton {
        let button = NSButton()
        button.title = props.label
        button.target = ButtonActionTarget.shared
        button.action = #selector(ButtonActionTarget.performAction(sender:))
        ButtonActionTarget.shared.registerAction(for: button, action: props.action)
        props.style.apply(to: button)
        return button
    }
}

// Bridging between systems

// Wrap AppKit in SwiftUI
extension AppKitButton {
    func asSwiftUI() -> NSViewRepresentable {
        return AppKitButtonRepresentable(props: props)
    }
}

// Wrap SwiftUI in AppKit
extension SwiftUIButton {
    func asAppKit() -> NSView {
        return NSHostingView(rootView: render(props: props))
    }
}
```

### 2. Data Flow with Combine

```swift
class CombineEntity<T: EntityState>: ObservableObject {
    private let entity: Entity<T>
    @Published var value: T
    private var cancellables = Set<AnyCancellable>()
    
    init(entity: Entity<T>) {
        self.entity = entity
        self.value = entity.read { $0 }!
        
        // Set up observation
        Timer.publish(every: 0.1, on: .main, in: .common)
            .autoconnect()
            .sink { [weak self] _ in
                if let newValue = self?.entity.read({ $0 }) {
                    if newValue != self?.value {
                        self?.value = newValue
                    }
                }
            }
            .store(in: &cancellables)
    }
    
    func update<R>(_ action: (inout T) -> R) -> R? {
        return entity.update(action)
    }
}
```

### 3. SwiftUI Integration

```swift
struct EntityView<T: EntityState & Equatable>: SwiftUI.View {
    @StateObject var state: CombineEntity<T>
    
    init(entity: Entity<T>) {
        _state = StateObject(wrappedValue: CombineEntity(entity: entity))
    }
    
    var body: some SwiftUI.View {
        ContentView(value: state.value, update: { newValue in
            state.update { $0 = newValue }
        })
    }
}

struct ContentView<T: EntityState & Equatable>: SwiftUI.View {
    let value: T
    let update: (T) -> Void
    
    var body: some SwiftUI.View {
        // Render the view based on the entity state
        VStack {
            Text("Value: \(String(describing: value))")
            Button("Update") {
                var newValue = value
                // Modify newValue
                update(newValue)
            }
        }
    }
}
```

## Performance Optimizations

### 1. Diffing and Reconciliation

```swift
protocol Diffable {
    func diff(from other: Self) -> [DiffOperation]
}

enum DiffOperation {
    case update(id: ViewID, properties: [String: Any])
    case insert(id: ViewID, index: Int)
    case delete(id: ViewID)
    case move(id: ViewID, fromIndex: Int, toIndex: Int)
}

class ElementDiffer {
    func diff<E: Element & Diffable>(old: E, new: E) -> [DiffOperation] {
        return old.diff(from: new)
    }
    
    func applyDiff(operations: [DiffOperation], to element: AnyElement) {
        // Apply diff operations to the element tree
    }
}
```

### 2. Caching

```swift
class CachedElement<E: Element>: Element {
    private var element: E
    private var cachedLayout: CGSize?
    private var layoutBounds: CGRect?
    private var needsLayout = true
    private var needsRender = true
    
    init(_ element: E) {
        self.element = element
    }
    
    func layout(in rect: CGRect) -> CGSize {
        if needsLayout || layoutBounds != rect {
            cachedLayout = element.layout(in: rect)
            layoutBounds = rect
            needsLayout = false
        }
        return cachedLayout!
    }
    
    func update() {
        element.update()
        needsLayout = true
        needsRender = true
    }
    
    func render(in context: GraphicsContext) {
        if needsRender {
            element.render(in: context)
            needsRender = false
        }
    }
    
    func handleEvent(_ event: InputEvent) -> Bool {
        return element.handleEvent(event)
    }
}
```

### 3. Async Rendering

```swift
class AsyncRenderer {
    private let renderer: Renderer
    private let renderQueue = DispatchQueue(label: "com.zed.render", qos: .userInteractive)
    private var renderTask: Task<Void, Never>?
    
    init(renderer: Renderer) {
        self.renderer = renderer
    }
    
    func requestRender() {
        renderTask?.cancel()
        renderTask = Task {
            // Perform layout on main thread
            await MainActor.run {
                renderer.layout(size: currentSize)
            }
            
            // Render in background
            let image = await withCheckedContinuation { continuation in
                renderQueue.async {
                    let renderer = ImageRenderer()
                    self.renderer.render(in: renderer.context)
                    continuation.resume(returning: renderer.image)
                }
            }
            
            // Display on main thread
            await MainActor.run {
                displayImage(image)
            }
        }
    }
}
```

## Key Differences from Rust Implementation

1. **UI Framework Integration**: The Swift implementation leverages SwiftUI for many UI components rather than building everything from scratch.

2. **Layout Engine**: While we can implement our own flexbox layout system, we can also leverage SwiftUI's built-in layout system when appropriate.

3. **Drawing System**: Swift provides CoreGraphics and Metal for rendering, which replace GPUI's custom rendering system.

4. **State Management**: Swift's Combine framework and property wrappers offer a different approach to reactive state compared to Rust's manual mutation tracking.

5. **Thread Model**: SwiftUI enforces UI updates on the main thread, which simplifies some concurrency concerns compared to GPUI's custom threading model.

## Implementation Challenges

1. **Performance**: Custom layout and rendering can be computationally expensive. Performance optimization is critical.

2. **Cross-Platform Support**: Ensuring the system works well on all Apple platforms (macOS, iOS, etc.)

3. **Integration with Apple Frameworks**: Seamlessly working with existing Apple frameworks while maintaining our architecture.

4. **State Synchronization**: Keeping entity state and UI state in sync without excessive updates.

5. **Memory Management**: Avoiding retain cycles when views hold references to entities and vice versa.

## SwiftUI-First Approach

Instead of building a parallel UI system, we can take a "SwiftUI-first" approach that adapts the entity system to work with SwiftUI:

```swift
/// A property wrapper that connects an entity to SwiftUI
@propertyWrapper
struct EntityState<T: Equatable>: DynamicProperty {
    @StateObject private var observer: EntityObserver<T>
    
    var wrappedValue: T {
        get { observer.value }
        nonmutating set {
            observer.entity.update { $0 = newValue }
        }
    }
    
    var projectedValue: Binding<T> {
        Binding(
            get: { wrappedValue },
            set: { wrappedValue = $0 }
        )
    }
    
    init(_ entity: Entity<T>) {
        _observer = StateObject(wrappedValue: EntityObserver(entity: entity))
    }
}

/// Example usage
struct UserProfileView: SwiftUI.View {
    @EntityState var user: UserProfile
    
    var body: some SwiftUI.View {
        VStack {
            Text(user.name)
            TextField("Name", text: $user.name)
            Button("Save") {
                // Entity state is automatically updated
            }
        }
    }
}
```

## Conclusion

Adapting Zed's GPUI to Swift involves leveraging Swift's strengths in UI development while maintaining the core architectural principles that make GPUI powerful. The Swift implementation can take advantage of native frameworks like SwiftUI and Combine while incorporating the entity-based state management and component composition patterns from GPUI.

This hybrid approach allows for a more natural Swift experience while preserving the core ideas that make Zed's UI system effective: declarative UI definition, efficient state management, and composable components. By focusing on the integration points between the entity system and SwiftUI, we can create a powerful yet idiomatic Swift implementation that meets the needs of a complex editor like Zed.