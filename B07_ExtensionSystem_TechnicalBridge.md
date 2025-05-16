# Extension System: From Concept to Implementation

## Layer Information
**Bridge Document**  
**Document ID:** B07  
**Focus Area:** Extension System & Sandboxing  

## Connection Map
**Bridges Between:**
- Stratospheric: [07_StratosphericView_ExtensionSystem.md](07_StratosphericView_ExtensionSystem.md)
- Cloud Level: [29_CloudLevel_ExtensionSandbox.md](29_CloudLevel_ExtensionSandbox.md)
- Swift Level: [45_Swift_ExtensionSystem.md](45_Swift_ExtensionSystem.md)

## Purpose of This Bridge Document

This bridge document connects the high-level architectural concepts of Zed's Extension System with its concrete implementation details. It explains how abstract concepts around extension management, sandboxing, and API surface are realized in specific code structures, focusing on the transition from Rust's WebAssembly-based approach to Swift's native capabilities.

## Tracing the Extension System Through Layers

### Layer 1: Stratospheric View (Extension System)
At this layer, the extension system is described as an architectural framework providing:

- Support for third-party extension development
- Secure sandboxing for extension code
- Well-defined API surface for extensions
- Extension discovery and installation
- Configuration and settings management

This layer focuses on "why" the extension system exists and its role in the architecture.

### Layer 2: Cloud Level (Extension Sandbox)
At this layer, the sandboxing mechanisms are detailed:

- WebAssembly (Wasm) as a secure execution environment
- Resource handles for safe object referencing
- API boundaries using WebAssembly Interface Types (WIT)
- Permission model for capability-based security
- Extension manifest format and validation

This layer focuses on "how" the extension sandboxing works internally.

### Layer 3: Swift Level (Swift Extension System)
At this layer, the Swift-specific implementation approach is specified:

- Native sandboxing using XPC and App Extensions
- Swift protocols for extension interfaces
- Codable for manifest and configuration
- Swift Concurrency for asynchronous extension operations
- Swift type safety for API boundaries

This layer focuses on "what" Swift code implements the extension system.

## Key Transformation Points

### Concept: Extension Sandboxing
- **Stratospheric (Why)**: Extensions need isolation for security and stability
- **Cloud Level (How)**: WebAssembly modules with memory isolation
- **Swift Level (What)**: XPC services with entitlement restrictions

```swift
// Conceptual Swift implementation
class XPCExtensionSandbox: ExtensionSandbox {
    private let connection: NSXPCConnection
    private let manifest: ExtensionManifest
    
    init(extensionBundle: Bundle, manifest: ExtensionManifest) {
        self.manifest = manifest
        
        // Set up XPC connection with appropriate entitlements
        self.connection = NSXPCConnection(serviceName: manifest.id)
        
        // Configure interface and security
        connection.remoteObjectInterface = NSXPCInterface(with: ExtensionServiceProtocol.self)
        connection.exportedInterface = NSXPCInterface(with: HostServiceProtocol.self)
        
        // Set up security restrictions based on manifest capabilities
        let restrictions = setupSandboxRestrictions(for: manifest.capabilities)
        connection.sandboxRestrictions = restrictions
    }
    
    func start() async throws {
        connection.resume()
        // Initialize connection and validate extension
    }
    
    func execute<Request: Encodable, Response: Decodable>(
        method: String,
        params: Request
    ) async throws -> Response {
        try await withCheckedThrowingContinuation { continuation in
            let proxy = connection.remoteObjectProxyWithErrorHandler { error in
                continuation.resume(throwing: error)
            }
            
            (proxy as? ExtensionServiceProtocol)?.execute(
                method: method,
                params: try! JSONEncoder().encode(params)
            ) { responseData, error in
                if let error = error {
                    continuation.resume(throwing: error)
                } else if let data = responseData {
                    do {
                        let response = try JSONDecoder().decode(Response.self, from: data)
                        continuation.resume(returning: response)
                    } catch {
                        continuation.resume(throwing: error)
                    }
                } else {
                    continuation.resume(throwing: ExtensionError.noResponse)
                }
            }
        }
    }
}
```

### Concept: Extension API
- **Stratospheric (Why)**: Extensions need a stable interface to interact with the editor
- **Cloud Level (How)**: WIT interface definitions with version compatibility
- **Swift Level (What)**: Swift protocols with version-specific extensions

```swift
// Conceptual Swift implementation
@objc public protocol ExtensionServiceProtocol {
    func execute(method: String, params: Data, completion: @escaping (Data?, Error?) -> Void)
    func initialize(withOptions options: Data, completion: @escaping (Bool, Error?) -> Void)
    func shutdown(completion: @escaping (Error?) -> Void)
}

@objc public protocol HostServiceProtocol {
    func readFile(path: String, completion: @escaping (Data?, Error?) -> Void)
    func writeFile(path: String, content: Data, completion: @escaping (Error?) -> Void)
    func listFiles(directory: String, completion: @escaping ([String]?, Error?) -> Void)
    func executeCommand(command: String, args: [String], completion: @escaping (Int, Data, Data, Error?) -> Void)
    func showNotification(message: String, type: Int, completion: @escaping (Error?) -> Void)
}

// Extension developer API
public protocol ZedExtension {
    // Initialization
    static func create() -> Self
    
    // Language server methods
    func languageServerCommand(
        id: String, 
        worktree: WorktreeInfo
    ) async throws -> CommandInfo
    
    func languageServerInitializationOptions(
        id: String, 
        worktree: WorktreeInfo
    ) async throws -> Any?
    
    // Slash command methods
    func completeSlashCommandArgument(
        command: String, 
        arguments: [String]
    ) async throws -> [SlashCommandCompletion]
    
    func runSlashCommand(
        command: String, 
        arguments: [String], 
        worktree: WorktreeInfo?
    ) async throws -> SlashCommandResult
}
```

### Concept: Extension Manifests
- **Stratospheric (Why)**: Extensions need to declare capabilities and metadata
- **Cloud Level (How)**: Structured JSON manifest with capability declarations
- **Swift Level (What)**: Swift Codable types with validation

```swift
// Conceptual Swift implementation
struct ExtensionManifest: Codable {
    let id: String
    let name: String
    let description: String
    let version: SemanticVersion
    let authors: [String]
    let repository: URL?
    
    // Extension capabilities
    let languageServers: [String: LanguageServerDefinition]?
    let slashCommands: [String: SlashCommandDefinition]?
    let themes: [ThemeDefinition]?
    let iconThemes: [IconThemeDefinition]?
    let languages: [LanguageDefinition]?
    let contextServers: [String: ContextServerDefinition]?
    
    // Security and permissions
    let capabilities: [ExtensionCapability]
}

enum ExtensionCapability: Codable {
    case fileSystemRead(path: String)
    case fileSystemWrite(path: String)
    case networkAccess(domains: [String])
    case processExecution(commandPattern: String)
    case clipboardAccess
}

// Validation extensions
extension ExtensionManifest {
    func validate() throws {
        // Ensure ID is valid
        guard id.matches(regex: "^[a-z][a-z0-9_\\-\\.]+$") else {
            throw ExtensionError.invalidManifest("Invalid extension ID format")
        }
        
        // Validate version
        guard version.isValid else {
            throw ExtensionError.invalidManifest("Invalid version format")
        }
        
        // Validate capabilities
        for capability in capabilities {
            try validateCapability(capability)
        }
    }
    
    private func validateCapability(_ capability: ExtensionCapability) throws {
        switch capability {
        case .fileSystemRead(let path):
            guard !path.contains("..") else {
                throw ExtensionError.invalidCapability("Path traversal not allowed")
            }
        case .processExecution(let pattern):
            guard !pattern.contains(";") && !pattern.contains("&&") else {
                throw ExtensionError.invalidCapability("Command injection patterns not allowed")
            }
        default:
            break
        }
    }
}
```

## Implementation Patterns

### Pattern: Resource Handles
This pattern appears across the layers but transforms:

```
Concept: Opaque handles to prevent direct resource access
       ↓
Algorithm: Resource table with capability checks
       ↓
Implementation: Swift protocol witnesses with type erasure
```

Swift implementation example:
```swift
// Type-erased resource identifier
struct ResourceHandle: Hashable, Sendable {
    let id: UInt32
    let type: ResourceType
    
    enum ResourceType: String, Codable {
        case document
        case terminal
        case project
        case languageServer
    }
}

// Resource table implementation
actor ResourceTable {
    private var resources: [ResourceHandle: Any] = [:]
    private var nextId: UInt32 = 1
    
    func add<T>(_ resource: T, type: ResourceHandle.ResourceType) -> ResourceHandle {
        let handle = ResourceHandle(id: nextId, type: type)
        nextId += 1
        resources[handle] = resource
        return handle
    }
    
    func get<T>(_ handle: ResourceHandle) throws -> T {
        guard let resource = resources[handle] else {
            throw ExtensionError.resourceNotFound(handle)
        }
        
        guard let typedResource = resource as? T else {
            throw ExtensionError.resourceTypeMismatch(handle)
        }
        
        return typedResource
    }
    
    func remove(_ handle: ResourceHandle) {
        resources.removeValue(forKey: handle)
    }
}
```

### Pattern: Extension Lifecycle
This pattern shows how extension activation and deactivation is managed:

```
Concept: Controlled extension loading and unloading
       ↓
Algorithm: Lifecycle state management with initialization phases
       ↓
Implementation: Swift actors and structured concurrency
```

Swift implementation example:
```swift
class ExtensionManager {
    private var extensions: [String: ManagedExtension] = [:]
    
    func loadExtension(id: String) async throws -> Extension {
        if let existing = extensions[id] {
            return existing.extension
        }
        
        // Find extension directory
        guard let extensionDir = try await findExtensionDirectory(for: id) else {
            throw ExtensionError.extensionNotFound(id)
        }
        
        // Load manifest
        let manifest = try await loadManifest(from: extensionDir)
        
        // Validate manifest
        try manifest.validate()
        
        // Check compatibility
        try checkCompatibility(manifest: manifest)
        
        // Create sandbox based on platform
        let sandbox: ExtensionSandbox
        
        #if os(macOS)
        sandbox = XPCExtensionSandbox(extensionDir: extensionDir, manifest: manifest)
        #elseif os(iOS)
        sandbox = AppExtensionSandbox(extensionDir: extensionDir, manifest: manifest)
        #endif
        
        // Start sandbox
        try await sandbox.start()
        
        // Create proxy
        let proxy = ExtensionProxy(sandbox: sandbox, manifest: manifest)
        
        // Initialize extension
        try await proxy.initialize()
        
        // Store managed extension
        let managed = ManagedExtension(extension: proxy, sandbox: sandbox)
        extensions[id] = managed
        
        return proxy
    }
    
    func unloadExtension(id: String) async throws {
        guard let managed = extensions[id] else {
            return
        }
        
        try await managed.deactivate()
        extensions.removeValue(forKey: id)
    }
}

class ManagedExtension {
    let `extension`: Extension
    private let sandbox: ExtensionSandbox
    private var isActive: Bool = true
    
    init(extension: Extension, sandbox: ExtensionSandbox) {
        self.extension = `extension`
        self.sandbox = sandbox
    }
    
    func deactivate() async throws {
        if isActive {
            try await (self.extension as? ExtensionProxy)?.shutdown()
            await sandbox.stop()
            isActive = false
        }
    }
}
```

## Codebase Connection

The bridge between conceptual and implementation layers connects to specific code in the Zed codebase:

- Extension system concepts: `crates/extension/src/extension.rs`
- Extension API: `crates/extension_api/src/lib.rs`
- Extension host: `crates/extension_host/src/extension_host.rs`
- Swift implementation path: `/swift_prototype/Sources/Extension/`

## Swift-Specific Migration Challenges

This bridge document highlights several implementation challenges when moving from Rust's WebAssembly-based extension system to Swift:

1. **Sandboxing Mechanism**: Replacing WebAssembly with XPC/App Extensions
2. **API Design**: Converting WIT interfaces to Swift protocols
3. **Serialization**: Moving from WIT to Swift's Codable
4. **Concurrency Model**: Adapting to Swift's structured concurrency
5. **Security Model**: Implementing equivalent capability controls

## Practical Implementation Advice

When implementing the extension system in Swift:

1. **Start with core types**:
   - `ExtensionManifest`: Define the manifest structure
   - `ExtensionHost`: Create the main host controller
   - `ExtensionSandbox`: Implement the sandbox abstraction
   - `ResourceTable`: Build the resource management system

2. **Implement sandboxing**:
   - Create XPC service infrastructure
   - Define security boundaries and entitlements
   - Implement proper message serialization

3. **Build protocol layer**:
   - Define Swift protocols for extension API
   - Create protocol delegates for callbacks
   - Implement versioning strategy

4. **Create extension SDK**:
   - Build developer-facing API
   - Create helper types and convenience methods
   - Document extension development process

5. **Add extension management**:
   - Implement discovery and installation
   - Create activation/deactivation lifecycle
   - Build settings and configuration UI

## References
- [Apple Documentation on XPC](https://developer.apple.com/documentation/xpc)
- [Swift Documentation on Codable](https://developer.apple.com/documentation/swift/codable)
- [Apple App Extension Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/index.html)
- [Structured Concurrency in Swift](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html)
- [Zed Extension codebase](https://github.com/zed-industries/zed)

---

*This bridge document is part of the Mission Cabbage documentation structure, designed to connect abstract concepts with concrete implementations.*