# Stratospheric View: Language Intelligence

## Purpose

The Language Intelligence system provides code-aware features that enhance the editing experience. It understands code structure, semantics, and conventions to offer syntax highlighting, code completion, error detection, navigation, and refactoring capabilities.

## Core Concepts

### Tree-sitter Integration

- **Grammar**: Language-specific parsing rules
- **Syntax Tree**: Structural representation of code
- **Incremental Parsing**: Efficient updates as code changes
- **Query System**: Pattern matching against the syntax tree
- **Node Types**: Classification of syntax elements

### Language Server Protocol (LSP)

- **LSP Client**: Communication with language servers
- **Request/Response**: Protocol for code intelligence queries
- **Diagnostics**: Errors, warnings, and hints
- **Code Actions**: Automated fixes and refactorings
- **Symbols**: Named elements in code

### Syntax Highlighting

- **Highlighting Rules**: Language-specific coloring rules
- **Scopes**: Classification of code elements
- **Themes**: Visual styling configurations
- **Token Types**: Categories of syntax elements
- **Semantic Tokens**: Language-server provided highlighting

### Code Intelligence

- **Completion**: Suggestions as you type
- **Hover**: Documentation on hover
- **Go to Definition**: Navigate to symbol definitions
- **Find References**: Locate all uses of a symbol
- **Rename**: Safely rename symbols across files
- **Signature Help**: Function parameter information

### Document Analysis

- **Outline**: Structural overview of a document
- **Folding**: Code folding based on structure
- **Indentation**: Smart indentation rules
- **Document Symbols**: Navigable document structure
- **Semantic Analysis**: Deeper understanding of code

## Architecture

### Core Components

1. **Language Registry**
   - Tracks supported languages
   - Maps file types to languages
   - Manages language-specific configuration

2. **Tree-sitter Engine**
   - Loads and manages Tree-sitter grammars
   - Maintains syntax trees for buffers
   - Executes queries against syntax trees
   - Provides incremental parsing

3. **LSP Manager**
   - Starts and manages language server processes
   - Routes requests to appropriate servers
   - Handles response callbacks
   - Manages server lifecycle

4. **Highlighter**
   - Applies syntax highlighting rules
   - Combines Tree-sitter and LSP highlighting
   - Maps syntax to theme colors
   - Maintains highlight state

5. **Intelligence Provider**
   - Coordinates code intelligence features
   - Manages completion requests
   - Handles navigation operations
   - Processes code actions

### Data Flow

1. **Syntax Analysis Flow**
   - Buffer changes trigger incremental parsing
   - Syntax tree is updated
   - Queries extract relevant information
   - Highlighting is applied based on syntax
   - Editor display is updated

2. **LSP Request Flow**
   - User action or edit triggers LSP request
   - Request is formatted and sent to language server
   - Editor waits for response (potentially with timeout)
   - Response is processed and applied to editor state
   - UI is updated to reflect new information

3. **Completion Flow**
   - User typing triggers completion request
   - Local completions (syntax-based) are immediately shown
   - LSP completion request is sent
   - Results are merged and ranked
   - Completion UI is updated with options

4. **Diagnostic Flow**
   - Language server publishes diagnostics
   - Diagnostics are mapped to buffer positions
   - UI indicators show error/warning locations
   - Problem panel is updated
   - Quick fixes are made available

## Key Interfaces

### Language Detection

```
// Conceptual interface, not actual Rust code
LanguageRegistry {
    detect_language(file_path: String, content: Option<String>) -> Language
    register_language(language: Language)
    get_language_by_id(id: String) -> Option<Language>
    get_languages() -> Vec<Language>
}
```

### Tree-sitter Integration

```
// Conceptual interface, not actual Rust code
SyntaxLayer {
    parse(text: String) -> SyntaxTree
    edit(tree: &mut SyntaxTree, edit_info: EditInfo)
    query(tree: &SyntaxTree, query: Query) -> QueryMatches
    node_at_point(tree: &SyntaxTree, point: Point) -> Node
    text_for_node(tree: &SyntaxTree, node: Node) -> String
    node_type(node: &Node) -> String
}
```

### LSP Client

```
// Conceptual interface, not actual Rust code
LspClient {
    initialize(server_path: String, workspace_path: String) -> ServerId
    request<T>(server_id: ServerId, method: String, params: Params) -> Future<T>
    notify(server_id: ServerId, method: String, params: Params)
    get_capabilities(server_id: ServerId) -> ServerCapabilities
    shutdown(server_id: ServerId)
}
```

### Completion Provider

```
// Conceptual interface, not actual Rust code
CompletionProvider {
    get_completions(buffer: &Buffer, position: Point) -> Vec<Completion>
    resolve_completion(completion: &Completion) -> CompletionDetail
    apply_completion(buffer: &mut Buffer, completion: &Completion)
    get_trigger_characters() -> Vec<char>
    should_trigger_automatically(context: &CompletionContext) -> bool
}
```

### Diagnostic Manager

```
// Conceptual interface, not actual Rust code
DiagnosticManager {
    set_diagnostics(source: String, uri: String, diagnostics: Vec<Diagnostic>)
    get_diagnostics(uri: String) -> Vec<Diagnostic>
    get_diagnostics_at(uri: String, position: Point) -> Vec<Diagnostic>
    get_code_actions(uri: String, range: Range) -> Vec<CodeAction>
    apply_code_action(uri: String, action: &CodeAction)
}
```

## State Management

### Language State

1. **Detection State**: Mapping of files to languages
2. **Grammar State**: Loaded Tree-sitter grammars
3. **Configuration State**: Language-specific settings

### Syntax State

1. **Parse Tree**: Current Tree-sitter parse tree per buffer
2. **Query Results**: Cached results of common queries
3. **Highlighting State**: Current syntax highlighting

### LSP State

1. **Server State**: Running language servers and their capabilities
2. **Request State**: In-flight requests and callbacks
3. **Document State**: Server's view of document contents
4. **Capability Map**: What each server can do

### Intelligence State

1. **Completion State**: Current completion session and options
2. **Diagnostic State**: Current errors and warnings
3. **Symbol State**: Cached symbol information
4. **Hover State**: Current hover information

## Swift Considerations

### Tree-sitter Integration

- Tree-sitter has C bindings that can be used from Swift
- Consider wrapping Tree-sitter in a Swift-friendly API
- Use Swift's memory management for Tree-sitter objects

### LSP Client Implementation

- Implement JSON-RPC over stdio/pipes using Swift's Foundation
- Use Swift's concurrency for async request/response
- Consider using Codable for LSP protocol types

### Syntax Highlighting

- Leverage TextKit or custom rendering for highlighting
- Consider using Swift's attributed strings
- Implement theme system using Swift's property wrappers

### Code Intelligence Features

- Use Swift's completion handlers for async operations
- Consider Swift UI for presenting intelligence UI
- Implement LSP types as Swift structs conforming to Codable

### Multi-threading

- Use Swift's concurrency features (async/await)
- Consider actors for thread-safe language services
- Use dispatch queues for background processing

## Key Implementation Patterns

1. **Dynamic Grammar Loading**: Languages load their grammars on demand
2. **Incremental Parsing**: Only reparse changed portions of documents
3. **Request Debouncing**: Limit frequency of LSP requests
4. **Document Synchronization**: Keep LSP server in sync with editor
5. **Capability Detection**: Features adapt based on server capabilities
6. **Mixed Highlighting**: Combine Tree-sitter and semantic highlighting
7. **Language Configuration**: Per-language settings for behavior

## Performance Considerations

1. **Lazy Grammar Loading**: Only load grammars when needed
2. **Incremental Parsing**: Efficient updates to syntax trees
3. **Query Caching**: Cache common Tree-sitter queries
4. **Background Processing**: Run language servers and parsing in background
5. **Request Throttling**: Limit frequency of expensive operations
6. **Partial Updates**: Send minimal document changes to servers
7. **Highlighting Optimization**: Limit highlight calculations to visible text

## Next Steps

After understanding the Language Intelligence system, we'll examine the Project Management system, which handles workspace organization, file navigation, and project-wide operations. This includes file system integration, search capabilities, and project configuration.