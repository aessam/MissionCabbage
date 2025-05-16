# Orbital View: Zed Project Architecture

## Introduction

Zed is a high-performance, collaborative code editor designed with modern workflows in mind. This document provides a high-altitude overview of Zed's architecture, core concepts, and how the major subsystems interact.

## Core Philosophy

Zed's architecture appears guided by several key principles:

1. **Performance-first design**: Optimized for responsiveness and low latency
2. **Collaboration as a first-class feature**: Built for real-time multi-user editing
3. **Extensibility**: Plugin architecture for customization
4. **Modern language intelligence**: Deep integration with language servers
5. **GPU-accelerated UI**: Custom rendering for smooth visuals

## Major Subsystems

Zed's architecture consists of several major subsystems, each handling distinct responsibilities:

### 1. GPUI Framework

At the core of Zed is GPUI, a custom UI framework that provides:
- Entity component system for state management
- Reactive rendering system
- Flexbox-based layout engine
- Concurrency primitives for UI programming
- Event dispatching system

### 2. Text Editor Core

The text editing engine includes:
- Buffer management for file contents
- Cursor and selection models
- Text manipulation operations
- Document history and undo/redo
- Multi-buffer editing

### 3. Language Intelligence

Advanced code understanding features:
- Language Server Protocol (LSP) integration
- Tree-sitter parsing for syntax highlighting and structure
- Code completion mechanisms
- Diagnostics display
- Symbol navigation
- Semantic index for codebase-wide understanding

### 4. Terminal Integration

Integrated terminal functionality:
- Terminal emulation
- Command execution
- Process management
- Terminal session persistence

### 5. Project Management

Workspace and project organization:
- File system integration
- Project structure representation
- Search capabilities
- Git integration
- Multiple workspace support

### 6. Collaboration System

Real-time collaboration features:
- Multiplayer editing with Operational Transformation
- Presence awareness
- Chat functionality
- Permissions and access control
- Call/voice functionality

### 7. Extension System

Extensibility architecture:
- Plugin API and capabilities
- Sandboxed execution environment
- Settings and configuration
- Theme and language extensions

### 8. AI Integration

AI assistance features:
- Multiple model integration (Anthropic, OpenAI, etc.)
- Context handling and prompting
- Code generation
- Inline completions
- Chat interface

## Data Flow Architecture

Zed's architecture follows several key patterns for data flow:

1. **Entity-based State Management**
   - State is organized into entities that can be observed and updated
   - Changes propagate through the system via notifications

2. **Event-driven Communication**
   - Components communicate via typed events
   - Action dispatch system for user interactions

3. **Concurrent Task Management**
   - Background tasks for non-UI work
   - Foreground tasks for UI updates
   - Task cancellation and lifecycle management

4. **Reactive UI Updates**
   - UI automatically reacts to state changes
   - Render tree diffing for efficient updates

## Core Abstractions

### Entity System

The Entity system provides the foundation for Zed's state management:
- Entities encapsulate state and behavior
- Context objects provide access to the entity system
- Weak references prevent memory leaks

### Window and Rendering

The rendering system manages:
- Window creation and management
- Input event handling
- Drawing and layout
- Focus management

### Project and Workspace

Organization of user's code:
- WorkTree represents a directory structure
- Project represents a collection of worktrees
- Workspace represents the editor's view of projects

### Buffer System

Text content management:
- Buffer represents file contents
- Rope data structure for efficient text operations
- Anchor system for tracking positions through edits

### Language Services

Code intelligence system:
- Language Server clients
- Tree-sitter grammar integration
- Syntax highlighting
- Code navigation

### Collaboration

Multi-user editing:
- Operational transform for conflict resolution
- Room management for collaborative sessions
- Presence indicators for user awareness

## Cross-cutting Concerns

Several aspects cut across multiple subsystems:

1. **Settings System**
   - User preferences
   - Language-specific settings
   - Theme configuration

2. **Command System**
   - Key binding management
   - Command registration and dispatch
   - Menu integration

3. **Telemetry**
   - Performance monitoring
   - Usage analytics
   - Error reporting

4. **Diagnostics and Logging**
   - Error handling
   - Logging framework
   - Debug information

## Platform Integration

Zed interfaces with the underlying platform through:
- File system access
- Process creation
- Window management
- Font rendering
- GPU access
- Network communication

## Data Persistence

Zed persists various types of data:
- Editor state and session
- User preferences
- Project history
- Buffer content (for unsaved files)
- Extension data

## Next Steps

This orbital view has outlined the major components and interactions in Zed. In subsequent documents, we'll zoom in to each major subsystem for more detailed analysis, beginning with the GPUI framework that underpins the entire application.