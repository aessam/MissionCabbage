# Swift Extension System for Zed

This document describes how to implement a Swift-native extension system that provides functionality similar to Zed's Rust extension system, focusing on security, flexibility, and ease of development.

## Overview of Zed's Extension System

Zed implements an extension system with these key characteristics:

1. **Sandboxed Extensions**: Extensions run in WebAssembly (WASM) sandboxes for security and isolation
2. **Declarative Manifest**: Extensions declare capabilities and metadata in a structured manifest
3. **Well-defined API**: Extensions interact with the editor through a stable, versioned API
4. **Multiple Extension Types**: Support for language servers, slash commands, themes, etc.
5. **Resource Limitations**: Extensions have controlled access to system resources

## Swift Implementation Design

### 1. Core Components

#### ExtensionManifest

```swift
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

struct LanguageServerDefinition: Codable {
    let languages: [String]
    let command: String?
    let args: [String]?
    let env: [String: String]?
}

struct SlashCommandDefinition: Codable {
    let description: String
    let requiresArgument: Bool
}
```

#### Extension Protocol

```swift
protocol Extension: AnyObject {
    // Extension identity
    var manifest: ExtensionManifest { get }
    var workDirectory: URL { get }
    
    // Language server support
    func languageServerCommand(
        for languageServerId: LanguageServerId, 
        in worktree: Worktree
    ) async throws -> Command
    
    func languageServerInitializationOptions(
        for languageServerId: LanguageServerId, 
        in worktree: Worktree
    ) async throws -> JSON?
    
    func languageServerWorkspaceConfiguration(
        for languageServerId: LanguageServerId, 
        in worktree: Worktree
    ) async throws -> JSON?
    
    // Slash commands
    func completeSlashCommandArgument(
        command: SlashCommand, 
        arguments: [String]
    ) async throws -> [SlashCommandArgumentCompletion]
    
    func runSlashCommand(
        command: SlashCommand, 
        arguments: [String], 
        worktree: Worktree?
    ) async throws -> SlashCommandOutput
    
    // Context servers
    func contextServerCommand(
        for contextServerId: ContextServerId, 
        in project: Project
    ) async throws -> Command
    
    func contextServerConfiguration(
        for contextServerId: ContextServerId, 
        in project: Project
    ) async throws -> ContextServerConfiguration?
    
    // Documentation support
    func suggestDocsPackages(for provider: String) async throws -> [String]
    
    func indexDocs(
        provider: String, 
        package: String, 
        database: KeyValueStore
    ) async throws
}
```

#### Extension Host

```swift
class ExtensionHost {
    private let extensions: [String: ManagedExtension] = [:]
    private let extensionManager: ExtensionManager
    private let fileSystem: FileSystem
    private let httpClient: HTTPClient
    
    init(fileSystem: FileSystem, httpClient: HTTPClient) {
        self.fileSystem = fileSystem
        self.httpClient = httpClient
        self.extensionManager = ExtensionManager(fileSystem: fileSystem)
    }
    
    func loadExtension(id: String) async throws -> Extension {
        guard let managedExtension = extensions[id] else {
            let newExtension = try await extensionManager.loadExtension(id: id)
            extensions[id] = newExtension
            return newExtension.extension
        }
        
        return managedExtension.extension
    }
}
```

### 2. Sandboxing Architecture

Swift's extension system will use platform-native sandboxing technologies:

```swift
protocol ExtensionSandbox {
    func start() async throws
    func stop() async
    var isRunning: Bool { get }
    
    func execute<Request: Encodable, Response: Decodable>(
        method: String,
        params: Request
    ) async throws -> Response
}

// macOS implementation using XPC
class XPCExtensionSandbox: ExtensionSandbox {
    private let connection: NSXPCConnection
    private let extensionBundle: Bundle
    private let manifest: ExtensionManifest
    
    init(extensionBundle: Bundle, manifest: ExtensionManifest) {
        self.extensionBundle = extensionBundle
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
        // Initialize extension...
    }
    
    // Other implementation details...
}
```

### 3. Extension Registration and Loading

```swift
class ExtensionManager {
    private let fileSystem: FileSystem
    private let extensionDirectories: [URL]
    
    init(fileSystem: FileSystem) {
        self.fileSystem = fileSystem
        
        // Set up extension directories (app bundle, user Library, etc.)
        self.extensionDirectories = [
            Bundle.main.bundleURL.appendingPathComponent("Extensions"),
            FileManager.default.homeDirectoryForCurrentUser
                .appendingPathComponent("Library/Application Support/Zed/Extensions")
        ]
    }
    
    func discoverExtensions() async throws -> [ExtensionManifest] {
        var manifests: [ExtensionManifest] = []
        
        for directory in extensionDirectories {
            guard let contents = try? fileSystem.contentsOfDirectory(at: directory) else {
                continue
            }
            
            for itemURL in contents {
                if let manifest = try? await loadManifest(from: itemURL) {
                    manifests.append(manifest)
                }
            }
        }
        
        return manifests
    }
    
    func loadExtension(id: String) async throws -> ManagedExtension {
        // Find extension directory
        guard let extensionDir = try await findExtensionDirectory(for: id) else {
            throw ExtensionError.extensionNotFound(id)
        }
        
        // Load manifest
        let manifest = try await loadManifest(from: extensionDir)
        
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
        
        // Return managed extension
        return ManagedExtension(extension: proxy, sandbox: sandbox)
    }
}
```

### 4. Swift Extension API

```swift
// The API exposed to extension developers
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
    
    // Other extension points...
}

// Extensions would implement their functionality like this
public class MyExtension: ZedExtension {
    public static func create() -> Self {
        return Self()
    }
    
    public func languageServerCommand(id: String, worktree: WorktreeInfo) async throws -> CommandInfo {
        return CommandInfo(
            command: "/usr/bin/node",
            args: ["path/to/language-server.js", "--stdio"]
        )
    }
    
    public func runSlashCommand(
        command: String, 
        arguments: [String], 
        worktree: WorktreeInfo?
    ) async throws -> SlashCommandResult {
        switch command {
        case "hello":
            return SlashCommandResult(
                text: "Hello, \(arguments.joined(separator: " "))!",
                sections: [
                    SlashCommandSection(
                        range: 0..<arguments.joined(separator: " ").count + 8,
                        label: "Greeting"
                    )
                ]
            )
        default:
            throw ZedExtensionError.commandNotImplemented
        }
    }
}

// Entry point annotation for Swift extensions
@_cdecl("zed_extension_main")
public func extensionMain() -> UnsafeMutableRawPointer {
    // Register the extension and return handle
    let extensionInstance = MyExtension.create()
    return ExtensionRuntime.shared.register(extension: extensionInstance)
}
```

### 5. Security Model

```swift
// Define capabilities that extensions can request
enum ExtensionCapability: Codable {
    case fileSystemRead(path: String)
    case fileSystemWrite(path: String)
    case networkAccess(domains: [String])
    case processExecution(commandPattern: String)
    case clipboardAccess
}

// Security validator
class ExtensionSecurityValidator {
    func validateCapabilities(
        _ capabilities: [ExtensionCapability], 
        forExtension id: String
    ) -> [ExtensionCapability] {
        // Filter out capabilities based on user preferences and security policies
        return capabilities.filter { capability in
            switch capability {
            case .fileSystemRead(let path):
                return isPathAllowed(path, forExtension: id, access: .read)
            case .fileSystemWrite(let path):
                return isPathAllowed(path, forExtension: id, access: .write)
            case .networkAccess(let domains):
                return areDomainsAllowed(domains, forExtension: id)
            case .processExecution(let pattern):
                return isCommandPatternAllowed(pattern, forExtension: id)
            case .clipboardAccess:
                return isClipboardAccessAllowed(forExtension: id)
            }
        }
    }
    
    // Implementation of security checks...
}
```

### 6. Resource Proxies and Delegates

```swift
// Worktree delegate
protocol WorktreeDelegate: AnyObject {
    var id: UInt64 { get }
    var rootPath: String { get }
    
    func readTextFile(at path: URL) async throws -> String
    func which(binaryName: String) async throws -> String?
    func shellEnvironment() async throws -> [String: String]
}

// Project delegate
protocol ProjectDelegate: AnyObject {
    var worktreeIds: [UInt64] { get }
}

// KeyValueStore delegate for document indexing
protocol KeyValueStoreDelegate: AnyObject {
    func insert(key: String, document: String) async throws
}
```

## Extension Packaging and Distribution

### 1. Extension Bundle Format

For macOS:

```
MyExtension.zedext/
  ├── Info.plist             # Bundle info and security declarations
  ├── manifest.json          # Extension manifest
  ├── extension.swift        # Main extension code (compiled)
  ├── Resources/             # Extension resources
  │   ├── themes/            # Optional theme files
  │   ├── icons/             # Optional icon files
  │   ├── languages/         # Optional language definitions
  │   └── ...
  └── _CodeSignature/        # Code signature (App Store distribution)
```

### 2. Distribution Methods

1. **App Store**: Extensions distributed through the Mac App Store
2. **Direct Download**: Extensions packaged as `.zedext` bundles
3. **Extension Library**: Built-in extension catalog within the editor
4. **Developer Mode**: Local extension development with hot reloading

## Migration Path for Existing Extensions

For Zed's existing Rust/WASM extensions, we can provide:

1. A compatibility layer that runs the original WASM extensions
2. Tools to assist in porting extensions to native Swift
3. Documentation for converting common extension patterns

## Swift Implementation Considerations

### 1. Concurrency Model

```swift
// Using Swift's structured concurrency
class ExtensionProxy: Extension {
    private let sandbox: ExtensionSandbox
    private let manifest: ExtensionManifest
    
    func languageServerCommand(
        for languageServerId: LanguageServerId, 
        in worktree: Worktree
    ) async throws -> Command {
        // Convert to serializable types
        let request = LanguageServerCommandRequest(
            languageServerId: languageServerId.rawValue,
            worktree: worktree.toSerializable()
        )
        
        // Execute in sandbox with timeout
        let response: LanguageServerCommandResponse = try await Task.timeout(seconds: 5) {
            try await sandbox.execute(method: "languageServerCommand", params: request)
        }
        
        // Convert response
        return Command(
            command: response.command,
            args: response.args,
            env: response.env
        )
    }
}
```

### 2. Memory Management

```swift
// Extension instance manager ensures proper lifecycle management
class ManagedExtension {
    let `extension`: Extension
    private let sandbox: ExtensionSandbox
    private var isActive: Bool = true
    
    init(extension: Extension, sandbox: ExtensionSandbox) {
        self.extension = `extension`
        self.sandbox = sandbox
    }
    
    func deactivate() async {
        if isActive {
            await sandbox.stop()
            isActive = false
        }
    }
    
    deinit {
        Task {
            await deactivate()
        }
    }
}
```

### 3. Error Handling

```swift
enum ExtensionError: Error {
    case extensionNotFound(String)
    case invalidManifest(String)
    case sandboxInitializationFailed(String)
    case methodCallFailed(String, underlying: Error)
    case responseDecodingFailed(String)
    case capabilityDenied(ExtensionCapability)
    case timeout(operation: String)
}

extension ExtensionError: LocalizedError {
    var errorDescription: String? {
        switch self {
        case .extensionNotFound(let id):
            return "Extension '\(id)' not found"
        case .invalidManifest(let id):
            return "Invalid manifest for extension '\(id)'"
        case .sandboxInitializationFailed(let reason):
            return "Failed to initialize extension sandbox: \(reason)"
        case .methodCallFailed(let method, let error):
            return "Extension method '\(method)' failed: \(error.localizedDescription)"
        case .responseDecodingFailed(let method):
            return "Failed to decode response from '\(method)'"
        case .capabilityDenied(let capability):
            return "Extension capability denied: \(capability)"
        case .timeout(let operation):
            return "Extension operation timed out: \(operation)"
        }
    }
}
```

### 4. Performance Considerations

1. **Extension Loading**: Load extensions asynchronously and on-demand
2. **Communication Overhead**: Use binary serialization for faster IPC
3. **Resource Caching**: Cache extension resources to reduce disk I/O
4. **Idle Extensions**: Unload inactive extensions to save memory
5. **Startup Impact**: Defer extension loading until needed

```swift
class ExtensionPerformanceMonitor {
    private var extensionTimings: [String: [OperationTiming]] = [:]
    
    func recordOperation(
        extensionId: String,
        operation: String,
        duration: TimeInterval
    ) {
        let timing = OperationTiming(
            operation: operation,
            duration: duration,
            timestamp: Date()
        )
        
        extensionTimings[extensionId, default: []].append(timing)
        
        // Log slow operations
        if duration > 0.5 {
            Logger.shared.warning("Slow extension operation: \(extensionId).\(operation) took \(duration)s")
        }
    }
    
    func getExtensionMetrics(extensionId: String) -> ExtensionMetrics {
        let timings = extensionTimings[extensionId] ?? []
        
        return ExtensionMetrics(
            averageOperationTime: timings.map(\.duration).reduce(0, +) / Double(timings.count),
            operationCount: timings.count,
            slowOperationCount: timings.filter { $0.duration > 0.5 }.count
        )
    }
}
```

## Example: Implementing a Simple Extension

```swift
import ZedExtensionKit

// Extension metadata and registration
@ZedExtension(
    id: "com.example.hello-world",
    name: "Hello World",
    version: "1.0.0",
    description: "A simple example extension"
)
class HelloWorldExtension {
    // State
    private var greetingCount = 0
    
    func initialize() {
        // Set up resources or state
        print("Hello World extension initialized")
    }
    
    // Slash command implementation
    @SlashCommand(
        name: "hello",
        description: "Says hello to someone",
        requiresArgument: true
    )
    func helloCommand(arguments: [String], worktree: Worktree?) throws -> SlashCommandOutput {
        greetingCount += 1
        
        let name = arguments.isEmpty ? "World" : arguments.joined(separator: " ")
        let greeting = "Hello, \(name)! (Greeted \(greetingCount) times)"
        
        return SlashCommandOutput(
            text: greeting,
            sections: [
                SlashCommandSection(
                    range: 0..<greeting.count,
                    label: "Greeting"
                )
            ]
        )
    }
    
    @SlashCommandCompletion(name: "hello")
    func completeHelloCommand(arguments: [String]) -> [SlashCommandCompletion] {
        return [
            SlashCommandCompletion(label: "World", newText: "World", runCommand: true),
            SlashCommandCompletion(label: "Friend", newText: "Friend", runCommand: true),
            SlashCommandCompletion(label: "Zed", newText: "Zed", runCommand: true)
        ]
    }
}
```

## Conclusion

Implementing Zed's extension system in Swift involves several key components:

1. A formal extension API with clear protocols and types
2. Native sandboxing using platform-specific technologies (XPC/App Extensions)
3. Resource management and access controls for security
4. Native Swift extension APIs with structured concurrency
5. Extension packaging and distribution methods

By adapting Zed's WASM-based system to Swift's native capabilities, we can create a more deeply integrated extension system that offers improved performance and platform-specific advantages while maintaining the necessary security guarantees.

The system also provides a clear path for migrating existing extensions and creating new ones, with Swift's strong type safety helping to prevent common errors in extension development.