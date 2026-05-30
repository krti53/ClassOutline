# ClassOutline — CLAUDE.md

## Project Overview
ClassOutline is a Visual Studio extension (VSIX) that provides a document/class outline panel inside the Visual Studio IDE. It shows a hierarchical tree view of the code structure of an open document, allowing quick navigation.

## Repository Structure

```
ClassOutline/
├── FireflyDocumentOutlineSolution.2017.sln      # VS 2017 solution file
├── ClassOutline.ControlLibrary/                  # WPF custom control library
│   ├── OutlineTreeviewControl.xaml               # Main treeview UI (XAML)
│   ├── OutlineTreeviewControl.xaml.cs            # Code-behind for treeview
│   ├── OutlineItem.cs                            # Outline node model / ViewModel
│   ├── ContextMenuItem.cs                        # Context menu item model
│   ├── NullImageConverter.cs                     # WPF value converter
│   ├── StringToImageConverter.cs                 # WPF value converter
│   └── packages.config                           # NuGet package references
├── ClassOutline.Logging/                         # Logging abstraction library
├── ClassOutlinePackage.2017/                     # Visual Studio Package (VSIX entry point)
├── OutlineControl.TestApp/                       # Standalone WPF test harness app
├── firefly_service.tests/                        # Automated service tests
├── Backup/                                       # Backup snapshots
├── packages/                                     # NuGet package restore cache
├── IntegrationTests.testsettings                 # Integration test settings
├── UnitTests.testsettings                        # Unit test settings
├── signfile.bat                                  # Code-signing script
└── eula.txt                                      # EULA
```

## Technology Stack
- **Language**: C#
- **Framework**: .NET Framework (WPF / XAML)
- **IDE Target**: Visual Studio 2017+
- **Project type**: VSIX (Visual Studio Extension)
- **UI**: WPF with XAML
- **Package management**: NuGet via `packages.config` (non-SDK style)

## Build Instructions

Open `FireflyDocumentOutlineSolution.2017.sln` in Visual Studio 2017 or later:

```
Build > Build Solution  (Ctrl+Shift+B)
```

NuGet packages are stored in the `packages/` directory and restored automatically. Do not switch to SDK-style `<PackageReference>` without verifying VSIX tooling compatibility.

## Key Architecture

### Projects

| Project | Purpose |
|---------|--------|
| `ClassOutlinePackage.2017` | VS Package entry point — registers tool windows, commands, and IDE event hooks |
| `ClassOutline.ControlLibrary` | Reusable WPF control library — the visible tree view panel |
| `ClassOutline.Logging` | Logging abstraction used across all projects |
| `OutlineControl.TestApp` | Standalone WPF app for manual UI testing without launching VS |
| `firefly_service.tests` | Automated tests for service-layer logic |

### Control Library Details
- `OutlineTreeviewControl` (XAML + code-behind): the primary WPF `UserControl` rendered inside the VS tool window.
- `OutlineItem`: the data model and ViewModel for each node in the tree. Contains display properties (name, icon, depth) and navigation metadata.
- Value converters (`NullImageConverter`, `StringToImageConverter`): bridge between data layer image identifiers and WPF `BitmapSource`.

## Conventions
- WPF MVVM pattern: XAML for views, minimal code-behind, business logic in model/ViewModel classes.
- Value converters live in `ClassOutline.ControlLibrary` alongside the control that uses them.
- NuGet dependencies managed via `packages.config` — add packages through Visual Studio's NuGet UI, not `dotnet add package`.
- Solution uses VS 2017-era non-SDK `.csproj` format — do not convert to SDK-style without testing the VSIX build pipeline.

## Testing

- **Unit tests**: configured via `UnitTests.testsettings` — run via Visual Studio Test Explorer.
- **Integration tests**: configured via `IntegrationTests.testsettings` — may require a running VS experimental instance.
- **Manual UI testing**: use `OutlineControl.TestApp` to verify the treeview renders correctly without deploying the extension.

## Code Signing

The `signfile.bat` script handles Authenticode signing of the output VSIX. It requires a code-signing certificate to be installed. Do not modify signing configuration without testing the full VSIX install flow.
