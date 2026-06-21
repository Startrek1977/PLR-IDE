# 16 — Discovery Questions for PLR

The exhaustive question list to turn assumptions into facts before/at kickoff.
Grouped by theme; **★** marks the **gating** questions that change architecture or
estimates. Answers feed the POC ([12](12-migration-roadmap.md)) and resolve the
open items flagged across the docs.

> How to use: send as a questionnaire; capture answers inline; convert each
> answer into a decision/assumption-removed note in the relevant doc.

---

## A. Existing codebase & build
1. ★ How large is the MFC/C++ codebase (LOC, # files/classes, # projects)?
2. ★ Will we get **full source access**? Under what terms (NDA, on-prem only)?
3. What toolset/compiler/MFC version? 32-bit, 64-bit, or both?
4. Build system (Visual Studio .sln/.vcxproj? scripts? CI)? Can it still build today?
5. Third-party libraries in use (charting, math, file, networking)? Licenses?
6. How modular is it (clean layers vs UI-coupled logic)? Any existing tests?
7. Existing documentation (architecture, file formats, user manual)?
8. Coding standards / patterns we should respect or deliberately replace?

## B. Hardware, drivers & acquisition ★
9. ★ For each bus (PCM, 1553, ARINC-429, UART, Ethernet, I/O): which **vendor
   cards/devices** and **SDK versions**?
10. ★ Do we have **driver source**, or only vendor **SDK DLLs**? C or C++ API?
11. ★ Are SDKs available for **64-bit** / current Windows? Any Linux SDKs (affects
    Avalonia viability)?
12. ★ What is the **1 µs synchronization** source — IRIG-A/B, PTP/IEEE-1588, GPS,
    card hardware timestamp? Who stamps the time?
13. How does the app currently receive data — callbacks, polling, DMA, shared
    memory?
14. Is real hardware (or a simulator) available to us for development & testing?

## C. Data scale & performance ★
15. ★ Peak/aggregate **data rates** (e.g., PCM Mbit/s, frames/s) and typical
    channel/parameter counts?
16. ★ Max **simultaneous live parameters** and **plots on screen**?
17. Typical & max **recording duration** and resulting **file sizes**?
18. Display refresh expectation (Hz)? Acceptable end-to-end live latency?
19. Any hard real-time/loss guarantees we must contractually meet?

## D. File formats & compatibility ★
20. ★ **Setup file** format & version history — spec available? Sample files?
21. ★ **Recording** format(s) for HD/Digital/Telemetry/PCM recorders — specs?
    Samples? Must the new app read **all** of them?
22. Must the new app **write** legacy formats too, or only read them?
23. Is **video** part of recordings? Containers/codecs? Must video be
    **correlated/synced** with parameters in playback? (→ [09](09-recording-and-playback.md))
24. Export specifics: exact **ASCII** layout, **binary** layout, **MATLAB**
    (.mat version) and **Excel** expectations engineers depend on?

## E. Functional parity & scope
25. ★ Is the goal **strict 1:1 parity** first, or are UX redesigns welcome from
    the start?
26. Which features are **most used / most loved**? Any that are **unused** (safe
    to drop)?
27. Any known **pain points / bugs / missing features** to fix during modernization?
28. Beyond the website list, are there **hidden/advanced** features (scripting,
    macros, custom plug-ins, report templates)?
29. The expression engine: examples of real **compound-parameter formulas**?
    Which **built-in functions** must exist? User-defined function mechanism?
30. Trigger/event/exceedence semantics — exact rules, dwell/counter behavior?

## F. Platform, deployment & environment ★
31. ★ **Windows-only** acceptable (→ WPF), or is **Linux/cross-platform** required
    (→ Avalonia)? (→ [05](05-ui-platform-options.md))
32. Target Windows versions? Typical workstation specs (CPU/RAM/GPU)?
33. Is a **GPU** present on target machines (needed for commercial charting)?
34. **Air-gapped** / offline deployment? Internet ever available?
35. Install/update mechanism preference (MSIX, installer, copy-deploy)?
36. Multi-monitor / high-DPI setups in the field?

## G. Charting & visualization
37. ★ Is a **commercial charting license** (SciChart/LightningChart) acceptable
    (budget + redistribution terms)? Any vendor preference/prior use?
38. Are there **must-have** chart interactions (cursors, annotations, multi-axis,
    specific gauge styles) beyond the legacy set?
39. Any branding/visual standards or a desired modern look (dark mode default)?

## H. AI (optional)
40. ★ Is **AI** desired now, later, or not at all? Any specific use case in mind?
41. ★ **Data-governance**: can telemetry be processed by **cloud/LLM** services,
    or **strictly on-prem/local** only? (→ [13](13-ai-integration.md))
42. Which would deliver most value: anomaly detection, auto-summaries, NL query,
    Setup copilot, predictive trends?

## I. Security & compliance ★
43. ★ Is the product/source **export-controlled (ITAR/EAR)** or otherwise
    classified? What are the handling rules for us?
44. Encryption-at-rest for recordings/setups required? Access control / audit?
45. Any certification/qualification standards the software must meet?
46. Where may the **GitHub repo** live (public/private/self-hosted)? Any IP
    constraints on these very docs?

## J. Project, team & commercials
47. ★ Desired **timeline / milestones**? Any fixed deadlines (program need)?
48. Budget envelope / engagement model (fixed-bid vs T&M vs staff-aug)?
49. Team: who from PLR is available (domain experts, original developers)?
50. Will we provide the **whole** team or augment PLR engineers (who then maintain it)?
51. Acceptance process & environment — how is **parity** signed off, and by whom?
52. Pilot/rollout plan — can we run new & legacy **side-by-side** during transition?
53. Maintenance/support expectations post-delivery?

---

## Answer-tracking template

| Q# | Answer | Decision / assumption removed | Doc to update |
|---|---|---|---|
| e.g. 10 | "Vendor SDK DLLs, C API, 64-bit" | Use P/Invoke bridge | [07](07-data-acquisition-interop.md) |
| | | | |

---

← Back to [README](../README.md) · Start at [01 — Product analysis](01-product-analysis.md)
