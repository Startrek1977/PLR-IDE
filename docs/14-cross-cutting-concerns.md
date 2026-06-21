# 14 — Cross-Cutting Concerns

Concerns that span every module and run in **every phase**
([12](12-migration-roadmap.md)), not bolted on at the end.

---

## 1. Testing strategy

| Layer | Tests | Tooling |
|---|---|---|
| Engine (`IDE.Core`) | Calibration, expression compile/eval, conditions | xUnit + FluentAssertions |
| Pipeline | Decommutation correctness, back-pressure, no-drop | xUnit + synthetic streams |
| **Parity (critical)** | Byte/numerically identical to legacy on real recordings | Custom **golden-file** harness over `IDE.Pipeline`/`IDE.Export` |
| Performance | Throughput (loss-free), UI tick allocations, page-switch | BenchmarkDotNet + macro throughput harness in CI |
| Recording | Round-trip new format; read all legacy formats | xUnit + sample files |
| UI | Smoke + key flows | FlaUI/WinAppDriver (WPF), Avalonia.Headless |

The **replay source** ([07 §6](07-data-acquisition-interop.md#6-testing-without-hardware-the-replay-source))
lets all of this run **without lab hardware**. See [skill:dotnet-ui-testing-core].

---

## 2. Theming & UX

- **`DynamicResource` everywhere** for brushes/colors; **no hardcoded colors** —
  so themes swap at runtime. See [skill:wpf-theming].
- `IThemeService` with dark/light (and high-contrast for accessibility). A dark,
  low-eyestrain theme suits long debrief sessions in dim labs.
- `WindowChrome` for a modern title bar **without** `WindowStyle=None`.
- Consistent value formatting via the shared `IValueFormatter`
  ([08](08-core-engine.md)).
- Accessibility: keyboard navigation, automation peers, sufficient contrast — see
  [skill:accessibility].

---

## 3. Logging & diagnostics

- **Microsoft.Extensions.Logging** abstraction; **Serilog** provider optional
  (file/console sinks; **offline-friendly**, no cloud required).
- Structured logs around acquisition start/stop, dropped-frame *warnings*,
  recording lifecycle, export results.
- A lightweight **diagnostics page** (throughput, buffer fill, dropped counts,
  GC stats) — invaluable during real-time bring-up.

---

## 4. Configuration & settings

- `IOptions<T>` + JSON store replaces `Properties.Settings`/INI/registry.
- Persist window placement, recent files/setups, theme, chart preferences,
  acquisition-source selection (Native/Interop/Replay).

---

## 5. High-DPI & multi-monitor

- **Per-Monitor V2** manifest; `UseLayoutRounding`/`SnapsToDevicePixels`.
- Engineering rigs are often multi-monitor at mixed DPI — pages must stay crisp
  and tear-off/secondary-window friendly. See [skill:dotnet-wpf-modern] (High-DPI chapter).

---

## 6. Localization

- `.resx` + `IStringLocalizer` if multi-language is needed (Israeli vendor; flight
  test is often English). RTL support available if Hebrew UI is wanted. Confirm in
  [16](16-discovery-questions.md). Keep strings externalized from day one to avoid
  rework even if only English ships first.

---

## 7. Security & export control (ITAR / defense) — **important**

This is a defense/aerospace product from an Israeli vendor; treat compliance as a
first-class constraint:

- **Source & data handling** may be export-controlled — confirm what may be shared,
  where it may be stored, and who may access it ([16](16-discovery-questions.md)).
- **Air-gap friendly:** the product must run **fully offline**. No mandatory cloud
  services; all dependencies vendorable for disconnected build/deploy.
- **AI governance:** any external/LLM call is opt-in, policy-gated, audited, and
  disable-able ([13](13-ai-integration.md)). Default to **local** models.
- **At rest:** consider encryption of recordings/setups if required; access
  control and audit logging for sensitive operations.
- **Supply chain:** pin dependencies; review licenses (commercial charting,
  expression libs); maintain an SBOM.

---

## 8. Packaging & deployment

- **MSIX** (clean install/update, per-machine) or self-contained
  `dotnet publish` for **offline** lab deployment.
- `TrimMode=partial` only (WPF relies on XAML reflection — never full trim).
- Side-by-side .NET (no machine-wide framework dependency).
- Versioned recording/setup formats so new builds read old files.

---

## 9. Performance discipline (recap)

- Acquisition decoupled from rendering; display-rate sampling
  ([06](06-visualization-layer.md)).
- Pooled buffers, `Span<T>`, struct frames, no hot-path allocations
  ([04](04-technology-stack.md)/[07](07-data-acquisition-interop.md)).
- CI throughput gates catch regressions before they ship.

---

### Next
→ [15 — Risks & mitigations](15-risks-and-mitigations.md)
