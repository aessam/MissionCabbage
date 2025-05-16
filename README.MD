# Mission Cabbage Connectivity Map

This document serves as the central navigation hub for the Mission Cabbage documentation. It visualizes how concepts flow across different abstraction layers, from high-level architecture to concrete implementation details.

## Documentation Layer Structure

The Mission Cabbage documentation is organized into five distinct layers, each representing a deeper level of technical detail:

```
┌──────────────────────────────┐
│ ORBITAL VIEW                 │
│ High-level system overview   │
├──────────────────────────────┤
│ STRATOSPHERIC VIEW           │
│ Major subsystem architecture │
├──────────────────────────────┤
│ ATMOSPHERIC VIEW             │
│ Component-level design       │
├──────────────────────────────┤
│ CLOUD LEVEL                  │
│ Algorithm & implementation   │
├──────────────────────────────┤
│ GROUND LEVEL                 │
│ Concrete implementations     │
└──────────────────────────────┘
```

Additionally, there are two special document types:

```
┌──────────────────────────────┐
│ BRIDGE DOCUMENTS             │
│ Connect abstract concepts    │
│ with implementation details  │
├──────────────────────────────┤
│ CROSS-CUTTING CONCERNS       │
│ Topics that span multiple    │
│ subsystems and layers        │
└──────────────────────────────┘
```

## Complete Document Catalog

### Orbital View
- [01_OrbitalView_ProjectArchitecture.md](01_OrbitalView_ProjectArchitecture.md) - High-level architecture overview

### Stratospheric View
- [02_StratosphericView_GPUI.md](02_StratosphericView_GPUI.md) - UI framework & state management
- [03_StratosphericView_TextEditorCore.md](03_StratosphericView_TextEditorCore.md) - Core text editing functionality
- [04_StratosphericView_LanguageIntelligence.md](04_StratosphericView_LanguageIntelligence.md) - Language features
- [05_StratosphericView_ProjectManagement.md](05_StratosphericView_ProjectManagement.md) - Project & workspace
- [06_StratosphericView_CollaborationSystem.md](06_StratosphericView_CollaborationSystem.md) - Collaboration features
- [07_StratosphericView_ExtensionSystem.md](07_StratosphericView_ExtensionSystem.md) - Extension architecture
- [08_StratosphericView_TerminalIntegration.md](08_StratosphericView_TerminalIntegration.md) - Terminal features
- [09_StratosphericView_AIFeatures.md](09_StratosphericView_AIFeatures.md) - AI capabilities
- [10_StratosphericView_Settings.md](10_StratosphericView_Settings.md) - Settings system
- [11_StratosphericView_CommandSystem.md](11_StratosphericView_CommandSystem.md) - Command dispatch
- [12_StratosphericView_ThemeSystem.md](12_StratosphericView_ThemeSystem.md) - Theme & styling

### Atmospheric View
- [13_AtmosphericView_BufferAndRope.md](13_AtmosphericView_BufferAndRope.md) - Text buffer implementation
- [14_AtmosphericView_CursorAndSelection.md](14_AtmosphericView_CursorAndSelection.md) - Cursor management
- [15_AtmosphericView_SyntaxHighlighting.md](15_AtmosphericView_SyntaxHighlighting.md) - Syntax highlighting
- [16_AtmosphericView_UIComponents.md](16_AtmosphericView_UIComponents.md) - UI component architecture
- [17_AtmosphericView_SearchSystem.md](17_AtmosphericView_SearchSystem.md) - Search functionality
- [18_AtmosphericView_GitIntegration.md](18_AtmosphericView_GitIntegration.md) - Git integration
- [19_AtmosphericView_TaskSystem.md](19_AtmosphericView_TaskSystem.md) - Task execution
- [20_AtmosphericView_DebugSupport.md](20_AtmosphericView_DebugSupport.md) - Debugging facilities
- [21_AtmosphericView_MultiBuffer.md](21_AtmosphericView_MultiBuffer.md) - Multi-buffer editing
- [22_AtmosphericView_FileWatcher.md](22_AtmosphericView_FileWatcher.md) - File system monitoring

### Cloud Level
- [23_CloudLevel_RopeAlgorithms.md](23_CloudLevel_RopeAlgorithms.md) - Rope data structure details
- [24_CloudLevel_TreeSitterIntegration.md](24_CloudLevel_TreeSitterIntegration.md) - Tree-sitter parsing
- [25_CloudLevel_EntitySystem.md](25_CloudLevel_EntitySystem.md) - Entity system design
- [26_CloudLevel_OperationalTransform.md](26_CloudLevel_OperationalTransform.md) - Collaborative editing
- [27_CloudLevel_TaskScheduling.md](27_CloudLevel_TaskScheduling.md) - Task execution implementation
- [28_CloudLevel_UILayout.md](28_CloudLevel_UILayout.md) - UI layout algorithms
- [29_CloudLevel_ExtensionSandbox.md](29_CloudLevel_ExtensionSandbox.md) - Extension sandboxing
- [30_CloudLevel_LanguageServerProtocol.md](30_CloudLevel_LanguageServerProtocol.md) - LSP implementation
- [31_CloudLevel_SearchAlgorithms.md](31_CloudLevel_SearchAlgorithms.md) - Search algorithms
- [32_CloudLevel_AIPrompting.md](32_CloudLevel_AIPrompting.md) - AI prompt engineering

### Ground Level
- [33_GroundLevel_PerformanceOptimizations.md](33_GroundLevel_PerformanceOptimizations.md) - Performance tuning
- [34_GroundLevel_MemoryManagement.md](34_GroundLevel_MemoryManagement.md) - Memory management
- [35_GroundLevel_TextProcessing.md](35_GroundLevel_TextProcessing.md) - Text manipulation
- [36_GroundLevel_UIRendering.md](36_GroundLevel_UIRendering.md) - UI rendering
- [37_GroundLevel_EventHandling.md](37_GroundLevel_EventHandling.md) - Event system
- [38_GroundLevel_FileIO.md](38_GroundLevel_FileIO.md) - File operations
- [39_GroundLevel_NetworkProtocols.md](39_GroundLevel_NetworkProtocols.md) - Network communications
- [40_GroundLevel_StatePersistence.md](40_GroundLevel_StatePersistence.md) - State persistence
- [46_GroundLevel_NextActionPrediction.md](46_GroundLevel_NextActionPrediction.md) - Predictive features
- [47_GroundLevel_TreeSitterImplementation.md](47_GroundLevel_TreeSitterImplementation.md) - Tree-sitter details
- [48_GroundLevel_LanguageServerImplementation.md](48_GroundLevel_LanguageServerImplementation.md) - LSP details
- [49_GroundLevel_ReactiveUIImplementation.md](49_GroundLevel_ReactiveUIImplementation.md) - Reactive UI patterns
- [51_GroundLevel_InputHandlingSystem.md](51_GroundLevel_InputHandlingSystem.md) - Input event handling
- [52_GroundLevel_RopeImplementation.md](52_GroundLevel_RopeImplementation.md) - Rope implementation details
- [53_GroundLevel_CommandDispatchSystem.md](53_GroundLevel_CommandDispatchSystem.md) - Command system implementation

### Swift Implementation
- [41_Swift_EntitySystem.md](41_Swift_EntitySystem.md) - Swift entity system
- [42_Swift_ReactiveUI.md](42_Swift_ReactiveUI.md) - Swift reactive UI
- [43_Swift_TextBuffer.md](43_Swift_TextBuffer.md) - Swift text buffer
- [44_Swift_ConcurrencyModel.md](44_Swift_ConcurrencyModel.md) - Swift concurrency
- [45_Swift_ExtensionSystem.md](45_Swift_ExtensionSystem.md) - Swift extensions

### Bridge Documents
- [B01_EntitySystem_TechnicalBridge.md](B01_EntitySystem_TechnicalBridge.md) - Entity system implementation bridge
- [B02_TextBuffer_TechnicalBridge.md](B02_TextBuffer_TechnicalBridge.md) - Text buffer implementation bridge
- [B03_UI_TechnicalBridge.md](B03_UI_TechnicalBridge.md) - UI implementation bridge
- [B04_LanguageIntelligence_TechnicalBridge.md](B04_LanguageIntelligence_TechnicalBridge.md) - Language features bridge

### Cross-Cutting Concerns
- [50_CrossCutting_PerformanceOptimizationPatterns.md](50_CrossCutting_PerformanceOptimizationPatterns.md) - Performance patterns
- [54_CrossCutting_ErrorHandlingRecoveryPatterns.md](54_CrossCutting_ErrorHandlingRecoveryPatterns.md) - Error handling
- [55_CrossCutting_MemoryManagementPatterns.md](55_CrossCutting_MemoryManagementPatterns.md) - Memory patterns

### Supporting Documents
- [AcquisitionProposal.md](AcquisitionProposal.md) - Original acquisition proposal
- [MinimumViableSwiftPrototype.md](MinimumViableSwiftPrototype.md) - Swift prototype plan
- [SubsystemRelationshipMap.md](SubsystemRelationshipMap.md) - Subsystem relationships

## Core Subsystem Map

Below is a visualization of how the core subsystems of Zed are documented across different layers:

```
                            ┌───────────────────────────┐
                            │ Project Architecture [01] │
                            └───────────────┬───────────┘
                                            │
                 ┌────────────┬─────────────┼────────────┬─────────────┐
                 │            │             │            │             │
    ┌────────────▼─────┐ ┌────▼────┐ ┌──────▼─────┐ ┌───▼────┐ ┌──────▼───────┐
    │ GPUI Framework   │ │ Editor  │ │ Language   │ │Project │ │Collaboration │
    │      [02]        │ │  [03]   │ │Intelligence│ │  [05]  │ │    [06]      │
    └────────────┬─────┘ └────┬────┘ │    [04]    │ └───┬────┘ └──────┬───────┘
                 │            │      └──────┬─────┘     │             │
                 │            │             │           │             │
    ┌────────────▼─────┐ ┌────▼────┐ ┌──────▼─────┐ ┌───▼────┐ ┌──────▼───────┐
    │Entity System [25]│ │Buffer & │ │Tree-Sitter │ │Search  │ │Operational   │
    │UI Layout    [28] │ │Rope [13]│ │   [24]     │ │  [17]  │ │Transform [26]│
    └────────────┬─────┘ └────┬────┘ └──────┬─────┘ └───┬────┘ └──────────────┘
                 │            │             │           │
                 ▼            ▼             ▼           ▼
    ┌────────────┴─────┐ ┌────┴────┐ ┌──────┴─────┐ ┌───┴────┐
    │ ReactiveUI [49]  │ │Rope     │ │LSP         │ │File I/O│
    │ UI Rendering [36]│ │Algo [23]│ │Implementation│ │  [38] │
    │ EventHandling[37]│ │Impl [52]│ │    [48]    │ └────────┘
    └────────────┬─────┘ └────┬────┘ └──────┬─────┘
                 │            │             │
                 ▼            ▼             ▼
    ┌────────────┴─────┐ ┌────┴────┐ ┌──────┴─────┐
    │Memory Management │ │Text     │ │Tree-Sitter │
    │      [34,55]     │ │Processing│ │Implementation│
    │Input Handling[51]│ │  [35]   │ │    [47]    │
    └──────────────────┘ └─────────┘ └────────────┘
```

## Bridge Document Map

The bridge documents connect abstract concepts with concrete implementations:

```
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│   Stratospheric     │     │     Bridge          │     │   Implementation    │
│       View          │     │    Document         │     │      Details        │
├─────────────────────┤     ├─────────────────────┤     ├─────────────────────┤
│ GPUI Framework [02] │────▶│ Entity System [B01] │────▶│ Swift Entity [41]   │
│                     │     │                     │     │ ReactiveUI [49]     │
├─────────────────────┤     ├─────────────────────┤     ├─────────────────────┤
│ Text Editor   [03]  │────▶│ Text Buffer  [B02]  │────▶│ Swift Buffer [43]   │
│                     │     │                     │     │ Rope Impl [52]      │
├─────────────────────┤     ├─────────────────────┤     ├─────────────────────┤
│ GPUI/UI Comp. [02/16]────▶│ UI          [B03]   │────▶│ Swift ReactiveUI[42]│
│                     │     │                     │     │ UI Rendering [36]   │
├─────────────────────┤     ├─────────────────────┤     ├─────────────────────┤
│ Language Intel [04] │────▶│ Lang Intel   [B04]  │────▶│ TreeSitter Impl [47]│
│                     │     │                     │     │ LSP Impl [48]       │
└─────────────────────┘     └─────────────────────┘     └─────────────────────┘
```

## Cross-Cutting Concerns Map

Cross-cutting documents address concerns that span multiple subsystems:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CROSS-CUTTING CONCERNS                                │
├─────────────────┬─────────────────────────────┬───────────────────────────┐
│                 │                             │                           │
│ Performance     │ Error Handling              │ Memory Management         │
│ Optimization    │ Recovery Patterns           │ Patterns                  │
│ Patterns [50]   │ [54]                        │ [55]                      │
│                 │                             │                           │
└─────────────────┴─────────────────────────────┴───────────────────────────┘
                                    │
                                    ▼
┌─────────────────┬─────────────────────────────┬───────────────────────────┐
│ Entity System   │ Text Editing                │ UI System                 │
│ - Swift Entity  │ - Rope Implementation       │ - Reactive UI             │
│ - Event Handling│ - Buffer Management         │ - Rendering               │
└─────────────────┴─────────────────────────────┴───────────────────────────┘
```

## Swift Implementation Path

The Swift implementation path shows how Zed's core concepts translate to Swift:

```
┌──────────────────────────────┐
│ Project Architecture [01]    │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│ Swift Entity System [41]     │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│ Swift Reactive UI [42]       │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│ Swift Text Buffer [43]       │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│ Swift Concurrency Model [44] │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│ Swift Extension System [45]  │
└──────────────────────────────┘
```

## AI Features Path

The path to implementing AI and predictive features:

```
┌──────────────────────────────┐
│ AI Features [09]             │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│ AI Prompting [32]            │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│ Next Action Prediction [46]  │
└──────────────────────────────┘
```

## Document Interdependency Map

Below is a detailed map showing how documents reference and depend on each other:

| Document                   | Depends On                         | Informs                          |
|----------------------------|------------------------------------|---------------------------------|
| 01_Project_Architecture    | None                               | All Stratospheric Documents     |
| 02_GPUI                    | 01_Project_Architecture            | 25_Entity_System, 28_UI_Layout  |
| 03_TextEditorCore          | 01_Project_Architecture            | 13_BufferAndRope, 14_Cursor     |
| 04_LanguageIntelligence    | 01_Project_Architecture            | 24_TreeSitter, 30_LSP           |
| 13_BufferAndRope           | 03_TextEditorCore                  | 23_RopeAlgorithms, 52_RopeImpl  |
| 24_TreeSitterIntegration   | 04_LanguageIntelligence            | 47_TreeSitterImplementation     |
| 25_EntitySystem            | 02_GPUI                            | 41_Swift_EntitySystem, 49_ReactiveUI |
| 37_EventHandling           | 02_GPUI, 25_EntitySystem           | 51_InputHandlingSystem          |
| 41_Swift_EntitySystem      | 25_EntitySystem, 34_MemoryMgmt     | Swift implementation            |
| 46_NextActionPrediction    | 09_AIFeatures, 32_AIPrompting      | Swift implementation            |
| 49_ReactiveUIImpl          | 02_GPUI, 36_UIRendering            | 42_Swift_ReactiveUI            |
| 50_PerformancePatterns     | Multiple subsystems                | All implementation documents    |
| 51_InputHandlingSystem     | 37_EventHandling                   | Swift implementation            |
| 52_RopeImplementation      | 13_BufferAndRope, 23_RopeAlgo      | 43_Swift_TextBuffer            |
| 53_CommandDispatchSystem   | 11_CommandSystem                   | Swift implementation            |
| 54_ErrorHandlingPatterns   | Multiple subsystems                | All implementation documents    |
| 55_MemoryManagementPatterns| 34_MemoryManagement                | All implementation documents    |
| B01_EntitySystem_Bridge    | 02_GPUI, 25_EntitySystem, 41_Swift | Implementation guidance         |
| B02_TextBuffer_Bridge      | 03_TextEditorCore, 13_Buffer, 52_Rope | Implementation guidance      |
| B03_UI_Bridge              | 02_GPUI, 16_UIComponents, 49_ReactiveUI | Implementation guidance    |
| B04_LanguageIntel_Bridge   | 04_LanguageIntelligence, 24_TreeSitter, 30_LSP | Implementation guidance |

## Recommended Reading Paths

### For Architecture Understanding
1. [01_OrbitalView_ProjectArchitecture.md](01_OrbitalView_ProjectArchitecture.md)
2. [02_StratosphericView_GPUI.md](02_StratosphericView_GPUI.md)
3. [03_StratosphericView_TextEditorCore.md](03_StratosphericView_TextEditorCore.md)
4. [04_StratosphericView_LanguageIntelligence.md](04_StratosphericView_LanguageIntelligence.md)
5. [06_StratosphericView_CollaborationSystem.md](06_StratosphericView_CollaborationSystem.md)
6. [SubsystemRelationshipMap.md](SubsystemRelationshipMap.md)

### For Implementation Focus
1. [01_OrbitalView_ProjectArchitecture.md](01_OrbitalView_ProjectArchitecture.md)
2. Relevant Bridge Documents (B01-B04)
3. Relevant Ground Level Documents
4. Cross-Cutting Concerns (50, 54, 55)

### For Swift Implementation
1. [01_OrbitalView_ProjectArchitecture.md](01_OrbitalView_ProjectArchitecture.md)
2. Relevant Bridge Documents (B01-B04)
3. Swift-specific Documents (41-45)
4. [MinimumViableSwiftPrototype.md](MinimumViableSwiftPrototype.md)

### For Understanding Specific Subsystems

#### Text Editing
1. [03_StratosphericView_TextEditorCore.md](03_StratosphericView_TextEditorCore.md)
2. [13_AtmosphericView_BufferAndRope.md](13_AtmosphericView_BufferAndRope.md)
3. [23_CloudLevel_RopeAlgorithms.md](23_CloudLevel_RopeAlgorithms.md)
4. [B02_TextBuffer_TechnicalBridge.md](B02_TextBuffer_TechnicalBridge.md)
5. [52_GroundLevel_RopeImplementation.md](52_GroundLevel_RopeImplementation.md)

#### UI System
1. [02_StratosphericView_GPUI.md](02_StratosphericView_GPUI.md)
2. [16_AtmosphericView_UIComponents.md](16_AtmosphericView_UIComponents.md)
3. [28_CloudLevel_UILayout.md](28_CloudLevel_UILayout.md)
4. [B03_UI_TechnicalBridge.md](B03_UI_TechnicalBridge.md)
5. [49_GroundLevel_ReactiveUIImplementation.md](49_GroundLevel_ReactiveUIImplementation.md)

#### Language Intelligence
1. [04_StratosphericView_LanguageIntelligence.md](04_StratosphericView_LanguageIntelligence.md)
2. [24_CloudLevel_TreeSitterIntegration.md](24_CloudLevel_TreeSitterIntegration.md)
3. [30_CloudLevel_LanguageServerProtocol.md](30_CloudLevel_LanguageServerProtocol.md)
4. [B04_LanguageIntelligence_TechnicalBridge.md](B04_LanguageIntelligence_TechnicalBridge.md)
5. [47_GroundLevel_TreeSitterImplementation.md](47_GroundLevel_TreeSitterImplementation.md)
6. [48_GroundLevel_LanguageServerImplementation.md](48_GroundLevel_LanguageServerImplementation.md)

## Documentation Progress

### Completed Documents
- ✅ All Orbital, Stratospheric, and Atmospheric views
- ✅ All Cloud Level documents
- ✅ Bridge documents connecting major subsystems
- ✅ Core Swift implementation documents
- ✅ Key Ground Level implementation documents
- ✅ Cross-cutting concerns for performance, error handling, and memory management

### In Progress or Planned
- 🔄 Additional Cross-cutting concerns
- 🔄 Integration documents showing subsystem interactions
- 🔄 Additional implementation-focused documents
- 🔄 Test frameworks and approaches
- 🔄 Extension system implementation details

## Topic Finding Guide

To find information on specific topics:

### Entity System and UI
- Architecture: [02_StratosphericView_GPUI.md](02_StratosphericView_GPUI.md)
- Implementation: [25_CloudLevel_EntitySystem.md](25_CloudLevel_EntitySystem.md), [28_CloudLevel_UILayout.md](28_CloudLevel_UILayout.md)
- Bridge: [B01_EntitySystem_TechnicalBridge.md](B01_EntitySystem_TechnicalBridge.md), [B03_UI_TechnicalBridge.md](B03_UI_TechnicalBridge.md)
- Concrete: [49_GroundLevel_ReactiveUIImplementation.md](49_GroundLevel_ReactiveUIImplementation.md), [51_GroundLevel_InputHandlingSystem.md](51_GroundLevel_InputHandlingSystem.md)
- Swift: [41_Swift_EntitySystem.md](41_Swift_EntitySystem.md), [42_Swift_ReactiveUI.md](42_Swift_ReactiveUI.md)

### Text Editing
- Architecture: [03_StratosphericView_TextEditorCore.md](03_StratosphericView_TextEditorCore.md)
- Components: [13_AtmosphericView_BufferAndRope.md](13_AtmosphericView_BufferAndRope.md), [14_AtmosphericView_CursorAndSelection.md](14_AtmosphericView_CursorAndSelection.md)
- Implementation: [23_CloudLevel_RopeAlgorithms.md](23_CloudLevel_RopeAlgorithms.md)
- Bridge: [B02_TextBuffer_TechnicalBridge.md](B02_TextBuffer_TechnicalBridge.md)
- Concrete: [52_GroundLevel_RopeImplementation.md](52_GroundLevel_RopeImplementation.md)
- Swift: [43_Swift_TextBuffer.md](43_Swift_TextBuffer.md)

### Language Features
- Architecture: [04_StratosphericView_LanguageIntelligence.md](04_StratosphericView_LanguageIntelligence.md)
- Implementation: [24_CloudLevel_TreeSitterIntegration.md](24_CloudLevel_TreeSitterIntegration.md), [30_CloudLevel_LanguageServerProtocol.md](30_CloudLevel_LanguageServerProtocol.md)
- Bridge: [B04_LanguageIntelligence_TechnicalBridge.md](B04_LanguageIntelligence_TechnicalBridge.md)
- Concrete: [47_GroundLevel_TreeSitterImplementation.md](47_GroundLevel_TreeSitterImplementation.md), [48_GroundLevel_LanguageServerImplementation.md](48_GroundLevel_LanguageServerImplementation.md)

### Cross-Cutting Concerns
- Performance: [50_CrossCutting_PerformanceOptimizationPatterns.md](50_CrossCutting_PerformanceOptimizationPatterns.md)
- Error Handling: [54_CrossCutting_ErrorHandlingRecoveryPatterns.md](54_CrossCutting_ErrorHandlingRecoveryPatterns.md)
- Memory Management: [55_CrossCutting_MemoryManagementPatterns.md](55_CrossCutting_MemoryManagementPatterns.md)

## Contribution Guidelines

When adding a new document to the Mission Cabbage documentation:

1. Use the appropriate naming convention:
   - Orbital Layer: `01_OrbitalView_*.md`
   - Stratospheric Layer: `XX_StratosphericView_*.md`
   - Atmospheric Layer: `XX_AtmosphericView_*.md`
   - Cloud Layer: `XX_CloudLevel_*.md`
   - Ground Layer: `XX_GroundLevel_*.md`
   - Bridge Documents: `BXX_*_TechnicalBridge.md`
   - Cross-Cutting: `XX_CrossCutting_*.md`
   - Swift Implementation: `XX_Swift_*.md`

2. Update this connectivity guide to include:
   - The new document in the appropriate section
   - Dependencies and relationships with existing documents
   - Any new reading paths it enables

3. Ensure your document:
   - References related documents
   - Includes links to dependent and informing documents
   - Follows the document template in [00_DocumentTemplate.md](00_DocumentTemplate.md)

This connectivity map will be updated as new documents are added to the Mission Cabbage project.