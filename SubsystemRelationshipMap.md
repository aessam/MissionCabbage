# Zed Subsystem Relationship Map

This document provides a comprehensive overview of how Zed's major subsystems connect and interact with each other, highlighting the key dependencies and data flows between components.

## Core Dependency Structure

```mermaid
flowchart TD
    GPUI[GPUI\nFoundation Framework] --> Editor[Text Editor Core]
    GPUI --> Project[Project Management]
    GPUI --> UI[UI Systems]
    GPUI --> Lang[Language Intelligence]
    GPUI --> Collab[Collaboration System]
    GPUI --> Ext[Extension System]
    GPUI --> Terminal[Terminal Integration]
    GPUI --> AI[AI Features]
    GPUI --> Settings[Settings System]

    Editor <--> Lang
    Editor <--> Project
    Editor <--> Collab
    Editor <--> Terminal
    Editor <--> AI

    Lang <--> AI
    Lang <--> Ext

    Project <--> Collab
    Project <--> Ext

    Settings --> Editor
    Settings --> Lang
    Settings --> Project
    Settings --> Collab
    Settings --> Terminal
    Settings --> AI
    Settings --> Ext

    Ext <--> Editor
    Ext <--> Terminal
    Ext <--> AI

    classDef core fill:#f96,stroke:#333,stroke-width:2px;
    classDef subsystem fill:#bbf,stroke:#33f,stroke-width:1px;

    class GPUI core;
    class Editor,Lang,Project,Collab,Ext,Terminal,AI,Settings,UI subsystem;
```

## Subsystem Initialization Sequence

```mermaid
sequenceDiagram
    participant App as Application
    participant GPUI
    participant Settings
    participant Project
    participant Editor
    participant Lang as Language Intelligence
    participant Ext as Extension System
    participant Collab as Collaboration
    participant Terminal
    participant AI as AI Features

    App->>GPUI: Initialize framework
    GPUI->>Settings: Initialize settings
    Settings->>Project: Load project configuration
    Project->>Editor: Initialize editor core
    Editor->>Lang: Start language services
    Lang->>Ext: Initialize extension system
    Ext->>Collab: Setup collaboration (if enabled)
    Ext->>Terminal: Initialize terminal
    Ext->>AI: Initialize AI features

    Note over App,AI: Extensions may enhance any subsystem
```

## Key Subsystem Interactions

### GPUI → Text Editor Core
- GPUI provides the UI framework, rendering capabilities, and state management
- Text Editor Core implements editor logic on top of GPUI primitives
- **Key Interface**: Entity-based state system for editor components
- **Data Flow**: GPUI event loop → editor state changes → UI updates

### Text Editor Core → Language Intelligence
- Text Editor Core provides buffer content and cursor information
- Language Intelligence provides syntax highlighting, diagnostics, and code intelligence
- **Key Interface**: Buffer change notifications, language server requests
- **Data Flow**: Edit operations → syntax updates → highlighting and diagnostics

### Text Editor Core → Project Management
- Text Editor Core operates on buffers provided by Project Management
- Project Management keeps track of files, directories, and project structures
- **Key Interface**: File access and modification APIs
- **Data Flow**: Project events → buffer loading/saving → editor operations

### Project Management → Collaboration
- Project Management provides project structure for collaboration
- Collaboration system synchronizes changes across client instances
- **Key Interface**: Project update events, operational transform
- **Data Flow**: Local changes → collaboration sync → remote updates

### Text Editor Core → Terminal Integration
- Text Editor Core provides workspace for terminal views
- Terminal Integration extends editor with shell access
- **Key Interface**: Terminal view APIs, process execution
- **Data Flow**: Terminal input → shell process → terminal output → view update

### Language Intelligence → AI Features
- Language Intelligence provides code context and structure
- AI Features use code understanding for intelligent assistance
- **Key Interface**: Code context extraction, semantic analysis
- **Data Flow**: Code selection → context building → AI query → suggested edits

### Extension System → Multiple Subsystems
- Extension System connects to almost all other subsystems
- Provides extension points for customizing editor functionality
- **Key Interface**: Extension API surface, capability system
- **Data Flow**: Extension activation → feature registration → extended functionality

### Settings → All Subsystems
- Settings system affects behavior of all other subsystems
- Provides user configuration for system-wide and per-project settings
- **Key Interface**: Configuration access APIs, change notifications
- **Data Flow**: Setting changes → configuration events → behavior adaptation

## Subsystem Dependencies Table

| Subsystem           | Directly Depends On                           | Provides Services To                       |
|---------------------|-----------------------------------------------|-------------------------------------------|
| GPUI                | None (foundational)                           | All other subsystems                       |
| Text Editor Core    | GPUI                                         | Language Intelligence, Terminal, AI Features |
| Language Intelligence | GPUI, Text Editor Core                       | Text Editor Core, AI Features              |
| Project Management  | GPUI, Text Editor Core                       | Text Editor Core, Collaboration, Extensions |
| Collaboration       | GPUI, Project Management, Text Editor Core   | Text Editor Core, Project Management        |
| Extension System    | GPUI, Text Editor Core                       | All other subsystems                       |
| Terminal Integration | GPUI, Text Editor Core                       | Text Editor Core, Extensions               |
| AI Features         | GPUI, Text Editor Core, Language Intelligence | Text Editor Core, Extensions               |
| Settings            | GPUI                                         | All other subsystems                       |

## Communication Patterns

```mermaid
classDiagram
    class EventBasedCommunication {
        +Publisher
        +Subscriber
        +EventBus
        +publish(event)
        +subscribe(eventType, callback)
        +unsubscribe(subscription)
    }

    class DirectAPICalls {
        +SynchronousCall
        +MethodInvocation
        +ReturnValue
        +callFunction(arguments)
        +getResult()
    }

    class EntityBasedStateSharing {
        +Entity~T~
        +WeakEntity~T~
        +Observer
        +notify()
        +read(cx)
        +update(cx, callback)
        +observe(entity, callback)
    }

    class TaskBasedAsyncOperations {
        +Task~T~
        +Future
        +Promise
        +spawn(async_fn)
        +background_spawn(async_fn)
        +await()
        +detach()
    }

    note for EventBasedCommunication "Used for loosely coupled subsystems\nExample: Buffer changes → Language updates"
    note for DirectAPICalls "Used for tightly integrated subsystems\nExample: Terminal process execution"
    note for EntityBasedStateSharing "Used for UI and state management\nExample: Editor state → UI rendering"
    note for TaskBasedAsyncOperations "Used for long-running operations\nExample: AI model requests, file operations"
```

### Data Flow Example: Code Editing to Syntax Highlighting

```mermaid
sequenceDiagram
    participant User
    participant Editor as Text Editor
    participant Buffer
    participant Lang as Language Intelligence
    participant Tree as Tree-sitter
    participant UI as Editor UI

    User->>Editor: Type character
    Editor->>Buffer: Insert character
    Buffer-->>Editor: Buffer changed event
    Editor->>Lang: Notify buffer changed
    Lang->>Tree: Update syntax tree
    Tree-->>Lang: Updated tree
    Lang->>UI: Update syntax highlighting
    UI-->>User: Display highlighted text

    Note over Buffer,Lang: Entity-based communication
    Note over Lang,Tree: Direct API calls
    Note over Lang,UI: Event-based communication
```

## State Ownership and Sharing

```mermaid
graph TD
    subgraph BufferContent["Buffer Content"]
        BC[Text Editor Core]
        BC -->|shares with| BC_LI[Language Intelligence]
        BC -->|shares with| BC_CS[Collaboration]
    end

    subgraph ProjectStructure["Project Structure"]
        PS[Project Management]
        PS -->|shares with| PS_TEC[Text Editor Core]
        PS -->|shares with| PS_ES[Extension System]
    end

    subgraph LanguageInfo["Language Information"]
        LI[Language Intelligence]
        LI -->|shares with| LI_TEC[Text Editor Core]
        LI -->|shares with| LI_AI[AI Features]
    end

    subgraph UIState["UI State"]
        UI[GPUI]
        UI -->|shares with| UI_ALL[All UI Components]
    end

    subgraph ExtensionReg["Extension Registry"]
        ER[Extension System]
        ER -->|shares with| ER_ALL[All Extensible Subsystems]
    end

    subgraph SettingsConfig["Settings/Configuration"]
        SC[Settings System]
        SC -->|shares with| SC_ALL[All Configurable Subsystems]
    end

    subgraph CollabState["Collaboration State"]
        CS[Collaboration System]
        CS -->|shares with| CS_TEC[Text Editor Core]
        CS -->|shares with| CS_PM[Project Management]
    end

    subgraph TerminalState["Terminal State"]
        TS[Terminal Integration]
        TS -->|shares with| TS_TEC[Text Editor Core]
    end

    subgraph AIContext["AI Context/History"]
        AI[AI Features]
        AI -->|shares with| AI_LI[Language Intelligence]
    end

    classDef owner fill:#f96,stroke:#333,stroke-width:2px;
    classDef shared fill:#bbf,stroke:#33f,stroke-width:1px;

    class BC,PS,LI,UI,ER,SC,CS,TS,AI owner;
    class BC_LI,BC_CS,PS_TEC,PS_ES,LI_TEC,LI_AI,UI_ALL,ER_ALL,SC_ALL,CS_TEC,CS_PM,TS_TEC,AI_LI shared;
```

### State Access Patterns

| State Category        | Primary Owner            | Shared With                           | Access Pattern |
|-----------------------|--------------------------|---------------------------------------|--------------------|
| Buffer Content        | Text Editor Core         | Language Intelligence, Collaboration  | Entity observation with change notification |
| Project Structure     | Project Management       | Text Editor Core, Extension System    | Entity reading with query APIs |
| Language Information  | Language Intelligence    | Text Editor Core, AI Features         | Event-based updates |
| UI State              | GPUI                     | All subsystems with UI components     | Entity system with rendering callbacks |
| Extension Registry    | Extension System         | All extensible subsystems             | Registry pattern with capability queries |
| Settings/Configuration| Settings System          | All configurable subsystems           | Observable settings with change events |
| Collaboration State   | Collaboration System     | Text Editor Core, Project Management  | Synchronized state with OT algorithm |
| Terminal State        | Terminal Integration     | Text Editor Core (for display)        | Buffer-like API with display hooks |
| AI Context/History    | AI Features              | Language Intelligence (for context)   | Context provider pattern |

## Initialization Sequence

1. GPUI framework initialization
2. Settings system initialization
3. Project Management startup
4. Text Editor Core initialization
5. Language Intelligence services startup
6. Extension system initialization and extension loading
7. Collaboration system connection (if enabled)
8. Terminal Integration initialization
9. AI Features initialization

## Cross-Document References

For detailed information on each subsystem, refer to:

- [01_OrbitalView_ProjectArchitecture.md](./01_OrbitalView_ProjectArchitecture.md) - Overall system architecture
- [02_StratosphericView_GPUI.md](./02_StratosphericView_GPUI.md) - UI framework and state management
- [03_StratosphericView_TextEditorCore.md](./03_StratosphericView_TextEditorCore.md) - Core editing functionality
- [04_StratosphericView_LanguageIntelligence.md](./04_StratosphericView_LanguageIntelligence.md) - Language support
- [05_StratosphericView_ProjectManagement.md](./05_StratosphericView_ProjectManagement.md) - Project handling
- [06_StratosphericView_CollaborationSystem.md](./06_StratosphericView_CollaborationSystem.md) - Real-time collaboration
- [07_StratosphericView_ExtensionSystem.md](./07_StratosphericView_ExtensionSystem.md) - Extension system
- [08_StratosphericView_TerminalIntegration.md](./08_StratosphericView_TerminalIntegration.md) - Terminal features
- [09_StratosphericView_AIFeatures.md](./09_StratosphericView_AIFeatures.md) - AI capabilities