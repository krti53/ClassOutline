# ClassOutline

A Visual Studio 2017 extension (VSIX) that displays a live class hierarchy tree of the currently active C# document in a dockable tool window.

## Project Purpose

ClassOutline adds a **Class Outline** tool window to Visual Studio that shows classes, methods, constructors, and `#region` directives for the active document. Double-clicking a node navigates to that code location. It integrates with Firefly's UIController pattern to provide special icons for UI controller classes.

## Solution Structure

```
FireflyDocumentOutlineSolution.2017.sln
│
├── ClassOutline.ControlLibrary/          # Reusable WPF control library
│   ├── OutlineItem.cs                    # Core tree-node data model (INotifyPropertyChanged)
│   ├── OutlineTreeviewControl.xaml(.cs)  # Custom TreeView control
│   ├── ContextMenuItem.cs
│   ├── NullImageConverter.cs
│   └── StringToImageConverter.cs
│
├── ClassOutline.Logging/                 # Visual Studio Output window logger
│   └── VisualStudioOutputLogger.cs
│
├── ClassOutlinePackage.2017/             # Main VSIX package project
│   ├── VSPackage1Package.cs              # AsyncPackage entry point; registers services & tool window
│   ├── ClassOutlineControl.xaml(.cs)     # Main tool window UserControl (all core UI logic lives here)
│   ├── ClassOutlineToolWindow.cs         # ToolWindowPane wrapper
│   ├── Guids.cs                          # Package & command set GUIDs
│   ├── PkgCmdID.cs                       # Command IDs
│   ├── VSPackage1.vsct                   # Command table definition
│   ├── source.extension.vsixmanifest     # VSIX metadata
│   ├── Entities/
│   │   └── CodeRegion.cs
│   ├── Services/
│   │   ├── CodeElementHelper.cs          # DTE cursor → CodeElement lookup
│   │   ├── RegionParser.cs               # Parses #region directives from document text
│   │   ├── ViewParser.cs                 # Locates MVC/MVVM view files for a class
│   │   ├── ImageCache.cs                 # Caches icons from the AppData folder
│   │   ├── ItemKindImageMapService.cs    # Maps vsCMElement kinds → icon URIs
│   │   └── BitmapToBitmapSource.cs
│   ├── TreeNodes/
│   │   ├── TreeNodeBase.cs               # Base class: wraps CodeElement, exposes Name/Kind
│   │   ├── ClassTreeNode.cs              # Resolves base-class names
│   │   ├── MethodTreeNode.cs
│   │   ├── PropertyTreeNode.cs
│   │   ├── VariableTreeNode.cs
│   │   ├── EnumTreeNode.cs / EventTreeNode.cs / StructTreeNode.cs
│   │   └── GenericTreeNode.cs
│   ├── Extensions/                       # C# extension methods
│   ├── UI/                               # Additional UI helpers
│   ├── Images/                           # Embedded PNG icons
│   ├── VSPackage1_UnitTests/             # MSTest unit tests
│   └── VSPackage1_IntegrationTests/      # VS-hosted integration tests
│
├── OutlineControl.TestApp/               # Standalone WPF harness for testing the control
├── firefly_service.tests/                # NUnit tests for service classes
└── packages/                             # NuGet packages (classic packages.config format)
```

## Technology Stack

- **Language**: C# (.NET Framework, Visual Studio SDK target)
- **UI**: WPF (XAML + code-behind)
- **VS Extensibility**: VSPackage / AsyncPackage, DTE / DTE2 COM automation API
- **Logging**: log4net (configured via `ClassOutlinePackage.2017/logging.config`)
- **Package management**: NuGet via `packages.config` (classic format — do not mix with PackageReference)
- **Testing**: MSTest (unit + VS-hosted integration), NUnit (`firefly_service.tests`)

## Architecture & Key Patterns

### Package Entry Point (VSPackage1Package.cs)
- Extends `AsyncPackage` with `AllowsBackgroundLoading = true`
- Registers `IClassOutlineSettingsProvider` as an async VS service via `AddService`
- Exposes `OptionPageGrid` for the **Firefly Community > Class Outline** options page
  - Single option: `FireflyImagesEnabled` — toggles special UIController icons
- Handles unhandled exceptions via `AppDomain.CurrentDomain.UnhandledException` → logs with log4net as Fatal

### DTE Initialization — Zombie Mode Handling (ClassOutlineControl.cs → DteInitializer)
Visual Studio enters "zombie mode" on startup. All DTE-dependent initialization is deferred via the inner `DteInitializer : IVsShellPropertyEvents` class, which monitors `VSSPROPID_Zombie` and runs the setup callback only after VS is fully initialized. Never access `DTE`/`DTE2` before this callback fires.

### Event-Driven Tree Refresh
The control subscribes to four DTE2 event sources (all stored as `Lazy<T>` to avoid premature initialization):

| Event | Handler |
|-------|---------|
| `SelectionEvents.OnChange` | `OutlineCode()` immediately |
| `DocumentEvents.DocumentSaved/DocumentOpened` | `refreshToolWindows()` |
| `WindowEvents.WindowActivated` | `refreshToolWindows()` (Document-kind only, deduped by caption) |
| `SolutionEvents.AfterClosing` | clears the outline |

All handlers must be detached in `detachDTEEventHandlers()` on VS shutdown to prevent leaks.

### Cursor Sync Timer
A `System.Windows.Forms.Timer` fires every **1000 ms** (`CodeSyncTimerInteral`) calling `selectActiveCodeInTree()`, which uses `CodeElementHelper.GetCodeElementAtCursor(_dte)` to find the element under the cursor and selects the matching `OutlineItem` in the tree via `tvOutline.Select(itm)`.

### Tree Data Model (OutlineItem in ClassOutline.ControlLibrary)
`OutlineItem` implements `INotifyPropertyChanged` and is the view-model for each tree node:
- `Children` — nested `OutlineItem` nodes (classes contain member nodes)
- `Methods` — sorted list of **prioritized** methods shown as quick-access shortcuts
- Regions — `#region` blocks parsed from raw document text and overlaid on the tree
- Event handlers: `GotoCodeLocationEventHandler`, `OpenProjectItemEventHandler`, `UpdateViewsEventHandler`

### Method Priority — getMethodPriority()
Only methods meeting a priority threshold appear as shortcuts in the context menu:

| Method | Priority |
|--------|----------|
| `Run()` | 100 |
| Constructor | 99 |
| `Initialize*` | 90 |
| `override` methods | 80 |
| Everything else | 0 (excluded) |

### UIController Special Icon
When `FireflyImagesEnabled = true` and a class's base-class hierarchy contains a name with `"UIController"`, the class node icon switches from `/Resources/Classes.png` to `/Resources/UIController.png`.

### Logging Convention
```csharp
private ILog _log = LogManager.GetLogger(typeof(ClassName));

_log.Debug("entering method");
_log.Error("description of failure", exception);
_log.Fatal("unrecoverable: " + errorMessage);
```
Configure appenders and levels in `ClassOutlinePackage.2017/logging.config`.

### Namespace Convention
- VSIX package code: `ClassOutline`
- Control library: `ClassOutline.ControlLibrary`
- Logging: `ClassOutline.Logging`

## Build & Development

### Prerequisites
- Visual Studio 2017 or later with the **Visual Studio extension development** workload
- Visual Studio SDK installed

### Opening the Solution
```
FireflyDocumentOutlineSolution.2017.sln
```

### Running in Debug
Press **F5** — this launches an **Experimental Instance** of Visual Studio with the extension loaded. The main VS instance is not affected.

### Release Build & Output
Build in Release configuration. Output is a `.vsix` file under `ClassOutlinePackage.2017/bin/Release/`.

### Signing
```bat
signfile.bat
```
Run after Release build to sign the `.vsix` for marketplace distribution.

### Running Tests
- **Unit tests**: Open Test Explorer → Run All (`VSPackage1_UnitTests`, `firefly_service.tests`)
- **Integration tests**: Use `IntegrationTests.testsettings` — requires VS experimental instance
- NUnit tests in `firefly_service.tests` run with any NUnit 3 runner

## Adding New Features

### New tree node type
1. Create a class in `TreeNodes/` extending `TreeNodeBase`
2. Handle the corresponding `vsCMElement` kind in `ClassOutlineControl.createClassList()`
3. Add a PNG icon to `ClassOutlinePackage.2017/Images/`
4. Register the icon mapping in `ItemKindImageMapService`

### New DTE event handler
1. Declare a `Lazy<XxxEvents>` field in `ClassOutlineControl`
2. Initialize it in `addDTEEventHandlers()` from `_eventRoot.Value`
3. Detach in `detachDTEEventHandlers()` — forgetting this causes a memory/event leak on VS shutdown

### New settings option
1. Add a property to `OptionPageGrid` with `[Category]`, `[DisplayName]`, `[Description]` attributes
2. Expose it via `IClassOutlineSettingsProvider`
3. Consume through `_settingService.Value` in `ClassOutlineControl`

## Important Notes & Gotchas

- **C# only**: `FileCodeModel` only works reliably for C# files. Non-C# documents fall through silently — this is by design.
- **COM threading**: All DTE calls must be made on the UI thread. In async contexts use `await ThreadHelper.JoinableTaskFactory.SwitchToMainThreadAsync()` before any DTE access.
- **Image cache location**: Custom member-type icons are loaded from `%APPDATA%\ClassOutlineImages`. Missing folder is handled gracefully by `ImageCache`.
- **Dead code in getSelectedProjectItem()**: There is unreachable code after the first `return` statement (the UIHierarchy path). Do not rely on or extend that branch.
- **packages.config format**: This project uses classic `packages.config`. Do not upgrade to `PackageReference` without testing the full build pipeline.
- **Backup/ folder**: Contains older project snapshots. Ignore for development — not part of the active solution.
