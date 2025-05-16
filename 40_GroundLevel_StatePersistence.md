# Ground Level: State Persistence

This document explains how Zed persists and restores state across application sessions, including settings, workspace layout, editor state, and more.

## Overview

Zed's state persistence system is responsible for saving and restoring various types of application state between sessions, ensuring a seamless user experience when reopening the application. The system handles:

1. User settings (global preferences)
2. Workspace configuration (open projects, window layouts)
3. Editor state (cursor positions, scroll positions)
4. Project-specific settings (local settings files)
5. Recently used projects
6. Terminal state

State persistence in Zed uses a combination of:

- SQLite databases (via `sqlez`) for structured data
- JSON files for settings and configuration
- File system paths for workspace history

## Core Persistence Components

### Database System (`/crates/db`)

The central persistence mechanism is based on SQLite, implemented through a custom wrapper:

```rust
pub async fn open_db<M: Migrator + 'static>(db_dir: &Path, scope: &str) -> ThreadSafeConnection {
    if *ZED_STATELESS {
        return open_fallback_db::<M>().await;
    }

    let main_db_dir = db_dir.join(format!("0-{}", scope));

    let connection = maybe!(async {
        smol::fs::create_dir_all(&main_db_dir)
            .await
            .context("Could not create db directory")
            .log_err()?;
        let db_path = main_db_dir.join(Path::new(DB_FILE_NAME));
        open_main_db::<M>(&db_path).await
    })
    .await;

    if let Some(connection) = connection {
        return connection;
    }

    // Set another static ref so that we can escalate the notification
    ALL_FILE_DB_FAILED.store(true, Ordering::Release);

    // If still failed, create an in memory db with a known name
    open_fallback_db::<M>().await
}
```

Key characteristics:
- SQLite with WAL (Write-Ahead Logging) for performance
- Fall-back to in-memory database if file persistence fails
- Managed through a thread-safe connection wrapper
- Support for migrations via specific domain types
- Configurable to be stateless (using `ZED_STATELESS` environment variable)

### Settings Store (`/crates/settings`)

The `SettingsStore` manages user preferences and application configuration:

```rust
pub struct SettingsStore {
    setting_values: HashMap<TypeId, Box<dyn AnySettingValue>>,
    raw_default_settings: Value,
    raw_user_settings: Value,
    raw_server_settings: Option<Value>,
    raw_extension_settings: Value,
    raw_local_settings: BTreeMap<(WorktreeId, Arc<Path>), Value>,
    raw_editorconfig_settings: BTreeMap<(WorktreeId, Arc<Path>), (String, Option<Editorconfig>)>,
    tab_size_callback: Option<(
        TypeId,
        Box<dyn Fn(&dyn Any) -> Option<usize> + Send + Sync + 'static>,
    )>,
    _setting_file_updates: Task<()>,
    setting_file_updates_tx:
        mpsc::UnboundedSender<Box<dyn FnOnce(AsyncApp) -> LocalBoxFuture<'static, Result<()>>>>,
}
```

Settings are organized in a layered structure:
1. Default settings (bundled with application)
2. User settings (global preferences)
3. Project-specific settings (local overrides)
4. Extension-provided settings
5. Release channel specific settings

The settings system uses a trait-based approach to handle different types of settings:

```rust
pub trait Settings: 'static + Send + Sync {
    const KEY: Option<&'static str>;
    const FALLBACK_KEY: Option<&'static str> = None;
    const PRESERVED_KEYS: Option<&'static [&'static str]> = None;
    type FileContent: Clone + Default + Serialize + DeserializeOwned + JsonSchema;
    fn load(sources: SettingsSources<Self::FileContent>, cx: &mut App) -> Result<Self> where Self: Sized;
    fn json_schema(generator: &mut SchemaGenerator, _: &SettingsJsonSchemaParams, _: &App) -> RootSchema;
    fn missing_default() -> anyhow::Error;
    fn import_from_vscode(vscode: &VsCodeSettings, current: &mut Self::FileContent);
}
```

### Workspace Persistence (`/crates/workspace/src/persistence`)

Workspace persistence manages saving and restoring the editor layout, open files, and project structure:

```rust
pub(crate) struct SerializedWorkspace {
    pub(crate) id: WorkspaceId,
    pub(crate) location: SerializedWorkspaceLocation,
    pub(crate) center_group: SerializedPaneGroup,
    pub(crate) window_bounds: Option<SerializedWindowBounds>,
    pub(crate) centered_layout: bool,
    pub(crate) display: Option<Uuid>,
    pub(crate) docks: DockStructure,
    pub(crate) session_id: Option<String>,
    pub(crate) breakpoints: BTreeMap<Arc<Path>, Vec<SourceBreakpoint>>,
    pub(crate) window_id: Option<u64>,
}
```

The workspace state captures:
- Window layout (size, position, maximized state)
- Pane structure (split panes, tabs)
- Open files and their positions within panes
- Active pane and item selection
- Dock visibility and state (left, right, bottom panels)
- Debug breakpoints

### Recent Projects Tracking (`/crates/recent_projects`)

The recent projects system tracks previously opened workspaces for quick access:

```rust
// Load recent workspaces from database
let workspaces = WORKSPACE_DB
    .recent_workspaces_on_disk()
    .await
    .log_err()
    .unwrap_or_default();
```

Recent projects track:
- Local file paths with ordering preserved
- SSH remote projects with host, user, port, and paths
- Last accessed timestamps
- Project type (local vs remote)

### Terminal State Persistence (`/crates/terminal_view/src/persistence.rs`)

Terminal state is persisted for terminal sessions, capturing:
- Command history
- Working directory
- Terminal buffer state

## Persistence File Locations

State is persisted in the following locations:

- **Application data directory**: Platform-specific location for databases and settings
  - macOS: `~/Library/Application Support/dev.zed.Zed/`
  - Linux: `~/.config/Zed/`
  - Windows: `%APPDATA%\Zed\`

- **Settings**:
  - Global settings: `settings.json` in the application data directory
  - Project settings: `.zed/settings.json` in the project root directory
  - Tasks: `.zed/tasks.json` in the project root directory
  - Editorconfig: `.editorconfig` in any project directory

- **Database**:
  - Main database: `db.sqlite` in the application data directory

## Persistence Flow

### Saving State

1. **Settings**:
   - Settings are saved through the `SettingsStore` using atomic file writes
   - Changes are queued via `setting_file_updates_tx` channel and processed asynchronously

```rust
pub fn update_settings_file<T: Settings>(
    &self,
    fs: Arc<dyn Fs>,
    update: impl 'static + Send + FnOnce(&mut T::FileContent, &App),
) {
    self.setting_file_updates_tx
        .unbounded_send(Box::new(move |cx: AsyncApp| {
            async move {
                let old_text = Self::load_settings(&fs).await?;
                let new_text = cx.read_global(|store: &SettingsStore, cx| {
                    store.new_text_for_update::<T>(old_text, |content| update(content, cx))
                })?;
                let settings_path = paths::settings_file().as_path();
                if fs.is_file(settings_path).await {
                    let resolved_path =
                        fs.canonicalize(settings_path).await.with_context(|| {
                            format!("Failed to canonicalize settings path {:?}", settings_path)
                        })?;

                    fs.atomic_write(resolved_path.clone(), new_text)
                        .await
                        .with_context(|| {
                            format!("Failed to write settings to file {:?}", resolved_path)
                        })?;
                } else {
                    fs.atomic_write(settings_path.to_path_buf(), new_text)
                        .await
                        .with_context(|| {
                            format!("Failed to write settings to file {:?}", settings_path)
                        })?;
                }

                anyhow::Ok(())
            }
            .boxed_local()
        }))
        .ok();
}
```

2. **Workspace state**:
   - Persisted to database when workspace is closed or periodically saved
   - Serializes the entire workspace structure including panes, editors, and dock state
   - Workspace locations (paths/SSH) are hashed to generate stable IDs for projects

3. **Recent projects**:
   - Updated whenever a project is opened or closed
   - Records are added to the database with timestamps for sorting

### Restoring State

1. **Application startup**:
   - Default settings loaded from bundled files
   - User settings loaded from settings.json
   - Extension settings loaded from installed extensions

2. **Workspace reopening**:
   - Workspace layout reconstructed from serialized state
   - Panes and editors recreated with saved cursor positions
   - Dock visibility and panel selection restored

```rust
pub(crate) async fn deserialize(
    self,
    project: &Entity<Project>,
    workspace_id: WorkspaceId,
    workspace: WeakEntity<Workspace>,
    cx: &mut AsyncWindowContext,
) -> Option<(
    Member,
    Option<Entity<Pane>>,
    Vec<Option<Box<dyn ItemHandle>>>,
)> {
    // Reconstruction logic for workspace structure
    // ...
}
```

3. **Recent projects list**:
   - Query database for recently accessed workspaces
   - Sort by access time
   - Display in UI for quick access

## Special Persistence Features

### Editor Buffer State

The editor persists detailed state information:
- Cursor positions and selections
- Scroll position
- Undo/redo history
- Unsaved changes (optional)

### Terminal Sessions

Terminal sessions can persist across application restarts:
- Working directory
- Command history
- Buffer content (scrollback)
- Running processes (can be restarted)

### Local vs Global Settings

Settings follow a cascading override system:
1. Default settings provide baseline values
2. User settings override defaults globally
3. Project settings override both within a project
4. Release-channel specific settings (dev, nightly, stable)

```rust
fn recompute_values(
    &mut self,
    changed_local_path: Option<(WorktreeId, &Path)>,
    cx: &mut App,
) -> std::result::Result<(), InvalidSettingsError> {
    // Logic to merge settings from different sources with proper priority
    // ...
}
```

### Editor Bookmarks and Navigation History

Zed maintains navigation history for each editor:
- Jump list (forward/backward navigation)
- Bookmarks within files
- Recent file access history

## Swift Implementation Considerations

When reimplementing Zed's state persistence system in Swift, consider:

1. **Database Layer**:
   - Use GRDB or SQLite.swift instead of custom SQLite wrapper
   - Implement migration system using Codable conformance
   - Consider CoreData for object graph management

2. **Settings System**:
   - Use Swift's type safety through Codable protocol
   - Consider property wrappers for settings access
   - Use UserDefaults for simple settings, files for complex ones

3. **File Handling**:
   - Use FileManager for path operations
   - Consider atomic file operations through temporary files
   - Use NSFileCoordinator for shared file access

4. **State Serialization**:
   - Use Codable for serializing/deserializing state
   - Consider using property lists for settings (more native to Apple platforms)
   - Use NSSecureCoding for more complex object graphs

5. **Error Handling**:
   - Implement robust error recovery for data corruption
   - Use Result type for error propagation
   - Consider CloudKit for sync capabilities

6. **UI State Restoration**:
   - Use SwiftUI's @SceneStorage and @AppStorage for simple state
   - Implement custom restoration for complex UI hierarchies
   - Consider scene-based state management

## Conclusion

Zed's state persistence system provides a robust framework for saving and restoring application state across sessions. The combination of SQLite databases and JSON files offers both structured data storage and human-readable configuration. The multi-layered settings system allows for flexible overrides at different levels, from global preferences to project-specific settings.

When reimplementing in Swift, the core concepts can be preserved while leveraging Swift's type safety and Apple's platform capabilities for persistence and state restoration.