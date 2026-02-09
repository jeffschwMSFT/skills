# WinForms Expert Agent

A custom VS Code agent for building, debugging, and optimizing Windows Forms applications. Targets .NET 10+ for new projects while supporting legacy .NET Framework codebases.

## Capabilities

- **Designer-safe code generation** — Strict separation of designer code-behind (C# 2.0–level serialization rules) from modern C# in regular code files. Enforces `InitializeComponent` structure, prohibited patterns, and backing field placement.
- **Layout & UI design** — `TableLayoutPanel`/`FlowLayoutPanel`-first layouts with proper sizing strategies (AutoSize > Percent > Absolute), DPI awareness, DarkMode support (.NET 9+), and accessibility (`AccessibleName`, tab order, mnemonics).
- **Data binding & MVVM (.NET 8+)** — Object DataSources, `BindingSource` as mediator, MVVM CommunityToolkit ViewModels, `Control.DataContext`, `ButtonBase.Command`/`ToolStripItem.Command` bindings, and `IValueConverter` workarounds via `Binding.Format`/`Parse`.
- **Async patterns (.NET 9+)** — `Control.InvokeAsync` overload selection, `ShowAsync`/`ShowDialogAsync`, safe `async void` event handlers with mandatory `try/catch`, and `ExceptionDispatchInfo` for stack-preserving rethrows.
- **Custom controls** — CodeDOM serialization management (`[DefaultValue]`, `[DesignerSerializationVisibility]`, `ShouldSerialize`/`Reset` patterns), owner-draw, and GDI+ resource lifecycle.
- **VB.NET support** — Application Framework conventions, `Friend WithEvents` backing fields, `Handles` clause preference, and `ApplicationEvents.vb` for app-wide defaults.
- **NuGet & project setup** — Prefers stable packages with version ranges (e.g., `[2.*,)`), Windows API projections targeting `10.0.22000.0`, and proper `HighDpiMode` configuration via code (not app.config).

## Usage

Select **WinForms Expert** from the agents dropdown in Copilot Chat, then describe your WinForms task or question.

## Tools

Full access — the agent can search, read, edit files, run terminal commands, manage NuGet packages, and use web resources.
