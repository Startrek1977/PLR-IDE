# CLAUDE.md

Guidance for Claude Code (and humans) working in this repository.

## What this repo is

A **modernization proposal & architecture set** for converting PLR Systems'
**IDE (Integrated Debriefing Environment)** — a Windows **MFC/C++** flight-test
telemetry application — into a modern **.NET 10** application that **preserves
functionality and efficiency**.

**Current state:** documentation only (`README.md` + `docs/01..16` + this file).
There is **no application code yet** and the MFC source is **not** in this repo.
Over time this repo is intended to grow into the actual .NET solution.

Start at [README.md](README.md); the doc index lists all 16 documents.

## Locked decisions (do not silently change — see README)

- **Primary UI:** WPF on **.NET 10 (LTS)**, **C# 14**.
- **Alternative UI:** Avalonia (cross-platform), sharing the same core/ViewModels.
- **Visualization:** commercial GPU charting (**SciChart** / **LightningChart**),
  chosen after a perf spike; accessed behind an `IChartHost` adapter.
- **Migration:** incremental **"wrap-the-core"** — preserve & wrap the native
  acquisition/driver layer; rewrite domain/engine/recording/export/UI in .NET;
  deliver module-by-module (Data Dump → Setup → Debriefing).
- **App framework:** CommunityToolkit.Mvvm 8.4 · Microsoft.Extensions Host/DI ·
  `DynamicResource` theming · Serilog/MS.Extensions.Logging.

Rationale lives in [docs/02](docs/02-modernization-strategy.md),
[docs/04](docs/04-technology-stack.md), [docs/05](docs/05-ui-platform-options.md).

## Planned solution layout (when code starts)

See [docs/03 §2](docs/03-target-architecture.md#2-projects--solution-layout). Summary:

```
src/ IDE.Core · IDE.Pipeline · IDE.Recording · IDE.Export
     IDE.Acquisition.Abstractions · .Native (C++/CLI) · .Interop (P/Invoke) · .Replay
     IDE.App.Core · IDE.App.Wpf · IDE.App.Avalonia · IDE.Intelligence (optional)
tests/ IDE.*.Tests · IDE.App.UITests
```

Naming convention: `{RootNamespace}.{PurposeSuffix}` with root **`IDE`**. Only
`IDE.Acquisition.Native` and `IDE.App.Wpf` are Windows-pinned
(`net10.0-windows`); keep everything else platform-neutral so Avalonia stays
viable.

## Conventions for when implementation begins

Follow the [skill:dotnet-wpf-modern] patterns and these rules:

- **MVVM, not code-behind.** Logic in ViewModels/services; XAML for layout. Use
  source-generated `[ObservableProperty]` / `[RelayCommand]`; `IAsyncRelayCommand`
  for long ops.
- **DI everywhere.** Generic Host composition root; constructor injection; no
  globals/service-locator. Each module exposes an `AddIdeXxx()` extension.
- **Keep `IDE.Core` UI-free.** No WPF/Avalonia types below `IDE.App.*`.
- **Hot-path discipline.** Acquisition decoupled from rendering; bounded
  `Channel<T>`; `Span<T>`/`ArrayPool<T>`; struct frames; no per-sample allocations
  (see [docs/07](docs/07-data-acquisition-interop.md)).
- **Theming.** `DynamicResource` brushes only — no hardcoded colors (see
  [skill:wpf-theming]). Use `WindowChrome`, never `WindowStyle=None`.
- **Trimming.** `TrimMode=partial` only (WPF uses XAML reflection).
- **Parity is sacred.** Decommutation/calibration/export must be byte/numerically
  identical to legacy — guard with golden-file tests.

## Hard constraints (keep in mind for any change)

- **Real-time/determinism** on the acquisition path (no GC stalls).
- **Backward compatibility** with legacy setup & recording files (readers/importers).
- **Offline/air-gapped** capable; **export-control/ITAR** sensitivity — default to
  **local** AI, no mandatory cloud (see [docs/14 §7](docs/14-cross-cutting-concerns.md)).
- **Confirm the gating unknowns** before architecture-affecting work — see
  [docs/16](docs/16-discovery-questions.md) (driver/SDK assets, data rates, legacy
  formats, cross-platform need, AI/ITAR posture).

## Build / run

- No solution yet. When created: standard `dotnet build` / `dotnet test`; WPF app
  is `IDE.App.Wpf` (`net10.0-windows`). Update this section when the `.sln`/`.slnx`
  lands.
- Docs are plain Markdown with **Mermaid** diagrams (GitHub renders them).

## How to extend the docs

- Keep one topic per file under `docs/`; preserve the `NN-title.md` numbering and
  the cross-links (each doc ends with a "Next" link; README has the master index).
- When a [discovery question](docs/16-discovery-questions.md) is answered, record
  the decision in the relevant doc and remove the corresponding assumption.
- Reference related skills where useful: [skill:dotnet-wpf-modern],
  [skill:wpf-theming], [skill:wpf-mvvm], [skill:dotnet-ui-testing-core],
  [skill:dotnet-wpf-migration], [skill:dotnet-native-aot].

## Repo etiquette

- This is on the `master` branch; `main` is the usual PR base. Branch before
  committing; commit/push only when asked.
- These documents may be **IP-sensitive** — confirm repo visibility before making
  anything public ([docs/16 Q46](docs/16-discovery-questions.md)).
