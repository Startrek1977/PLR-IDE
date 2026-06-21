# 10 — Feature Conversion Examples (MFC → modern .NET)

Concrete **before/after** patterns showing how typical MFC/C++ constructs become
modern WPF + MVVM on .NET 10. These are templates the team applies repeatedly.
Patterns follow the [skill:dotnet-wpf-modern] conventions (CommunityToolkit.Mvvm
8.4, Host-builder DI, `IAsyncRelayCommand`, `DynamicResource`).

> Code is illustrative (the legacy source isn't in hand yet); the *shape* of each
> conversion is what matters.

---

## 1. `CDialog` setup form → View + ViewModel

**Before (MFC):** dialog class with `DoDataExchange`, `DDX_Text`, message-map
handlers, manual `UpdateData(TRUE/FALSE)`.

```cpp
// ParamDlg.cpp (MFC)
void CParamDlg::DoDataExchange(CDataExchange* pDX) {
    DDX_Text(pDX, IDC_NAME, m_name);
    DDX_Text(pDX, IDC_SCALE, m_scale);
}
void CParamDlg::OnBnClickedOk() {
    UpdateData(TRUE);
    if (m_scale <= 0) { AfxMessageBox(_T("Scale must be > 0")); return; }
    // ... write back to model ...
    CDialog::OnOK();
}
```

**After (WPF + MVVM):** declarative binding, source-generated properties,
validation, async command.

```csharp
// ParameterEditViewModel.cs (IDE.App.Core)
public sealed partial class ParameterEditViewModel(ISetupRepository repo) : ObservableValidator
{
    [ObservableProperty] private string _name = "";

    [ObservableProperty]
    [NotifyDataErrorInfo]
    [Range(double.Epsilon, double.MaxValue, ErrorMessage = "Scale must be > 0")]
    private double _scale = 1.0;

    [RelayCommand(CanExecute = nameof(CanSave))]
    private async Task SaveAsync(CancellationToken ct)
    {
        ValidateAllProperties();
        if (HasErrors) return;
        await repo.SaveParameterAsync(ToModel(), ct);
    }
    private bool CanSave() => !HasErrors;
}
```

```xml
<!-- ParameterEditView.xaml (IDE.App.Wpf) -->
<StackPanel>
  <TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}"/>
  <TextBox Text="{Binding Scale, UpdateSourceTrigger=PropertyChanged,
                          ValidatesOnNotifyDataErrors=True}"/>
  <Button Content="Save" Command="{Binding SaveCommand}"/>
</StackPanel>
```

---

## 2. SDI/MDI Doc/View → services + navigation

**Before (MFC):** `CWinApp` + `CDocument`/`CView`, `CDocTemplate`, global state,
`AfxGetApp()`.

**After:** Generic Host composition root; state in injected services;
`INavigationService` switches modules.

```csharp
// App.xaml.cs (IDE.App.Wpf) — see dotnet-wpf-modern
public partial class App : Application
{
    private IHost _host = null!;
    protected override async void OnStartup(StartupEventArgs e)
    {
        _host = Host.CreateApplicationBuilder()
            .AddIdeCore().AddIdePipeline().AddIdeRecording().AddIdeExport()
            .AddAcquisition(/* config: Native|Interop|Replay */)
            .AddWpfPresentation()    // ViewModels + views + IChartHost
            .Build();
        await _host.StartAsync();
        _host.Services.GetRequiredService<MainWindow>().Show();
    }
    protected override async void OnExit(ExitEventArgs e) => await _host.StopAsync();
}
```

```csharp
// Navigation between Setup / Debriefing / Data Dump
navigation.NavigateTo<DebriefingViewModel>();
```

---

## 3. Custom `CView`/`OnDraw` plot → GPU chart

**Before (MFC):** manual GDI in `OnDraw` — `MoveTo/LineTo`, `CDC`, flicker,
hand-rolled scaling; collapses at high point counts.

```cpp
void CPlotView::OnDraw(CDC* pDC) {
    pDC->MoveTo(MapX(0), MapY(m_samples[0]));
    for (int i = 1; i < m_count; ++i)
        pDC->LineTo(MapX(i), MapY(m_samples[i]));   // slow for large m_count
}
```

**After:** GPU series with batched, decimated real-time append (vendor-neutral
via `IChartHost`, [06 §5](06-visualization-layer.md#5-charting-abstraction-vendor-and-framework-agnostic)).

```csharp
// In a GraphUnitViewModel render tick (display-rate, not data-rate)
_series.AppendBlock(timeSpanBlock, valueBlock);   // engine decimates to viewport
```

---

## 4. Worker thread + `PostMessage` → Channels + async/await

**Before (MFC):** `AfxBeginThread`, `::PostMessage(hwnd, WM_USER, ...)`, manual
critical sections to ferry data to the UI.

**After:** bounded `Channel<T>` producer/consumer; UI updates via `await` (no
manual marshaling — the WPF sync context handles it).

```csharp
// Producer: acquisition/pipeline writes blocks
await channel.Writer.WriteAsync(block, ct);

// Consumer on UI: await marshals back to the UI thread automatically
await foreach (var block in channel.Reader.ReadAllAsync(ct))
    page.Apply(block);   // no Dispatcher.Invoke needed in async context
```

---

## 5. `CFile`/`CArchive` parsing → `Span`-based pipeline stage

**Before (MFC):** `CFile::Read` into buffers, pointer casts, endian by hand,
copies everywhere.

**After:** zero-copy span parsing in a pipeline stage; pooled buffers.

```csharp
static void Decommutate(ReadOnlySpan<byte> frame, ParameterMap map, Span<double> outRaw)
{
    foreach (var p in map.Parameters)
    {
        uint word = BinaryPrimitives.ReadUInt32BigEndian(frame.Slice(p.ByteOffset, 4));
        outRaw[p.Index] = ExtractBits(word, p.StartBit, p.BitLength);   // no allocations
    }
}
```

---

## 6. `CString`/`CArray`/`afxMessageBox` → modern equivalents

| MFC | Modern .NET |
|---|---|
| `CString` | `string` / `ReadOnlySpan<char>` |
| `CArray`, `CList`, `CMap` | `List<T>`, `Dictionary<TK,TV>`, `Span<T>` |
| `AfxMessageBox` | `IDialogService` (testable) abstraction over `MessageBox` |
| `CWinThread`/`::PostMessage` | `Task`/`Channel<T>`/`IAsyncRelayCommand` |
| `SetTimer`/`OnTimer` | `DispatcherTimer` (UI) / `PeriodicTimer` (logic) |
| Registry/INI/`Profile` | `IOptions<T>` + JSON settings |
| `DoModal` dialogs | `IDialogService.ShowDialogAsync<TViewModel>()` |

---

## 7. Conversion principles (apply everywhere)

1. **Logic → ViewModel/service; XAML for layout only.** No business logic in code-behind.
2. **Bind, don't push.** Replace `UpdateData`/manual control updates with bindings.
3. **Inject, don't reach.** Replace globals/`AfxGetApp()` with DI.
4. **Async, don't block.** Replace worker-thread + `PostMessage` with `Channel`/`await`.
5. **Span, don't copy.** Replace buffer copies with `Span<T>`/pooled memory on hot paths.
6. **Abstract the platform.** Dialogs, files, charts behind interfaces → testable + portable.
7. **Theme via resources.** `DynamicResource` brushes; no hardcoded colors ([14](14-cross-cutting-concerns.md)).

---

### Next
→ [11 — Module implementation guide](11-module-implementation-guide.md)
