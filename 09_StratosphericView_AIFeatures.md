# Stratospheric View: AI Features

## Purpose

The AI Features system integrates various AI capabilities into the Zed editor, providing intelligent assistance for coding tasks. It offers code generation, completions, contextual explanations, and conversational interfaces that enhance developer productivity by leveraging large language models and other AI technologies.

## Implementation Note

In the Zed codebase, AI features are implemented across multiple specialized crates rather than as a single monolithic system:

- **Provider Crates**: Separate crates for different AI models (`anthropic`, `open_ai`, `bedrock`, `google_ai`, `mistral`, `ollama`, `lmstudio`, `deepseek`, `copilot`, `supermaven`)
- **Core Agent System**: The primary `agent` crate contains the assistant UI, threading model, and chat functionality
- **Assistant Components**: Multiple assistant-related crates (`assistant_context_editor`, `assistant_slash_command`, `assistant_slash_commands`, `assistant_tool`, `assistant_tools`) for various aspects of the assistant functionality
- **Completion System**: The `inline_completion` crate provides the core functionality for AI code suggestions
- **Model Management**: The `language_model` and `language_models` crates provide unified interfaces across multiple providers

This modular approach allows for flexible integration of various AI models and clean separation of concerns between different AI features.

## Core Concepts

### AI Integration Architecture

- **Model Providers**: Connections to AI services (OpenAI, Anthropic, Mistral, Ollama, LMStudio, DeepSeek, Bedrock, Google AI, Copilot, Supermaven)
- **Context Generation**: Building relevant context for AI requests (`agent/src/context.rs`, `assistant_context_editor/src/context.rs`)
- **Query Processing**: Handling user queries to AI systems (`agent/src/thread.rs`)
- **Response Processing**: Integrating AI outputs into the editor (`agent/src/agent.rs`)
- **Tool Execution**: Allowing AI to use editor tools and features (`assistant_tool/src/tool_registry.rs`, `assistant_tools/*`)

### Inline Completions

- **Completion Providers**: Sources of code suggestions
- **Trigger Mechanism**: When completions are activated
- **Completion Display**: How suggestions appear in editor
- **Acceptance Flow**: Process of accepting completions
- **Filtering**: Relevance ranking of suggestions

### AI Assistant

- **Assistant Interface**: Chat-based interaction with AI
- **Conversation Context**: Maintaining chat history and state
- **Slash Commands**: Special directives for assistant
- **Code Actions**: Assistant-driven coding operations
- **Contextual Awareness**: Understanding of workspace

### Language Model Integration

- **Model Configuration**: Settings for different AI models
- **API Communication**: Protocols for model interaction
- **Prompt Engineering**: Crafting effective AI instructions
- **Response Streaming**: Progressive display of AI responses
- **Token Management**: Handling model context limitations

### AI Tools

- **Code Generation**: Creating new code from descriptions
- **Code Explanation**: Understanding existing code
- **Refactoring Assistance**: Improving code structure
- **Documentation**: Generating code documentation
- **Bug Finding**: Identifying potential issues

## Architecture

### Core Components

1. **Language Model Manager**
   - Manages connections to different AI providers
   - Handles authentication and API keys
   - Tracks usage and quotas
   - Provides unified interface across models
   - Manages model-specific configuration

2. **Context Provider**
   - Gathers relevant code context
   - Builds prompts with appropriate information
   - Handles context windowing for token limits
   - Includes file, project, and selection context
   - Maintains conversation history

3. **Assistant Panel**
   - Provides conversational UI for AI interaction
   - Displays conversation history
   - Handles user input and AI responses
   - Manages threading and conversation flow
   - Integrates with slash commands

4. **Inline Completion System**
   - Intercepts typing events
   - Requests and displays suggestions
   - Handles completion acceptance
   - Manages visibility and relevance
   - Coordinates with language servers

5. **Tool Integration**
   - Allows AI to perform editor actions
   - Manages permissions for AI operations
   - Handles tool execution requests
   - Provides feedback on tool operations
   - Ensures safety of AI-performed actions

### Data Flow

1. **Inline Completion Flow**
   - User types in editor
   - Completion system detects trigger condition
   - Context is gathered from current file and cursor position
   - Context and partial input are sent to model
   - Model returns completion suggestions
   - Suggestions are displayed inline
   - User accepts, ignores, or modifies suggestion

2. **Assistant Conversation Flow**
   - User opens assistant panel
   - User enters query or selects code
   - Context provider gathers relevant information
   - Context and query are sent to language model
   - Model processes and streams response
   - Response is displayed in conversation UI
   - User can follow up or request actions

3. **Tool Execution Flow**
   - AI determines need for tool use
   - AI requests specific tool with parameters
   - Tool use permission is checked
   - Tool executes requested operation
   - Operation results are returned to AI
   - AI incorporates results into response
   - User sees outcome of operation

4. **Model Selection Flow**
   - User configures preferred models
   - System selects appropriate model based on task
   - Authentication is performed with provider
   - Model capabilities are determined
   - Features are enabled based on capabilities
   - Best model is used for each interaction

## Key Interfaces

### Language Model Manager

```
// Conceptual interface, not actual Rust code
LanguageModelManager {
    get_available_models() -> Vec<ModelInfo>
    set_active_model(model_id: String)
    get_active_model() -> ModelInfo
    send_completion_request(prompt: String, options: CompletionOptions) -> CompletionResponse
    send_chat_request(messages: Vec<Message>, options: ChatOptions) -> ChatResponse
    stream_chat_response(messages: Vec<Message>, options: ChatOptions) -> Stream<ChatChunk>
    get_model_capabilities(model_id: String) -> ModelCapabilities
    get_remaining_quota() -> QuotaInfo
    configure_model(model_id: String, config: ModelConfig)
}
```

### Context Provider

```
// Conceptual interface, not actual Rust code
ContextProvider {
    get_selection_context() -> Context
    get_file_context(file_id: FileId) -> Context
    get_project_context() -> Context
    get_conversation_context(conversation_id: ConversationId) -> Context
    build_prompt(query: String, context_type: ContextType) -> Prompt
    get_relevant_files(query: String) -> Vec<FileInfo>
    truncate_context(context: Context, max_tokens: usize) -> Context
    enhance_context(context: Context, enhancement_type: EnhancementType) -> Context
}
```

### Assistant Panel

```
// Conceptual interface, not actual Rust code
AssistantPanel {
    show_panel()
    hide_panel()
    send_message(message: String) -> MessageId
    get_conversation_history() -> Vec<Message>
    clear_conversation()
    start_new_conversation() -> ConversationId
    save_conversation(conversation_id: ConversationId)
    load_conversation(conversation_id: ConversationId)
    register_slash_command(command: SlashCommand)
    execute_slash_command(command: String, args: Vec<String>)
}
```

### Inline Completion System

```
// Conceptual interface, not actual Rust code
InlineCompletionSystem {
    request_completion(buffer_id: BufferId, position: Position) -> CompletionResult
    accept_completion(buffer_id: BufferId, completion_id: CompletionId)
    reject_completion(buffer_id: BufferId, completion_id: CompletionId)
    get_visible_completions(buffer_id: BufferId) -> Vec<Completion>
    set_completion_trigger(trigger_type: TriggerType)
    enable_completions(enabled: bool)
    configure_completions(config: CompletionConfig)
    get_completion_statistics() -> CompletionStats
}
```

### Tool Integration

```
// Conceptual interface, not actual Rust code
ToolIntegration {
    register_tool(tool: Tool) -> ToolId
    execute_tool(tool_id: ToolId, args: ToolArgs) -> ToolResult
    get_available_tools() -> Vec<ToolInfo>
    set_tool_permissions(tool_id: ToolId, permissions: ToolPermissions)
    get_tool_execution_history() -> Vec<ToolExecution>
    add_tool_execution_observer(observer: Observer)
    validate_tool_args(tool_id: ToolId, args: ToolArgs) -> ValidationResult
}
```

## State Management

### Model State

1. **Provider Configuration**: API keys and endpoints
2. **Active Models**: Currently configured models
3. **Model Preferences**: User preferred models
4. **Usage Metrics**: Token usage and quotas
5. **Request History**: Record of model interactions
6. **Capability Registry**: What each model can do
7. **Performance Metrics**: Response times and quality

### Assistant State

1. **Conversation History**: Messages in current conversation
2. **Threading State**: Conversation organization
3. **Input State**: Current user input
4. **Response State**: Current AI response
5. **UI State**: Panel visibility and configuration
6. **Command Registry**: Available slash commands
7. **Action State**: Pending and completed actions

### Completion State

1. **Suggestion Cache**: Recently generated completions
2. **Display State**: Currently shown suggestions
3. **Trigger State**: Activation conditions
4. **Acceptance Rate**: User interaction metrics
5. **Provider Priority**: Ordering of suggestion sources
6. **Mode Configuration**: How completions behave
7. **Performance Metrics**: Latency and quality stats

### Context State

1. **Selection Context**: Current text selection
2. **File Context**: Relevant parts of current file
3. **Project Context**: Related code from workspace
4. **Language Context**: Language-specific information
5. **Conversation Context**: Previous exchanges
6. **Token Counts**: Context size metrics
7. **Relevance Metrics**: Context quality indicators

### Tool State

1. **Available Tools**: Registered tool catalog
2. **Permission State**: What tools AI can use
3. **Execution History**: Record of tool operations
4. **Result Cache**: Outcomes of recent tool uses
5. **Error State**: Failed tool executions
6. **Safety Locks**: Protections against risky operations
7. **Performance Metrics**: Tool execution statistics

## Swift Considerations

### Language Model Integration

- Use Swift's concurrency for async model communication
- Consider implementing client libraries for major providers
- Design flexible abstractions for different models
- Use Swift's strong typing for model responses

### Assistant Implementation

- Consider SwiftUI for assistant interface
- Design clean Swift protocols for assistant functionality
- Use Swift's Combine for reactive response handling
- Implement proper thread safety for assistant state

### Inline Completions

- Design Swift-native inline completion display
- Consider CoreText integration for precise rendering
- Use Swift's pattern matching for completion filtering
- Implement efficient completion caching

### Context Handling

- Use Swift's value types for immutable context
- Consider custom Swift types for context modeling
- Implement efficient context windowing algorithms
- Use Swift's strong type system for context safety

### Tool Integration

- Design a Swift protocol-based tool system
- Use Swift's result builders for tool definitions
- Consider capability-based permission model
- Implement proper error handling for tool execution

## Key Implementation Patterns

1. **Streaming Responses**: Progressive display of AI responses
2. **Context Windowing**: Managing context size limits
3. **Completion Debouncing**: Limiting completion requests
4. **Prompt Templates**: Standardized AI instruction formats
5. **Fallback Chain**: Cascading through models when needed
6. **Tool Permission Model**: Safety-focused tool authorization
7. **Conversation Threading**: Organizing assistant interactions

## Performance Considerations

1. **Request Optimization**: Minimizing API calls
2. **Context Pruning**: Smart selection of relevant context
3. **Completion Caching**: Reusing recent suggestions
4. **Response Streaming**: Progressive UI updates
5. **Background Processing**: Non-blocking AI operations
6. **Local Models**: Support for on-device models
7. **Adaptive Quality**: Adjusting to network conditions

## Next Steps

After understanding the AI Features system, we'll examine the Settings System, which handles user preferences, configuration storage, and customization of the editor behavior. This includes both application-wide and project-specific settings, as well as the UI for settings management.