# Project Management: From Concept to Implementation

## Layer Information
**Bridge Document**  
**Document ID:** B05  
**Focus Area:** Project Management & File System Integration  

## Connection Map
**Bridges Between:**
- Stratospheric: [05_StratosphericView_ProjectManagement.md](05_StratosphericView_ProjectManagement.md)
- Ground Level: [38_GroundLevel_FileIO.md](38_GroundLevel_FileIO.md)
- Ground Level: [40_GroundLevel_StatePersistence.md](40_GroundLevel_StatePersistence.md)

## Purpose of This Bridge Document

This bridge document connects the high-level architectural concepts of Zed's Project Management system with its concrete implementation details. It explains how abstract concepts around workspace organization, file system interaction, and project state management materialize into specific code structures and algorithms.

## Tracing Project Management Through Layers

### Layer 1: Stratospheric View (Project Management)
At this layer, the project management system is described as an architectural framework providing:

- Workspace organization and structure
- File system access and monitoring
- Search capabilities across projects
- Git integration for version control
- Project-specific configuration management

This layer focuses on "why" the project management system exists and its role in the architecture.

### Layer 2: Ground Level (File I/O)
At this layer, the file system operations and algorithms are detailed:

- Platform-agnostic file system abstractions
- File watching and event propagation
- Atomic file operations for data integrity
- Path manipulation and normalization 
- Resource management for file operations

This layer focuses on "how" the file system operations are implemented internally.

### Layer 3: Ground Level (State Persistence)
At this layer, the concrete state persistence mechanisms are specified:

- SQLite database for structured data
- JSON files for settings storage
- Serialization protocols for workspace state
- Layered configuration system
- Recovery mechanisms for state corruption

This layer focuses on "what" code implements the persistence aspects of project management.

## Key Transformation Points

### Concept: Workspace Structure
- **Stratospheric (Why)**: Users need organized projects with flexible layouts
- **Ground Level (How)**: Serialized workspace structure in database
- **Swift Implementation (What)**: Swift structs with Codable conformance for persistence

```swift
// Conceptual Swift implementation
struct SerializedWorkspace: Codable {
    let id: UUID
    let location: WorkspaceLocation
    let centerGroup: SerializedPaneGroup
    let windowBounds: WindowBounds?
    let centeredLayout: Bool
    let display: UUID?
    let docks: DockStructure
    let session: String?
    let breakpoints: [URL: [SourceBreakpoint]]
}

// Workspace state is loaded/saved using persistence service
class WorkspacePersistenceService {
    func saveWorkspace(_ workspace: Workspace) async throws {
        let serialized = workspace.serialized()
        try await database.write { db in
            try serialized.save(db)
        }
    }
    
    func loadWorkspace(id: UUID) async throws -> SerializedWorkspace? {
        try await database.read { db in
            try SerializedWorkspace.fetchOne(db, id: id)
        }
    }
}
```

### Concept: File System Access
- **Stratospheric (Why)**: Editor needs safe, consistent access to files with proper error handling
- **Ground Level (How)**: Trait-based abstraction with platform-specific implementations
- **Swift Implementation (What)**: Protocol-based system with platform optimizations

```swift
// Conceptual Swift implementation
protocol FileSystem {
    func createDirectory(at path: URL) async throws
    func createFile(at path: URL, options: CreateOptions) async throws
    func loadString(from path: URL) async throws -> String
    func loadData(from path: URL) async throws -> Data
    func atomicWrite(string: String, to path: URL) async throws
    func watch(path: URL, latency: TimeInterval) -> AsyncStream<[PathEvent]>
}

class RealFileSystem: FileSystem {
    func atomicWrite(string: String, to path: URL) async throws {
        let tempURL = FileManager.default.temporaryDirectory
            .appendingPathComponent(UUID().uuidString)
        
        try string.write(to: tempURL, atomically: true, encoding: .utf8)
        try FileManager.default.moveItem(at: tempURL, to: path)
    }
    
    // Other implementation methods...
}
```

### Concept: Settings Management
- **Stratospheric (Why)**: Projects need specific configuration separate from global settings
- **Ground Level (How)**: Layered settings system with cascading priorities
- **Swift Implementation (What)**: Property wrappers with custom resolution logic

```swift
// Conceptual Swift implementation
@propertyWrapper
struct Setting<T: Codable> {
    let key: String
    let defaultValue: T
    
    var wrappedValue: T {
        get {
            SettingsManager.shared.value(for: key, defaultValue: defaultValue)
        }
        set {
            SettingsManager.shared.setValue(newValue, for: key)
        }
    }
}

class SettingsManager {
    static let shared = SettingsManager()
    
    private var userSettings: [String: Any] = [:]
    private var projectSettings: [ProjectId: [String: Any]] = [:]
    
    func resolveValue<T: Codable>(for key: String, in project: ProjectId?) -> T? {
        // Check project settings first
        if let project = project, 
           let projectDict = projectSettings[project],
           let value = projectDict[key] as? T {
            return value
        }
        
        // Fall back to user settings
        return userSettings[key] as? T
    }
}
```

## Implementation Patterns

### Pattern: Virtual File System
This pattern appears across the layers but transforms:

```
Abstract File System Interface (Stratospheric)
       ↓
Platform-Specific File Adapters with Common Protocol (Ground)
       ↓
Swift Protocol with Platform Extensions (Implementation)
```

Swift implementation example:
```swift
protocol VirtualFileSystem {
    func readFile(at path: URL) async throws -> Data
    func writeFile(at path: URL, data: Data) async throws
    func watch(path: URL) -> AsyncStream<FileEvent>
}

// macOS implementation
class DarwinFileSystem: VirtualFileSystem {
    func watch(path: URL) -> AsyncStream<FileEvent> {
        AsyncStream { continuation in
            let eventStream = FSEventStreamCreate(/* parameters */)
            // Setup FSEvents monitoring
            continuation.onTermination = { FSEventStreamStop(eventStream) }
        }
    }
}
```

### Pattern: Project Tree
This pattern shows how project structure is modeled:

```
Concept: Hierarchical file representation
       ↓
Algorithm: Tree structure with lazy loading
       ↓
Implementation: Swift actors and value types
```

Swift implementation example:
```swift
actor ProjectTree {
    private var root: ProjectNode
    private var expandedNodes: Set<URL> = []
    
    func children(of path: URL) async throws -> [ProjectNode] {
        if let cachedNode = root.findNode(path: path),
           cachedNode.isLoaded {
            return cachedNode.children
        }
        
        // Load children from file system
        let fileSystem = FileSystemProvider.shared.fileSystem
        let entries = try await fileSystem.contentsOfDirectory(at: path)
        
        // Create nodes and update cache
        let nodes = entries.map { entry in
            ProjectNode(path: entry, parent: path)
        }
        
        root.update(path: path, children: nodes)
        return nodes
    }
}
```

## Codebase Connection

The bridge between conceptual and implementation layers connects to specific code in the Zed codebase:

- Project management concepts: `crates/project/src/project.rs`
- File system abstraction: `crates/fs/src/fs.rs`
- Settings system: `crates/settings/src/settings.rs`
- Swift implementation path: `/swift_prototype/Sources/ProjectManagement/`

## Swift-Specific Migration Challenges

This bridge document highlights several implementation challenges when moving from Rust to Swift:

1. **File System Abstraction**: Replacing Rust traits with Swift protocols
2. **Error Handling**: Adapting Rust's Result type to Swift's throwing functions
3. **Concurrency Model**: Moving from Rust's async/await to Swift's structured concurrency
4. **Path Handling**: Adapting path manipulation to use Swift's URL system
5. **Settings Storage**: Implementing type-safe settings with Swift's property system

## Practical Implementation Advice

When implementing the project management system in Swift:

1. **Start with core types**: 
   - `Project`, `Workspace`, `WorkTree`, `FileSystem`
   - Build protocol definitions for system boundaries
   - Implement concrete types for common use cases

2. **Implement file system layer**:
   - Create a platform-agnostic FileSystem protocol
   - Provide Darwin-specific optimizations for macOS
   - Implement file watching with FSEvents

3. **Build workspace model**:
   - Design serializable workspace structure
   - Implement pane management
   - Create project tree visualization

4. **Add settings persistence**:
   - Implement layered settings resolution
   - Create JSON serialization for settings
   - Build settings UI components

5. **Integrate Git support**:
   - Implement git command execution
   - Create status monitoring system
   - Build diff visualization components

## References
- [Swift documentation on FileManager](https://developer.apple.com/documentation/foundation/filemanager)
- [Swift documentation on Codable](https://developer.apple.com/documentation/swift/codable)
- [GRDB for Swift database operations](https://github.com/groue/GRDB.swift)
- [FSEvents API Reference](https://developer.apple.com/documentation/coreservices/file_system_events)
- [Zed Project codebase](https://github.com/zed-industries/zed)

---

*This bridge document is part of the Mission Cabbage documentation structure, designed to connect abstract concepts with concrete implementations.*