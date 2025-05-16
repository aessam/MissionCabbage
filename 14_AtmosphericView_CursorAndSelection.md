# Atmospheric View: Cursor and Selection

## Purpose

The Cursor and Selection system manages user interaction with text content, enabling precise positioning, text selection, and multi-cursor editing. This system builds on the Buffer and Anchor capabilities to create a responsive and intuitive text editing experience, handling everything from simple cursor movements to complex multi-selection operations.

## Core Concepts

### Cursor Fundamentals

```mermaid
graph TD
    subgraph CursorSystem
        C[Cursor] --> Pos[Position]
        C --> Goal[Goal Position]
        C --> VState[Visibility State]
        
        Pos --> PA[Primary Anchor]
        Pos --> DP[Display Position]
        
        Goal --> GC[Goal Column]
        Goal --> GPref[Goal Preferences]
        
        VState --> Visible[Visibility]
        VState --> Style[Cursor Style]
        VState --> Blink[Blink State]
    end
    
    subgraph CursorTypes
        RC[Regular Cursor]
        BC[Block Cursor]
        IC[Insert Cursor]
        OC[Overwrite Cursor]
        VC[Virtual Cursor]
    end
    
    C -.-> RC & BC & IC & OC & VC
    
    style C fill:#f96,stroke:#333,stroke-width:2px
    style Pos,Goal,VState fill:#bbf,stroke:#33f,stroke-width:1px
    style PA,DP,GC,GPref,Visible,Style,Blink fill:#dfd,stroke:#393,stroke-width:1px
    style RC,BC,IC,OC,VC fill:#eee,stroke:#999,stroke-width:1px
```

- **Cursor**: Visual indicator of insertion point
- **Position**: Location in buffer (row, column)
- **Goal Position**: Desired column for vertical movement
- **Buffer Anchor**: Stable reference to buffer position
- **Cursor Style**: Visual appearance (block, line, etc.)
- **Cursor State**: Visibility, blink status, etc.

### Selection Model

```mermaid
graph TD
    subgraph SelectionModel
        S[Selection] --> AR[Active Region]
        S --> SD[Selection Direction]
        S --> SM[Selection Mode]
        
        AR --> Head[Head Anchor]
        AR --> Tail[Tail Anchor]
        AR --> Range[Selected Range]
        
        SD --> F[Forward]
        SD --> B[Backward]
        
        SM --> C[Character Mode]
        SM --> L[Line Mode]
        SM --> BL[Block Mode]
        SM --> ST[Semantic Mode]
    end
    
    subgraph SelectionBehavior
        SE[Selection Expansion]
        SC[Selection Contraction]
        SS[Selection Shift]
        SK[Selection Skip]
    end
    
    S --> SE & SC & SS & SK
    
    style S fill:#f96,stroke:#333,stroke-width:2px
    style AR,SD,SM fill:#bbf,stroke:#33f,stroke-width:1px
    style Head,Tail,Range,F,B,C,L,BL,ST fill:#dfd,stroke:#393,stroke-width:1px
    style SE,SC,SS,SK fill:#ffc,stroke:#660,stroke-width:1px
```

- **Selection**: Range of text between two positions
- **Head**: Leading edge of selection (cursor position)
- **Tail**: Trailing edge of selection (anchor position)
- **Direction**: Whether selection runs forward or backward
- **Selection Mode**: Character, line, or block-based selection
- **Semantic Selection**: Language-aware selection units

### Multi-Cursor System

```mermaid
graph TD
    subgraph MultiCursorSystem
        MC[Multi-Cursor Controller] --> PC[Primary Cursor]
        MC --> SC[Secondary Cursors]
        MC --> CA[Cursor Addition]
        MC --> CR[Cursor Removal]
        
        PC --> MainCursor[Main Editing Cursor]
        PC --> Leader[Movement Leader]
        
        SC --> CursorSet[Cursor Collection]
        SC --> Ordering[Cursor Ordering]
        
        CA --> Manual[Manual Addition]
        CA --> Find[Find/Replace Addition]
        CA --> Column[Column Selection]
        
        CR --> Merge[Cursor Merging]
        CR --> Delete[Cursor Deletion]
        CR --> Undo[Operation Undo]
    end
    
    style MC fill:#f96,stroke:#333,stroke-width:2px
    style PC,SC,CA,CR fill:#bbf,stroke:#33f,stroke-width:1px
    style MainCursor,Leader,CursorSet,Ordering,Manual,Find,Column,Merge,Delete,Undo fill:#dfd,stroke:#393,stroke-width:1px
```

- **Multi-Cursor**: Multiple independent cursor positions
- **Primary Cursor**: Main cursor among multiple cursors
- **Secondary Cursors**: Additional editing positions
- **Cursor Set**: Collection of all active cursors
- **Cursor Addition**: Ways to create new cursors
- **Cursor Removal**: Ways to reduce cursor count

### Cursor Navigation

- **Movement**: Cursor position changes
- **Vertical Motion**: Moving up and down lines
- **Horizontal Motion**: Moving left and right
- **Word Navigation**: Moving by word boundaries
- **Smart Motion**: Context-aware movement
- **Jump Lists**: History of significant positions

### Selection Operations

- **Extend**: Grow the selection
- **Shrink**: Reduce the selection
- **Flip**: Swap head and tail
- **Multiple Selections**: Independent selection ranges
- **Selection Modification**: Operations on selected text
- **Selection Persistence**: Maintaining selection through operations

## Architecture

### Core Components

```mermaid
classDiagram
    class CursorController {
        +cursors: Vec~Cursor~
        +primary_cursor_index: usize
        +new() CursorController
        +set_position(point)
        +add_cursor(point)
        +remove_cursor(index)
        +active_cursor() Cursor
        +move_up(count)
        +move_down(count)
        +move_left(count)
        +move_right(count)
        +select_up(count)
        +select_down(count)
        +select_all()
    }
    
    class Cursor {
        +head: Anchor
        +tail: Option~Anchor~
        +goal_column: Option~u32~
        +new(point) Cursor
        +new_with_selection(head, tail) Cursor
        +position() Point
        +set_position(point)
        +has_selection() bool
        +selection_range() Option~Range~
        +clear_selection()
        +select_to(point)
        +select_range(range)
    }
    
    class Selection {
        +head: Anchor
        +tail: Anchor
        +mode: SelectionMode
        +new(range, mode) Selection
        +range() Range
        +is_reversed() bool
        +flip()
        +contains(point) bool
        +text(buffer) String
        +set_mode(mode)
    }
    
    class MultiCursor {
        +cursors: Vec~Cursor~
        +active_index: usize
        +new() MultiCursor
        +add_cursor(cursor)
        +remove_cursor(index)
        +active_cursor() Cursor
        +set_active_cursor(index)
        +merge_overlapping()
        +sort_cursors()
        +for_each_cursor(fn)
    }
    
    class CursorView {
        +cursor: Cursor
        +style: CursorStyle
        +visibility: bool
        +blink_state: bool
        +new(cursor) CursorView
        +render(display)
        +update_style(style)
        +show()
        +hide()
        +toggle_blink()
    }
    
    class SelectionView {
        +selection: Selection
        +style: SelectionStyle
        +new(selection) SelectionView
        +render(display)
        +update_style(style)
    }
    
    CursorController --> Cursor
    Cursor o-- Selection
    MultiCursor --> Cursor
    CursorView --> Cursor
    SelectionView --> Selection
    
    note for CursorController "Manages all cursor operations"
    note for Cursor "Single cursor with optional selection"
    note for Selection "Range between head and tail anchors"
    note for MultiCursor "Manages multiple independent cursors"
    note for CursorView "Visual representation of cursor"
    note for SelectionView "Visual representation of selection"
```

### Detailed Component Structure

```mermaid
classDiagram
    class EditorState {
        +buffer: Buffer
        +cursor_controller: CursorController
        +display_map: DisplayMap
        +new(buffer) EditorState
        +handle_input(input)
        +execute_command(command)
        +render_view(view)
    }
    
    class CursorController {
        +cursors: Vec~Cursor~
        +primary_cursor_index: usize
        +mode: CursorMode
        +new() CursorController
        +active_cursor() Cursor
        +move_cursors(direction, count)
        +select(direction, count)
        +extend_selections(direction, count)
        +add_cursor_above()
        +add_cursor_below()
        +select_all()
        +select_word()
        +select_line()
    }
    
    class Cursor {
        +head: Anchor
        +tail: Option~Anchor~
        +goal_column: Option~u32~
        +new(point) Cursor
        +position() Point
        +has_selection() bool
        +selection_range() Option~Range~
        +select_to(point)
        +extend_to(point)
        +clear_selection()
    }
    
    class DisplayMap {
        +buffer: Buffer
        +line_wrapping: bool
        +tab_size: u8
        +new(buffer) DisplayMap
        +buffer_to_display_position(point) DisplayPoint
        +display_to_buffer_position(display_point) Point
        +is_position_visible(point) bool
        +fold_at(range)
        +unfold_at(range)
    }
    
    class InputHandler {
        +key_map: KeyMap
        +mode: InputMode
        +new(key_map) InputHandler
        +handle_key_event(event, controller)
        +handle_mouse_event(event, controller)
        +handle_gesture_event(event, controller)
        +set_mode(mode)
    }
    
    class SelectionSet {
        +selections: Vec~Selection~
        +active_index: usize
        +new() SelectionSet
        +add_selection(selection)
        +remove_selection(index)
        +active_selection() Selection
        +merge_overlapping()
        +clear()
    }
    
    EditorState --> CursorController
    EditorState --> DisplayMap
    CursorController o-- Cursor
    EditorState --> InputHandler
    CursorController o-- SelectionSet
    
    note for EditorState "Central editor state manager"
    note for CursorController "Manages all cursor operations"
    note for DisplayMap "Maps buffer positions to display"
    note for InputHandler "Processes user input"
    note for SelectionSet "Manages multiple selections"
```

### Data Flow

#### Cursor Movement Flow

```mermaid
sequenceDiagram
    participant User
    participant Input as Input Handler
    participant CC as Cursor Controller
    participant C as Cursor
    participant B as Buffer
    participant DM as Display Map
    participant View as Editor View
    
    User->>Input: Press arrow key
    Input->>CC: Request cursor movement
    
    CC->>C: Get current cursor
    C->>CC: Current cursor state
    
    CC->>C: Calculate new position
    C->>B: Get buffer constraints
    B-->>C: Buffer information
    
    alt With goal column
        C->>C: Apply goal column
    else Without goal column
        C->>C: Use target column
    end
    
    C->>DM: Map to valid position
    DM-->>C: Valid buffer position
    
    C->>C: Update cursor position
    C-->>CC: Position updated
    
    CC->>View: Update display
    View-->>User: Show updated cursor
    
    Note over C,DM: Goal column maintains desired<br>horizontal position through<br>vertical movement
```

#### Selection Creation Flow

```mermaid
sequenceDiagram
    participant User
    participant Input as Input Handler
    participant CC as Cursor Controller
    participant C as Cursor
    participant S as Selection
    participant B as Buffer
    participant View as Editor View
    
    User->>Input: Shift+arrow key
    Input->>CC: Request selection
    
    CC->>C: Get current cursor
    C->>CC: Current cursor state
    
    alt No existing selection
        C->>C: Create selection at cursor
        C->>S: Initialize with current position
    else Existing selection
        C->>S: Get existing selection
    end
    
    CC->>S: Extend selection
    S->>B: Get text constraints
    B-->>S: Buffer information
    
    S->>S: Calculate new range
    S-->>C: Update selection range
    
    C-->>CC: Selection updated
    
    CC->>View: Update display
    View-->>User: Show updated selection
    
    Note over C,S: Selection maintains head and tail<br>anchors through edits
```

#### Multi-Cursor Creation Flow

```mermaid
sequenceDiagram
    participant User
    participant Input as Input Handler
    participant CC as Cursor Controller
    participant MC as Multi-Cursor
    participant NC as New Cursor
    participant B as Buffer
    participant View as Editor View
    
    User->>Input: Ctrl+Alt+arrow down
    Input->>CC: Request additional cursor
    
    CC->>MC: Get cursor set
    MC->>CC: Current cursors
    
    CC->>B: Get buffer state
    B-->>CC: Buffer information
    
    CC->>NC: Create new cursor
    NC->>B: Get position constraints
    B-->>NC: Valid position info
    
    NC->>NC: Initialize cursor
    NC-->>CC: New cursor created
    
    CC->>MC: Add cursor to set
    MC->>MC: Update cursor collection
    MC-->>CC: Cursor added
    
    CC->>View: Update display
    View-->>User: Show multiple cursors
    
    Note over MC,NC: Cursors are kept ordered<br>by buffer position
```

#### Find and Multi-Select Flow

```mermaid
sequenceDiagram
    participant User
    participant Input as Input Handler
    participant CC as Cursor Controller
    participant Search as Search Engine
    participant B as Buffer
    participant MC as Multi-Cursor
    participant View as Editor View
    
    User->>Input: Find all (Ctrl+Shift+F)
    Input->>CC: Command: find all
    
    CC->>Search: Search for pattern
    Search->>B: Search buffer content
    B-->>Search: Buffer text
    
    Search->>Search: Find all matches
    Search-->>CC: Match positions
    
    CC->>MC: Clear existing cursors
    CC->>MC: Create cursor at each match
    
    loop For each match
        MC->>MC: Add cursor at match
    end
    
    MC-->>CC: Cursors created
    
    CC->>View: Update display
    View-->>User: Show cursors at all matches
    
    Note over Search,MC: Matches become the new<br>cursor set, replacing previous cursors
```

#### Column Selection Flow

```mermaid
stateDiagram-v2
    [*] --> SingleCursor
    
    SingleCursor --> MouseDragStart: Mouse down + Alt
    MouseDragStart --> ColumnSelectionActive: Mouse drag
    
    ColumnSelectionActive --> ColumnSelectionActive: Mouse movement
    ColumnSelectionActive --> ColumnSelectionComplete: Mouse up
    
    ColumnSelectionComplete --> MultiCursorEditing: Enter edit mode
    
    state ColumnSelectionActive {
        [*] --> CalculateRectangle
        CalculateRectangle --> CreateColumnCursors
        CreateColumnCursors --> UpdateDisplay
        UpdateDisplay --> [*]
    }
    
    state MultiCursorEditing {
        [*] --> SynchronizedEditing
        SynchronizedEditing --> IndividualEditing: Cursor divergence
    }
    
    MultiCursorEditing --> SingleCursor: Escape key
    
    note right of ColumnSelectionActive
        Selection forms a rectangle
        between start and end points
    end note
    
    note right of MultiCursorEditing
        All cursors initially perform
        identical edits in parallel
    end note
```

## Key Interfaces

### Cursor Controller

```
// Conceptual interface, not actual Rust code
CursorController {
    // Cursor management
    new() -> CursorController
    new_with_buffer(buffer: Buffer) -> CursorController
    active_cursor() -> Cursor
    cursors() -> Vec<Cursor>
    cursor_count() -> usize
    set_active_cursor(index: usize) -> Result<()>
    add_cursor(point: Point) -> CursorId
    remove_cursor(index: usize)
    clear_cursors()
    merge_cursors()
    
    // Basic movement
    move_up(count: usize)
    move_down(count: usize)
    move_left(count: usize)
    move_right(count: usize)
    move_to_line_start()
    move_to_line_end()
    move_to_first_line()
    move_to_last_line()
    
    // Word movement
    move_word_left()
    move_word_right()
    move_to_word_start()
    move_to_word_end()
    
    // Page movement
    page_up()
    page_down()
    
    // Semantic movement
    move_to_enclosing_bracket()
    move_to_next_paragraph()
    move_to_previous_paragraph()
    
    // Selection
    select_up(count: usize)
    select_down(count: usize)
    select_left(count: usize)
    select_right(count: usize)
    select_to_line_start()
    select_to_line_end()
    select_word()
    select_line()
    select_paragraph()
    select_all()
    
    // Multiple cursors
    add_cursor_above()
    add_cursor_below()
    split_selection_into_cursors()
    select_next_occurrence()
    select_all_occurrences()
    
    // Column selection
    start_column_selection(start: Point)
    update_column_selection(end: Point)
    end_column_selection()
    
    // Visibility
    visible_cursors() -> Vec<Cursor>
    scroll_to_cursor()
    ensure_cursor_visible()
    
    // Events
    on_cursor_changed(callback: Callback) -> Subscription
    on_selection_changed(callback: Callback) -> Subscription
}
```

### Cursor

```
// Conceptual interface, not actual Rust code
Cursor {
    // Creation
    new(point: Point) -> Cursor
    new_with_selection(head: Point, tail: Point) -> Cursor
    
    // Position
    position() -> Point
    set_position(point: Point)
    display_position(display_map: &DisplayMap) -> DisplayPoint
    
    // Goal column
    goal_column() -> Option<u32>
    set_goal_column(column: u32)
    clear_goal_column()
    
    // Selection
    has_selection() -> bool
    selection_range() -> Option<Range>
    select_to(point: Point)
    clear_selection()
    is_selection_reversed() -> bool
    flip_selection()
    
    // Anchor access
    head_anchor() -> Anchor
    tail_anchor() -> Option<Anchor>
    
    // State
    is_valid() -> bool
    clone() -> Cursor
    
    // Buffer interaction
    selected_text(buffer: &Buffer) -> String
    delete_selection(buffer: &mut Buffer) -> Option<Transaction>
}
```

### Selection

```
// Conceptual interface, not actual Rust code
Selection {
    // Creation
    new(range: Range) -> Selection
    new_with_mode(range: Range, mode: SelectionMode) -> Selection
    
    // Range
    range() -> Range
    set_range(range: Range)
    contains(point: Point) -> bool
    is_empty() -> bool
    
    // Anchors
    head() -> Point
    tail() -> Point
    set_head(point: Point)
    set_tail(point: Point)
    
    // Direction
    is_reversed() -> bool
    flip()
    
    // Mode
    mode() -> SelectionMode
    set_mode(mode: SelectionMode)
    
    // Buffer interaction
    text(buffer: &Buffer) -> String
    delete(buffer: &mut Buffer) -> Transaction
    
    // Utility
    clone() -> Selection
    merge(other: &Selection) -> Selection
    overlaps(other: &Selection) -> bool
    is_adjacent(other: &Selection) -> bool
}
```

### Multi-Cursor System

```
// Conceptual interface, not actual Rust code
MultiCursor {
    // Management
    new() -> MultiCursor
    add_cursor(cursor: Cursor) -> CursorId
    remove_cursor(id: CursorId)
    get_cursor(id: CursorId) -> Option<&Cursor>
    active_cursor() -> &Cursor
    set_active_cursor(id: CursorId)
    cursor_count() -> usize
    
    // Operations
    merge_overlapping()
    sort_cursors()
    filter_cursors(predicate: Fn(&Cursor) -> bool)
    for_each_cursor(operation: Fn(&mut Cursor))
    map_cursors<R>(operation: Fn(&Cursor) -> R) -> Vec<R>
    
    // Bulk operations
    move_all(direction: Direction, count: usize)
    select_all(direction: Direction, count: usize)
    clear_all_selections()
    
    // Column operations
    create_column_cursors(start: Point, end: Point) -> Vec<CursorId>
    
    // Events
    on_cursors_changed(callback: Callback) -> Subscription
}
```

### Display Integration

```
// Conceptual interface, not actual Rust code
CursorView {
    // Creation
    new(cursor: Cursor) -> CursorView
    
    // Rendering
    render(display: &mut Display)
    update(cursor: &Cursor)
    
    // Styling
    set_style(style: CursorStyle)
    set_color(color: Color)
    
    // Visibility
    show()
    hide()
    is_visible() -> bool
    
    // Blinking
    set_blink_enabled(enabled: bool)
    toggle_blink()
    blink_interval() -> Duration
    set_blink_interval(interval: Duration)
}

SelectionView {
    // Creation
    new(selection: Selection) -> SelectionView
    
    // Rendering
    render(display: &mut Display)
    update(selection: &Selection)
    
    // Styling
    set_style(style: SelectionStyle)
    set_colors(foreground: Color, background: Color)
    
    // Options
    set_show_inactive(show: bool)
    set_round_corners(round: bool)
}
```

## Behavioral Patterns

### Cursor Movement Patterns

```mermaid
graph TD
    subgraph CursorMovement
        CM[Cursor Movement] --> BM[Basic Movement]
        CM --> WM[Word Movement]
        CM --> SM[Semantic Movement]
        
        BM --> U[Up]
        BM --> D[Down]
        BM --> L[Left]
        BM --> R[Right]
        BM --> LS[Line Start]
        BM --> LE[Line End]
        
        WM --> WL[Word Left]
        WM --> WR[Word Right]
        WM --> WSt[Word Start]
        WM --> WE[Word End]
        
        SM --> PU[Paragraph Up]
        SM --> PD[Paragraph Down]
        SM --> EB[Enclosing Bracket]
        SM --> FF[Function Forward]
        SM --> FB[Function Back]
    end
    
    subgraph MovementBehaviors
        GC[Goal Column]
        VC[Virtual Cursor]
        WB[Word Boundaries]
        SU[Semantic Units]
    end
    
    U & D --> GC
    WL & WR --> WB
    PU & PD & FF & FB --> SU
    LS & LE --> VC
    
    style CM fill:#f96,stroke:#333,stroke-width:2px
    style BM,WM,SM fill:#bbf,stroke:#33f,stroke-width:1px
    style U,D,L,R,LS,LE,WL,WR,WSt,WE,PU,PD,EB,FF,FB fill:#dfd,stroke:#393,stroke-width:1px
    style GC,VC,WB,SU fill:#ffc,stroke:#660,stroke-width:1px
```

#### Key Movement Concepts

1. **Goal Column**: Maintaining desired horizontal position during vertical movement
2. **Word Boundaries**: Language-aware word start/end detection
3. **Virtual Positions**: Positions beyond physical text (line end, etc.)
4. **Wrapping Behavior**: How cursor moves at line boundaries
5. **Semantic Navigation**: Movement based on code structure
6. **Visual vs. Logical**: Movement in displayed vs. stored text

### Selection Patterns

```mermaid
graph TD
    subgraph SelectionModes
        SM[Selection Modes]
        CM[Character Mode]
        LM[Line Mode]
        BM[Block Mode]
        SEM[Semantic Mode]
    end
    
    SM --> CM & LM & BM & SEM
    
    subgraph SelectionOperations
        SO[Selection Operations]
        E[Extend]
        S[Shrink]
        F[Flip]
        MT[Move/Transform]
        
        SO --> E & S & F & MT
    end
    
    subgraph SelectionActions
        Word[Select Word]
        Line[Select Line]
        Para[Select Paragraph]
        All[Select All]
        Next[Select Next Occurrence]
        Bracket[Select Inside Brackets]
    end
    
    Word & Line & Para & Bracket --> SEM
    Next --> CM
    
    style SM,SO fill:#f96,stroke:#333,stroke-width:2px
    style CM,LM,BM,SEM,E,S,F,MT fill:#bbf,stroke:#33f,stroke-width:1px
    style Word,Line,Para,All,Next,Bracket fill:#dfd,stroke:#393,stroke-width:1px
```

#### Key Selection Concepts

1. **Mode-specific Behavior**: Different selection modes for different tasks
2. **Extension Direction**: How selection grows in different directions
3. **Head/Tail Dynamics**: How selection endpoints behave during editing
4. **Selection Persistence**: Maintaining selection through operations
5. **Selection Transformation**: Operations that modify selection shape
6. **Selection Merging**: Combining overlapping selections

### Multi-Cursor Patterns

```mermaid
graph TD
    subgraph MultiCursorPatterns
        MC[Multi-Cursor Operations]
        
        CA[Cursor Addition]
        CO[Cursor Organization]
        CE[Cursor Editing]
        CC[Cursor Cleanup]
        
        MC --> CA & CO & CE & CC
        
        CA --> Manual[Manual Addition]
        CA --> Find[Find/Replace]
        CA --> Column[Column Selection]
        CA --> Split[Split Selection]
        
        CO --> Sorting[Position Sorting]
        CO --> Grouping[Logical Grouping]
        CO --> Primary[Primary Selection]
        
        CE --> Synchronized[Synchronized Edits]
        CE --> Divergent[Divergent Edits]
        CE --> Individual[Individual Operations]
        
        CC --> Merge[Merge Overlapping]
        CC --> Remove[Remove Duplicates]
        CC --> Collapse[Collapse to Primary]
    end
    
    style MC fill:#f96,stroke:#333,stroke-width:2px
    style CA,CO,CE,CC fill:#bbf,stroke:#33f,stroke-width:1px
    style Manual,Find,Column,Split,Sorting,Grouping,Primary,Synchronized,Divergent,Individual,Merge,Remove,Collapse fill:#dfd,stroke:#393,stroke-width:1px
```

#### Key Multi-Cursor Concepts

1. **Creation Methods**: Different ways to create multiple cursors
2. **Primary Cursor**: Special status of main cursor
3. **Cursor Synchronization**: How multiple cursors behave during edits
4. **Cursor Lifetime**: When cursors are created and destroyed
5. **Cursor Independence**: How cursors operate independently
6. **Cursor Merging**: Rules for cursor collision and overlap

## State Management

### Cursor State

```mermaid
stateDiagram-v2
    [*] --> Single
    
    Single --> WithSelection: Select
    Single --> Multi: Add cursor
    
    WithSelection --> Single: Clear selection
    WithSelection --> Multi: Add cursor
    
    Multi --> Single: Collapse to one
    Multi --> MultiWithSelection: Select
    
    MultiWithSelection --> Multi: Clear selection
    
    state Multi {
        [*] --> Synchronized
        Synchronized --> Diverged: Different edits
        Diverged --> Synchronized: Reset
        
        state Synchronized {
            SamePosition
            SameLine
            SimilarContext
        }
    }
    
    note right of WithSelection
        Single cursor with active
        text selection
    end note
    
    note right of Multi
        Multiple independent cursors
        for parallel editing
    end note
```

#### Key State Elements

1. **Cursor Position State**: Current location of each cursor
2. **Selection State**: Active selections for each cursor
3. **Goal Column State**: Desired column for vertical movement
4. **Visual State**: Cursor appearance and visibility
5. **Mode State**: Current editing mode affecting behavior
6. **History State**: Previous cursor positions for navigation

### Position Tracking

```mermaid
graph TD
    subgraph PositionTracking
        P[Position] --> BP[Buffer Position]
        P --> DP[Display Position]
        P --> VP[Visual Position]
        
        BP --> Row[Row]
        BP --> Col[Column]
        
        DP --> DRow[Display Row]
        DP --> DCol[Display Column]
        
        VP --> X[X Coordinate]
        VP --> Y[Y Coordinate]
    end
    
    subgraph PositionMappings
        B2D[Buffer to Display]
        D2B[Display to Buffer]
        D2V[Display to Visual]
        V2D[Visual to Display]
    end
    
    BP --> B2D --> DP
    DP --> D2B --> BP
    DP --> D2V --> VP
    VP --> V2D --> DP
    
    style P fill:#f96,stroke:#333,stroke-width:2px
    style BP,DP,VP fill:#bbf,stroke:#33f,stroke-width:1px
    style Row,Col,DRow,DCol,X,Y fill:#dfd,stroke:#393,stroke-width:1px
    style B2D,D2B,D2V,V2D fill:#ffc,stroke:#660,stroke-width:1px
```

#### Key Position Concepts

1. **Buffer Coordinates**: Positions in the actual text content
2. **Display Coordinates**: Positions in the rendered view (with wrapping, etc.)
3. **Anchor Positions**: Stable positions maintained through edits
4. **Visual Coordinates**: Pixel positions on screen
5. **Position Validation**: Ensuring positions are valid in buffer
6. **Position Mapping**: Converting between coordinate systems

### Selection State

```mermaid
graph TD
    subgraph SelectionState
        SS[Selection State]
        Active[Active Selection]
        History[Selection History]
        Multiple[Multiple Selections]
        
        SS --> Active & History & Multiple
        
        Active --> Range[Current Range]
        Active --> Mode[Selection Mode]
        Active --> Direction[Selection Direction]
        
        History --> Previous[Previous Selections]
        History --> Restore[Restore Points]
        
        Multiple --> Ordered[Ordered Collection]
        Multiple --> Primary[Primary Selection]
        Multiple --> Relationships[Selection Relationships]
    end
    
    style SS fill:#f96,stroke:#333,stroke-width:2px
    style Active,History,Multiple fill:#bbf,stroke:#33f,stroke-width:1px
    style Range,Mode,Direction,Previous,Restore,Ordered,Primary,Relationships fill:#dfd,stroke:#393,stroke-width:1px
```

#### Key Selection State Elements

1. **Range State**: Current selected ranges for each cursor
2. **Mode State**: Character, line, or block selection mode
3. **Direction State**: Whether selection runs forward or backward
4. **History State**: Previous selections for navigation
5. **Multiple Selection State**: State of multiple independent selections
6. **Ordering State**: Relative ordering of multiple selections

## Swift Considerations

### Swift Implementation Approach

```mermaid
graph TD
    subgraph SwiftImplementation
        SI[Swift Implementation]
        DS[Data Structures]
        PA[Protocol Architecture]
        UI[User Interface]
        IC[Input Coordination]
        
        SI --> DS & PA & UI & IC
        
        DS --> ValueTypes[Value Types]
        DS --> ReferenceTypes[Reference Types]
        
        PA --> Protocols[Protocol Definitions]
        PA --> Extensions[Protocol Extensions]
        
        UI --> UIKit[UIKit Integration]
        UI --> SwiftUI[SwiftUI Components]
        
        IC --> Responders[Responder Chain]
        IC --> EventHandling[Event Handling]
    end
    
    style SI fill:#f96,stroke:#333,stroke-width:2px
    style DS,PA,UI,IC fill:#bbf,stroke:#33f,stroke-width:1px
    style ValueTypes,ReferenceTypes,Protocols,Extensions,UIKit,SwiftUI,Responders,EventHandling fill:#dfd,stroke:#393,stroke-width:1px
```

### Cursor Implementation in Swift

- Use `struct` for immutable cursor positions and selections
- Consider `class` for cursor controller with mutable state
- Use value semantics for position types (struct Point)
- Implement proper Equatable and Hashable for position types
- Consider using custom collection types for cursor sets
- Use Swift's strong type system for cursor and selection types
- Implement Foundation-compatible cursor serialization

### Selection Models in Swift

- Use Swift's ranges for selection representation
- Consider custom types for different selection modes
- Implement efficient text extraction with Swift strings
- Design clear protocols for selection behavior
- Use value semantics for selection ranges
- Consider attributed strings for selection rendering
- Implement proper Unicode handling for selection boundaries

### Multi-Cursor Implementation

- Design a clear model for cursor collections
- Use Swift arrays with proper sorting semantics
- Consider custom collection types for specialized behavior
- Implement efficient cursor merging algorithms
- Use protocol-based design for cursor behavior
- Consider using Swift's Result type for operation outcomes
- Implement custom equality for cursor comparison

### Input Handling and Event Flow

- Consider using the responder chain for input handling
- Design a clear protocol for cursor commands
- Implement efficient key binding resolution
- Use Swift's pattern matching for input processing
- Consider custom event types for specialized input
- Design clear APIs for gesture recognition
- Implement proper accessibility support

## Performance Considerations

1. **Efficient Position Tracking**: Use anchors for stable positions
2. **Batch Rendering**: Update UI efficiently with multiple cursors
3. **Lazy Calculation**: Only compute needed information
4. **Selection Caching**: Cache selection text when appropriate
5. **Cursor Merging**: Efficiently merge overlapping cursors
6. **Position Validation**: Fast checking of cursor constraints
7. **Memory Management**: Proper handling of cursor/selection lifecycle

## Interaction with Other Subsystems

### Cursor System → Buffer System
- Cursors reference buffer positions through anchors
- Edit operations are performed at cursor locations
- Selection ranges define edit targets in buffer
- See: [13_AtmosphericView_BufferAndRope.md](./13_AtmosphericView_BufferAndRope.md)

### Cursor System → Text Editor Core
- Editor view renders cursors and selections
- Input events are translated to cursor operations
- Editor commands operate on cursor state
- See: [03_StratosphericView_TextEditorCore.md](./03_StratosphericView_TextEditorCore.md)

### Cursor System → Language Intelligence
- Semantic cursor movement uses language structure
- Smart selection leverages syntax understanding
- Code actions operate at cursor positions
- See: [04_StratosphericView_LanguageIntelligence.md](./04_StratosphericView_LanguageIntelligence.md)

### Cursor System → Collaboration System
- Remote cursors show collaborator positions
- Selection sharing enables awareness
- Cursor operations are synchronized across clients
- See: [06_StratosphericView_CollaborationSystem.md](./06_StratosphericView_CollaborationSystem.md)

For a complete map of how the Cursor System connects to all other subsystems, see: [SubsystemRelationshipMap.md](./SubsystemRelationshipMap.md)

## Next Steps

After understanding the Cursor and Selection systems, we'll examine the Syntax Highlighting implementation, which provides visual differentiation of code elements based on language syntax. This includes Tree-sitter integration, tokenization, and theme-based styling.