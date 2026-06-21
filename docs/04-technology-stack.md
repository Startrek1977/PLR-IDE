# 04 — Technology Stack

Every choice below is justified by a one-line **why**, and where a decision is
still open it points to [16 — Discovery questions](16-discovery-questions.md).
Versions reflect the current LTS as of this writing; pin exact versions at
solution-creation time.

---

## 1. Platform & language

| Concern | Choice | Why |
|---|---|---|
| Runtime | **.NET 10 (LTS)** | Current LTS; long support window; best perf; full WPF support |
| Language | **C# 14** | `field` keyword, partial properties/events — cleaner MVVM |
| App model | **WPF** (primary) + **Avalonia** (optional) | See [05](05-ui-platform-options.md) |
| Native bridge | **C++/CLI** (`net10.0-windows`) or **P/Invoke** | Wrap proven drivers; see [07](07-data-acquisition-interop.md) |

> .NET Framework patterns are **out**: no `packages.config`, no `App.config` DI,
> no standalone `AssemblyInfo.cs`, no `ServiceLocator`. SDK-style projects only.

---

## 2. Application framework

| Concern | Choice | Why |
|---|---|---|
| MVVM | **CommunityToolkit.Mvvm 8.4** | Source-generated `ObservableProperty`/`RelayCommand`; tiny; no runtime reflection |
| Hosting / DI | **Microsoft.Extensions.Hosting** (Generic Host) | Standard composition root, lifetime, config, logging |
| Navigation | Custom `INavigationService` over the Host | In-app page/module navigation (see `dotnet-wpf-modern`) |
| Async commands | `IAsyncRelayCommand` | Non-blocking UI for long ops (dump/export) |
| Theming | `DynamicResource` + `IThemeService` | Runtime dark/light; see [skill:wpf-theming] and [14](14-cross-cutting-concerns.md) |
| Window chrome | `WindowChrome` (WPF) | Modern title bar **without** `WindowStyle=None` |
| Logging | **Microsoft.Extensions.Logging** (+ optional **Serilog**) | Structured logs, file/console sinks; offline-friendly for defense |
| Config/settings | `IOptions<T>` + JSON store | Replaces `Properties.Settings`; window placement, recent files |

---

## 3. Real-time ingestion & performance

| Concern | Choice | Why |
|---|---|---|
| Producer/consumer | **System.Threading.Channels** (bounded) | Lock-free-ish, back-pressure, async — ideal acquisition→pipeline seam |
| Heavy transforms | **TPL Dataflow** (where graph-shaped) | Parallel decommutation/calibration blocks when needed |
| Zero-copy parsing | **`Span<T>` / `Memory<T>` / `ReadOnlySequence<T>`** | Decommutate frames with no allocations |
| Buffer reuse | **`ArrayPool<byte>` / `MemoryPool<T>`** | Avoid GC pressure on the hot path |
| Numeric/bit work | `System.Buffers.Binary`, `BitOperations`, `MemoryMarshal` | Fast, correct endian/bit extraction |
| Optional SIMD | `System.Numerics.Vector<T>` | Vectorized calibration over sample blocks |
| Optional AOT/hot path | unsafe/blittable structs; consider [skill:dotnet-native-aot] for tools | Tight, predictable hot loops |

See [07](07-data-acquisition-interop.md) for how these fit the pipeline.

---

## 4. Visualization (commercial high-performance)

| Concern | Choice | Why |
|---|---|---|
| Primary chart engine | **SciChart WPF** *or* **LightningChart .NET** | GPU/DirectX; millions of real-time points; aerospace-proven |
| Selection method | Perf spike against representative rates, then pick | Avoid lock-in before data; see [06](06-visualization-layer.md) |
| Tables / messages | Virtualized `DataGrid` / `ItemsRepeater` | Thousands of rows without UI stalls |
| Gauges / bars / stacks | Vendor gauge controls or lightweight custom | Match legacy graph-unit types |
| Image export | Chart engine bitmap export + clipboard | Reproduce "copy pictures" |

Full mapping of each legacy graph-unit type and the real-time feeding strategy is
in [06 — Visualization layer](06-visualization-layer.md).

---

## 5. Domain engine

| Concern | Choice | Why |
|---|---|---|
| Expression engine | **DynamicExpresso** / **Jace.NET** / **Flee**, or a custom compiler | Compile compound-parameter formulas **once** to `Func<>`; evaluate at rate |
| Calibration | Strategy interfaces (`ICalibration`) + compiled polynomials | Pluggable, testable, fast |
| Setup persistence | **SQLite** (EF Core or `Microsoft.Data.Sqlite`) or structured files (JSON/Protobuf) | Robust new format; legacy importers separate |
| Validation | `ObservableValidator` / DataAnnotations | Setup-form validation in the UI |

Decision note: a **custom compile-to-delegate** engine gives the best control and
performance for hot evaluation; a library is faster to start with. Start with a
library behind an `IExpressionCompiler` interface, swap later if needed. See
[08](08-core-engine.md).

---

## 6. Recording, playback & export

| Concern | Choice | Why |
|---|---|---|
| Recording I/O | **Memory-mapped files** / async sequential writes | High sustained throughput; indexable |
| Container/format | New self-describing format + index; **legacy readers** | Backward compatibility is mandatory |
| Excel export | **ClosedXML** or **OpenXML SDK** | Native `.xlsx`, no Excel install required |
| MATLAB export | `.mat` writer (e.g. **MatFileHandler**) or CSV fallback | Reproduce MATLAB upload |
| ASCII/binary dump | Custom writers (parity-tested) | Bit/numerically identical to legacy |

See [09](09-recording-and-playback.md) and [11](11-module-implementation-guide.md).

---

## 7. Quality, testing & delivery

| Concern | Choice | Why |
|---|---|---|
| Unit tests | **xUnit** + FluentAssertions | Standard, expressive |
| Parity/golden-file | Custom harness over `IDE.Pipeline`/`IDE.Export` | Prove bit-exact vs legacy |
| Property/perf | BenchmarkDotNet (micro), throughput harness (macro) | Guard the hot path in CI |
| UI tests | **WinAppDriver/FlaUI** (WPF), **Avalonia.Headless** | Smoke + key flows; see [skill:dotnet-ui-testing-core] |
| Packaging | **MSIX** (per-machine) or self-contained `dotnet publish` | Modern install/update; offline deployable |
| CI | GitHub Actions / Azure DevOps | Build, test, parity & perf gates |
| High-DPI | Per-Monitor V2 manifest | Crisp on engineering multi-monitor rigs |

---

## 8. Package summary (indicative)

```xml
<!-- IDE.App.Wpf.csproj (sketch) -->
<PackageReference Include="Microsoft.Extensions.Hosting" Version="10.0.*" />
<PackageReference Include="CommunityToolkit.Mvvm" Version="8.4.*" />
<PackageReference Include="Serilog.Extensions.Hosting" Version="8.*" />        <!-- optional -->
<!-- Charting: exactly one, by license -->
<!-- <PackageReference Include="SciChart" Version="..." /> -->
<!-- <PackageReference Include="LightningChart..." Version="..." /> -->
<!-- Engine/export -->
<PackageReference Include="DynamicExpresso.Core" Version="..." />              <!-- or Jace/Flee -->
<PackageReference Include="ClosedXML" Version="..." />
<PackageReference Include="Microsoft.Data.Sqlite" Version="10.0.*" />
```

> **Open decisions** (→ [16](16-discovery-questions.md)): charting vendor;
> expression library vs custom; SQLite vs structured-file setups; MATLAB writer
> choice; whether any cloud/AI packages are permitted under export control.

---

### Next
→ [05 — UI platform options](05-ui-platform-options.md)
