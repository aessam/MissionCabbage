# Cross-Cutting Concerns: Error Handling and Recovery Patterns

## Layer Information
**Document Type:** Cross-Cutting Concerns  
**Document ID:** 54  
**Focus Area:** Error Handling and Recovery  

## Connection Map
**Relates To:**
- [41_Swift_EntitySystem.md](41_Swift_EntitySystem.md)
- [42_Swift_ReactiveUI.md](42_Swift_ReactiveUI.md)
- [43_Swift_TextBuffer.md](43_Swift_TextBuffer.md)
- [44_Swift_ConcurrencyModel.md](44_Swift_ConcurrencyModel.md)
- [34_GroundLevel_MemoryManagement.md](34_GroundLevel_MemoryManagement.md)
- [39_GroundLevel_NetworkProtocols.md](39_GroundLevel_NetworkProtocols.md)
- [27_CloudLevel_TaskScheduling.md](27_CloudLevel_TaskScheduling.md)

## Purpose of This Document

This document provides a comprehensive strategy for error handling and recovery patterns across all subsystems of the Zed Swift implementation. It outlines consistent approaches to error propagation, user-facing error reporting, crash recovery, and system resilience that should be applied throughout the codebase.

## Core Error Handling Patterns

### 1. Error Propagation and Type System

Zed's Swift implementation uses a layered approach to error handling, moving from low-level specific errors to higher-level contextual errors:

**Swift Implementation:**
```swift
// Low-level error types with specific information
enum BufferError: Error {
    case indexOutOfBounds(index: Int, length: Int)
    case invalidUtf8(at: Int)
    case readOnly
    case concurrentModification
    case versionMismatch(expected: Int, actual: Int)
}

enum FileSystemError: Error {
    case notFound(path: String)
    case permissionDenied(path: String)
    case alreadyExists(path: String)
    case ioError(path: String, underlying: Error)
}

// Mid-level error aggregations with context
enum EditorError: Error {
    case buffer(BufferError)
    case filesystem(FileSystemError)
    case languageServer(LSPError)
    case userCancelled
    
    // Add context to errors as they move up the stack
    func withContext(_ message: String) -> EditorError {
        return .context(message, self)
    }
    
    case context(String, Error)
}

// High-level system errors for user-facing reporting
enum SystemError: Error, LocalizedError {
    case critical(Error, recoveryOptions: [RecoveryOption])
    case warning(Error)
    case background(Error)
    
    var errorDescription: String? {
        switch self {
        case .critical(let error, _):
            return "Critical error: \(error.localizedDescription)"
        case .warning(let error):
            return "Warning: \(error.localizedDescription)"
        case .background(let error):
            return "Background operation failed: \(error.localizedDescription)"
        }
    }
}

struct RecoveryOption {
    let title: String
    let action: () async throws -> Void
}
```

**Application Areas:**
- Text buffer operations
- File system access
- Network requests
- Entity system operations
- Language server communication

### 2. Result-Based Error Handling

For operations that can reasonably fail, Swift's `Result` type provides a clear pattern:

```swift
func findReferences(at position: BufferPosition) -> Result<[Reference], LSPError> {
    guard let languageServer = languageServer else {
        return .failure(.notInitialized)
    }
    
    do {
        let references = try languageServer.references(
            document: documentUri,
            position: position,
            context: ReferenceContext(includeDeclaration: true)
        )
        return .success(references)
    } catch let error as LSPError {
        return .failure(error)
    } catch {
        return .failure(.unknown(error))
    }
}

// Usage with pattern matching
switch findReferences(at: cursorPosition) {
case .success(let references):
    displayReferences(references)
case .failure(.notInitialized):
    showLanguageServerWarning()
case .failure(let error):
    logError(error)
    showGenericErrorMessage()
}
```

### 3. Swift Async/Await Error Handling

For asynchronous operations, Swift's built-in error handling with async/await provides structured error propagation:

```swift
func openProject(at path: URL) async throws -> Project {
    // Validate project first
    guard await FileManager.default.fileExists(at: path) else {
        throw FileSystemError.notFound(path: path.path)
    }
    
    // Open project files
    do {
        let manifest = try await loadManifestFile(at: path)
        let config = try await loadConfigurationFile(at: path)
        
        // Initialize the project
        return try await Project(
            path: path,
            manifest: manifest,
            configuration: config
        )
    } catch let error as DecodingError {
        // Transform and add context to JSON decoding errors
        throw ProjectError.invalidConfiguration(path: path.path, reason: error.localizedDescription)
    } catch {
        // Re-throw other errors with context
        throw ProjectError.failedToOpen(path: path.path, underlying: error)
    }
}

// Usage with async try/catch
Task {
    do {
        let project = try await openProject(at: selectedPath)
        await MainActor.run {
            windowManager.openNewWindow(for: project)
        }
    } catch let error as ProjectError {
        await MainActor.run {
            showProjectOpenError(error)
        }
    } catch {
        await MainActor.run {
            showGenericError(error)
        }
    }
}
```

### 4. Recoverable Error Handling

For errors that users can potentially recover from:

```swift
enum RecoveryAction {
    case retry
    case skipFile
    case cancelOperation
    case useAlternative(URL)
}

struct RecoverableError: LocalizedError {
    let underlyingError: Error
    let context: String
    let recoveryOptions: [RecoveryAction]
    
    var errorDescription: String? {
        return "\(context): \(underlyingError.localizedDescription)"
    }
    
    var recoverySuggestion: String? {
        var suggestions = [String]()
        if recoveryOptions.contains(.retry) {
            suggestions.append("You can try again.")
        }
        if recoveryOptions.contains(.skipFile) {
            suggestions.append("You can skip this file and continue.")
        }
        if recoveryOptions.contains(where: {
            if case .useAlternative(_) = $0 { return true } else { return false }
        }) {
            suggestions.append("You can use an alternative file.")
        }
        
        return suggestions.joined(separator: " ")
    }
}

// Usage example
func importProject(from directory: URL) async throws -> Project {
    do {
        return try await projectImporter.import(from: directory)
    } catch let error as FileSystemError where error.isAccessDenied {
        // Create a recoverable error with options
        let recoverableError = RecoverableError(
            underlyingError: error,
            context: "Cannot access project directory",
            recoveryOptions: [.retry, .useAlternative(URL(fileURLWithPath: "/")), .cancelOperation]
        )
        
        // Show UI and get user choice
        let recovery = await showRecoveryOptions(for: recoverableError)
        
        switch recovery {
        case .retry:
            return try await importProject(from: directory)
        case .useAlternative(let newDirectory):
            return try await importProject(from: newDirectory)
        case .cancelOperation:
            throw UserCancellationError()
        default:
            throw error
        }
    }
}
```

## Cross-Subsystem Error Handling Strategies

### 1. UI-Core Error Communication Pipeline

Errors need to flow from core systems to UI with proper context:

```
Core Subsystem Error → Error Transformation → Error Context Addition → UI-Ready Error
```

**Key Strategies:**
1. **Error Transformation**: Convert low-level errors to higher-level domain errors
2. **Context Enrichment**: Add contextual information as errors propagate
3. **Severity Classification**: Categorize errors by impact (critical, warning, info)
4. **Recovery Options**: Attach recovery options where applicable
5. **User Action Mapping**: Map technical errors to user-understandable actions

Example implementation:
```swift
protocol ErrorTransformer {
    func transform(_ error: Error) -> SystemError
}

class EditorErrorTransformer: ErrorTransformer {
    func transform(_ error: Error) -> SystemError {
        switch error {
        case let bufferError as BufferError:
            switch bufferError {
            case .concurrentModification:
                return .warning(EditorError.buffer(bufferError).withContext(
                    "Changes to the document couldn't be applied due to concurrent modifications"
                ))
            case .indexOutOfBounds:
                return .warning(EditorError.buffer(bufferError).withContext(
                    "An operation was attempted beyond the document's bounds"
                ))
            default:
                return .background(EditorError.buffer(bufferError))
            }
            
        case let fsError as FileSystemError:
            switch fsError {
            case .notFound:
                return .critical(
                    EditorError.filesystem(fsError).withContext("Document not found"),
                    recoveryOptions: [
                        RecoveryOption(title: "Create New File") {
                            // Create new file logic
                        },
                        RecoveryOption(title: "Browse for File") {
                            // File browser logic
                        }
                    ]
                )
            case .permissionDenied:
                return .critical(
                    EditorError.filesystem(fsError).withContext("Permission denied"),
                    recoveryOptions: [
                        RecoveryOption(title: "Open as Read-Only") {
                            // Read-only logic
                        },
                        RecoveryOption(title: "Retry with Elevated Permissions") {
                            // Permission elevation logic
                        }
                    ]
                )
            default:
                return .warning(EditorError.filesystem(fsError))
            }
            
        default:
            return .warning(error)
        }
    }
}
```

### 2. Crash Recovery and State Persistence

Crash recovery strategies that span multiple subsystems:

```
Regular State Snapshots → Disk Persistence → Crash Detection → Recovery Flow
```

**Key Strategies:**
1. **Incremental State Snapshots**: Regularly capture essential editor state
2. **Document History**: Track document edits with undo/redo capability
3. **Crash Detection**: Identify abnormal termination on next launch
4. **Recovery UI**: Present recovery options to users after crashes
5. **Partial Recovery**: Support recovering parts of state even if full recovery fails

Example implementation:
```swift
class CrashRecoveryManager {
    private let stateStore: StateStore
    private let crashDetector: CrashDetector
    
    init(stateStore: StateStore, crashDetector: CrashDetector) {
        self.stateStore = stateStore
        self.crashDetector = crashDetector
    }
    
    func checkForUnrecoveredState() async -> [RecoverableSession]? {
        guard await crashDetector.didCrashPreviously() else {
            return nil
        }
        
        do {
            let sessions = try await stateStore.getSavedSessions()
            if sessions.isEmpty {
                return nil
            }
            return sessions
        } catch {
            // Log but don't propagate - we don't want recovery errors to prevent app startup
            Logger.error("Failed to check for recoverable sessions: \(error)")
            return nil
        }
    }
    
    func recoverSession(_ session: RecoverableSession) async throws -> WorkspaceState {
        do {
            let workspace = try await stateStore.recoverSession(session)
            
            // Verify recovered documents
            for document in workspace.documents {
                try await document.validate()
            }
            
            return workspace
        } catch {
            // Try partial recovery if full recovery fails
            return try await partialRecover(session, error: error)
        }
    }
    
    private func partialRecover(_ session: RecoverableSession, error: Error) async throws -> WorkspaceState {
        // Logic to recover what we can from the session
        let partialRecovery = try await stateStore.partialRecoverSession(session)
        
        // Log what couldn't be recovered
        Logger.warning("Partial recovery performed. Some data couldn't be recovered: \(error)")
        
        return partialRecovery
    }
    
    func saveCurrentState(workspace: Workspace) async {
        do {
            try await stateStore.saveSession(from: workspace)
        } catch {
            Logger.error("Failed to save session state: \(error)")
            // We don't propagate this error - it's a background operation
        }
    }
}
```

## Swift-Specific Error Handling Considerations

### 1. Error Type Design Guidelines

Swift error handling benefits from a structured approach to error types:

| Error Category | Recommended Approach | Example |
|----------------|---------------------|---------|
| Low-level errors | Enum with associated values | `enum ParseError { case invalidToken(found: String, expected: String) }` |
| Domain errors | Protocol conformance + enums | `protocol DomainError: Error {}` <br> `enum EditorError: DomainError { ... }` |
| Cross-boundary errors | Error transformation | `func transformNetworkError(_ error: NetworkError) -> AppError` |
| User-facing errors | `LocalizedError` conformance | `extension SystemError: LocalizedError { var errorDescription: String? { ... } }` |

### 2. Swift Concurrency Error Handling

Swift's structured concurrency requires special error handling patterns:

1. **Task Group Error Aggregation**:
```swift
func processFiles(paths: [URL]) async throws -> [ProcessedFile] {
    try await withThrowingTaskGroup(of: ProcessedFile.self) { group in
        var results = [ProcessedFile]()
        var errors = [Error]()
        
        // Add tasks to group
        for path in paths {
            group.addTask {
                try await processFile(at: path)
            }
        }
        
        // Collect results, handling errors individually
        for try await result in group {
            results.append(result)
        }
        
        // If any errors occurred, throw an aggregate error
        if !errors.isEmpty {
            throw AggregateError(errors: errors)
        }
        
        return results
    }
}
```

2. **Cancellation Handling**:
```swift
func performLongRunningTask() async throws -> Result {
    // Check for cancellation at key points
    try Task.checkCancellation()
    
    // Set up cancellation handlers
    let cleanup = {
        // Release resources, save partial progress
    }
    
    return try await withTaskCancellationHandler(
        operation: {
            // Main operation that may take a long time
            let result = try await longOperation()
            try Task.checkCancellation()
            return result
        },
        onCancel: {
            // Run cleanup when cancelled
            cleanup()
        }
    )
}
```

3. **Actor Isolation Errors**:
```swift
actor StateManager {
    private var state: AppState
    
    func updateState(_ update: (inout AppState) throws -> Void) async throws {
        do {
            try update(&state)
        } catch {
            // Log the error within the actor
            Logger.error("State update failed: \(error)")
            throw StateError.updateFailed(underlying: error)
        }
    }
}

// Usage
do {
    try await stateManager.updateState { state in
        try state.applyChanges(from: newData)
    }
} catch let error as StateError {
    // Handle state-specific errors
} catch {
    // Handle other errors
}
```

### 3. Swift Optionals Strategy

Swift's optionals can be used strategically in error handling:

```swift
// For operations where nil is a valid business case, not an error
func findUserByUsername(_ username: String) async -> User? {
    // Returns nil if user doesn't exist (not an error)
    return await database.findUser(byUsername: username)
}

// For operations where absence is truly an error
func getUserSettings(for userId: UserId) async throws -> UserSettings {
    guard let settings = await database.getUserSettings(userId) else {
        throw UserError.settingsNotFound(userId: userId)
    }
    return settings
}

// Combining optionals with Result for richer error handling
func findUserPreferences(username: String) async -> Result<UserPreferences?, UserError> {
    // First get the user (nil is valid - user might not exist)
    guard let user = await findUserByUsername(username) else {
        return .success(nil) // Not an error, just no preferences because no user
    }
    
    // Now get preferences (could fail for various reasons)
    do {
        let preferences = try await getUserPreferences(userId: user.id)
        return .success(preferences)
    } catch let error as DatabaseError {
        return .failure(.databaseError(error))
    } catch {
        return .failure(.unknown(error))
    }
}
```

## Error Handling in UI Components

Error handling in UI requires specialized patterns:

### 1. Error Display Hierarchy

```swift
// Base protocol for error displaying components
protocol ErrorDisplaying {
    func display(error: SystemError)
    func clearError()
}

// ErrorView that can be embedded in various UI locations
struct ErrorView: View {
    let error: SystemError
    let recoveryHandler: (RecoveryOption) -> Void
    
    var body: some View {
        VStack(alignment: .leading) {
            Text(error.errorDescription ?? "An error occurred")
                .font(.headline)
            
            if let recoverySuggestion = error.recoverySuggestion {
                Text(recoverySuggestion)
                    .font(.subheadline)
            }
            
            // Show recovery options if available
            if case .critical(_, let options) = error {
                HStack {
                    ForEach(options, id: \.title) { option in
                        Button(option.title) {
                            recoveryHandler(option)
                        }
                        .buttonStyle(.bordered)
                    }
                }
            }
        }
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 8)
                .fill(errorBackgroundColor(for: error))
        )
    }
    
    private func errorBackgroundColor(for error: SystemError) -> Color {
        switch error {
        case .critical: return Color.red.opacity(0.1)
        case .warning: return Color.yellow.opacity(0.1)
        case .background: return Color.gray.opacity(0.1)
        }
    }
}

// Error manager that decides where and how to display errors
class ErrorManager: ObservableObject {
    @Published var activeErrors: [UUID: SystemError] = [:]
    
    func handle(error: Error, in context: ErrorContext) {
        let systemError = transform(error)
        let errorId = UUID()
        
        switch (systemError, context) {
        case (.critical, _):
            // Critical errors always show dialog
            showErrorDialog(systemError, id: errorId)
            
        case (.warning, .editor(let editorId)):
            // Editor warnings show in the editor
            showInlineError(systemError, in: editorId, id: errorId)
            
        case (.background, _):
            // Background errors show in notification area
            showNotificationError(systemError, id: errorId)
            
        default:
            // Default to notification area
            showNotificationError(systemError, id: errorId)
        }
        
        // Auto-dismiss non-critical errors
        if case .critical = systemError {} else {
            autoDismissError(id: errorId)
        }
    }
    
    private func transform(_ error: Error) -> SystemError {
        // Use the appropriate transformer based on error type
        if let systemError = error as? SystemError {
            return systemError
        }
        
        // Transform other error types
        // ...
        
        return .warning(error)
    }
    
    // Implementation details for different display methods...
}
```

### 2. Structured Error Recovery

```swift
class DocumentRecoveryController {
    private let document: TextDocument
    private let errorManager: ErrorManager
    
    init(document: TextDocument, errorManager: ErrorManager) {
        self.document = document
        self.errorManager = errorManager
    }
    
    func recoverFromSaveError(_ error: Error) async {
        // Transform the file system error
        let recoveryError = transformToRecoverableError(error)
        
        // Show options to user
        let selectedOption = await errorManager.presentRecoveryOptions(recoveryError)
        
        // Handle the user's selection
        switch selectedOption {
        case .saveAs:
            await handleSaveAs()
        case .discardChanges:
            await handleDiscard()
        case .retry:
            await handleRetrySave()
        case .cancel:
            // User cancelled, do nothing
            break
        }
    }
    
    private func transformToRecoverableError(_ error: Error) -> RecoverableError {
        // Transform based on the specific error
        if let fsError = error as? FileSystemError {
            switch fsError {
            case .permissionDenied:
                return RecoverableError(
                    underlyingError: fsError,
                    context: "Cannot save due to permission issues",
                    recoveryOptions: [.saveAs, .retry, .discardChanges, .cancel]
                )
            case .deviceFull:
                return RecoverableError(
                    underlyingError: fsError,
                    context: "Cannot save because device is full",
                    recoveryOptions: [.saveAs, .discardChanges, .cancel]
                )
            default:
                return RecoverableError(
                    underlyingError: fsError,
                    context: "Cannot save file",
                    recoveryOptions: [.saveAs, .retry, .discardChanges, .cancel]
                )
            }
        }
        
        // Default recovery error
        return RecoverableError(
            underlyingError: error,
            context: "Error saving document",
            recoveryOptions: [.saveAs, .retry, .discardChanges, .cancel]
        )
    }
    
    // Implementation of recovery handlers...
}
```

## Error Telemetry System

A comprehensive error tracking system aids in improving application quality:

```swift
protocol ErrorTelemetry {
    func recordError(_ error: Error, context: TelemetryContext)
    func recordRecoveryAttempt(for error: Error, action: RecoveryAction, succeeded: Bool)
    func uploadErrorReports() async
}

struct TelemetryContext {
    let subsystem: String
    let operation: String
    let userInitiated: Bool
    let additionalInfo: [String: String]
    
    init(
        subsystem: String,
        operation: String,
        userInitiated: Bool = false,
        additionalInfo: [String: String] = [:]
    ) {
        self.subsystem = subsystem
        self.operation = operation
        self.userInitiated = userInitiated
        self.additionalInfo = additionalInfo
    }
}

class ErrorTelemetryService: ErrorTelemetry {
    private let storage: TelemetryStorage
    private let networkService: TelemetryNetworkService
    private let userPreferences: UserPreferences
    
    init(storage: TelemetryStorage, networkService: TelemetryNetworkService, userPreferences: UserPreferences) {
        self.storage = storage
        self.networkService = networkService
        self.userPreferences = userPreferences
    }
    
    func recordError(_ error: Error, context: TelemetryContext) {
        guard userPreferences.telemetryEnabled else { return }
        
        Task {
            do {
                let report = try createErrorReport(error, context: context)
                try await storage.storeReport(report)
                
                // Attempt to upload immediately if online
                if await networkService.isOnline() {
                    try await uploadErrorReports()
                }
            } catch {
                // Log locally but don't propagate - telemetry shouldn't disrupt app operation
                Logger.error("Failed to record error telemetry: \(error)")
            }
        }
    }
    
    func recordRecoveryAttempt(for error: Error, action: RecoveryAction, succeeded: Bool) {
        guard userPreferences.telemetryEnabled else { return }
        
        Task {
            do {
                try await storage.recordRecoveryAttempt(
                    errorId: getErrorIdentifier(error),
                    action: action.identifier,
                    succeeded: succeeded
                )
            } catch {
                Logger.error("Failed to record recovery attempt: \(error)")
            }
        }
    }
    
    func uploadErrorReports() async {
        guard userPreferences.telemetryEnabled else { return }
        
        do {
            let reports = try await storage.getPendingReports()
            if reports.isEmpty { return }
            
            try await networkService.uploadReports(reports)
            
            // Mark as uploaded
            try await storage.markReportsAsUploaded(reports.map(\.id))
        } catch {
            Logger.error("Failed to upload error reports: \(error)")
        }
    }
    
    private func createErrorReport(_ error: Error, context: TelemetryContext) throws -> ErrorReport {
        // Implementation creates a sanitized error report
        // Filter out any PII or sensitive information
        // ...
    }
    
    private func getErrorIdentifier(_ error: Error) -> String {
        // Generate consistent ID for this error type
        // ...
        return "error-id"
    }
}
```

## Integration Testing for Error Handling

To verify cross-subsystem error handling, implement structured tests:

```swift
class ErrorHandlingTests: XCTestCase {
    
    func testFileSystemErrorPropagation() async throws {
        // Arrange
        let testPath = URL(fileURLWithPath: "/non-existent/path.txt")
        let fileSystem = MockFileSystem()
        fileSystem.simulateError(for: testPath, error: FileSystemError.notFound(path: testPath.path))
        
        let editor = Editor(fileSystem: fileSystem)
        let errorHandler = MockErrorHandler()
        
        // Act
        let result = await editor.openFile(at: testPath, errorHandler: errorHandler)
        
        // Assert
        XCTAssertNil(result)
        XCTAssertEqual(errorHandler.capturedErrors.count, 1)
        
        let capturedError = try XCTUnwrap(errorHandler.capturedErrors.first as? EditorError)
        if case .filesystem(let fsError) = capturedError {
            XCTAssertEqual(fsError.path, testPath.path)
        } else {
            XCTFail("Expected filesystem error but got \(capturedError)")
        }
    }
    
    func testCrashRecovery() async throws {
        // Arrange
        let crashDetector = MockCrashDetector(didCrash: true)
        let stateStore = MockStateStore()
        let testSession = RecoverableSession(id: "test-session", timestamp: Date())
        stateStore.savedSessions = [testSession]
        
        let recoveryManager = CrashRecoveryManager(
            stateStore: stateStore,
            crashDetector: crashDetector
        )
        
        // Act
        let sessions = try await recoveryManager.checkForUnrecoveredState()
        
        // Assert
        XCTAssertEqual(sessions?.count, 1)
        XCTAssertEqual(sessions?.first?.id, "test-session")
    }
    
    func testPartialRecovery() async throws {
        // Arrange
        let stateStore = MockStateStore()
        let testSession = RecoverableSession(id: "partial-session", timestamp: Date())
        stateStore.savedSessions = [testSession]
        
        // Simulate failure on full recovery but success on partial
        stateStore.simulateError(for: testSession.id, error: StateError.corruptData(path: "document1.txt"))
        
        let recoveryManager = CrashRecoveryManager(
            stateStore: stateStore,
            crashDetector: MockCrashDetector(didCrash: true)
        )
        
        // Act
        let workspace = try await recoveryManager.recoverSession(testSession)
        
        // Assert
        XCTAssertTrue(workspace.partialRecovery)
        XCTAssertEqual(workspace.documents.count, stateStore.partialRecoveryDocumentCount)
    }
}
```

## Conclusion

Error handling and recovery in Zed's Swift implementation must be a comprehensive cross-cutting concern that ensures a robust, user-friendly experience even when things go wrong. By applying these consistent patterns throughout the codebase, we can create a system that gracefully handles errors, provides meaningful feedback to users, recovers from crashes, and maintains system integrity.

The key principles to follow are:
1. Use Swift's type system for clear error typing and propagation
2. Transform errors as they move up the stack to add context
3. Implement meaningful recovery options when possible
4. Persist state to enable crash recovery
5. Provide appropriate UI feedback based on error severity
6. Track errors to improve system quality over time

## References
- [Swift Error Handling](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/errorhandling/)
- [Swift Concurrency](https://developer.apple.com/videos/play/wwdc2021/10132/)
- [Swift Structured Concurrency: Error Handling](https://developer.apple.com/videos/play/wwdc2021/10134/)
- `crates/text/src/buffer.rs`: Rust error handling patterns in the text buffer
- `crates/workspace/src/persistence.rs`: Session persistence in Zed

---

*This document is part of the Mission Cabbage documentation structure, addressing cross-cutting error handling concerns.*