# 01 — Product Analysis: IDE (Integrated Debriefing Environment)

> Source of truth for this analysis: the public product page
> [plris.com/component/products/IDE.php](https://www.plris.com/component/products/IDE.php)
> and the PLR Systems product catalogue. Items quoted verbatim are in "quotes".
> Anything inferred (not stated on the site) is explicitly marked **(inferred)**.

This document captures **what IDE is today** so that every later document can be
traced back to a concrete, existing capability. The golden rule of the whole
program is **functional parity first** — we cannot modernize what we have not
first pinned down.

---

## 1. Identity & domain

| Attribute | Value |
|---|---|
| Product | Integrated Debriefing Environment (**IDE**) |
| Vendor | PLR Systems Ltd. (Netanya, Israel) |
| Domain | Flight-test instrumentation (FTI), avionics integration & test, telemetry |
| Primary users | "design and development engineers during integration and test phases" |
| Platform (today) | Windows desktop, MFC / C++ **(inferred from the job brief)** |
| Role in ecosystem | Ground/lab software companion to PLR's airborne **recorders** & telemetry hardware |

IDE is a **debriefing & data-analysis workstation**: it acquires data live from
test buses, records it, and lets engineers replay, visualize, search and export
it after a test. It is *not* a general IDE in the "code editor" sense — the
acronym is domain-specific.

---

## 2. The three functional modules

The product is explicitly organized into three modules. These become the three
top-level **functional areas** of the modern app.

### 2.1 Setup module
Defines *what* the system knows about the data:
- Create/modify **setups** (the master configuration document).
- Define **Data Source Handlers (DSH)** — one per physical/logical input channel.
- Configure **communication parameters** per channel (bus type, rates, addressing).
- Define **messages** and **sub-messages** carried on each channel.
- Define individual **parameters** with: name, bit/word **position**, data **type**,
  **calibration** formula, **display format**, and **limits**.
- Define **compound parameters**: "new data parameters may be defined as
  algebraic/logic expressions/functions (both built-in and user defined)".
- Organize parameters into **subsets** for display/reporting.
- Lay out **display pages** containing graph units.

### 2.2 Debriefing Data module
The live & replay cockpit:
- **Real-time display concurrent with recording.**
- **Playback** with forward/backward stepping, adjustable speeds.
- **Event marks**, **trigger** activation.
- **Parameter redefinition** during a session.
- Synchronous display of **asynchronous** channels.
- **Search** for conditions: exceedences, counters, logical conditions.
- **Zoom** in time and on axes.
- **Alarm and exceedence coloring.**

### 2.3 Data Dump module
The batch export engine:
- Convert flight-recorded data to **calibrated ASCII**.
- Extract parameter subsets in **binary** form.
- **Batch** file processing.
- Produce **time-tagged** output compatible with **EXCEL** and **MATLAB**.

---

## 3. Inputs, buses & data types

IDE provides an "asynchronous multi-channel standard interface to numerous input
channels":

| Bus / interface | Notes for modernization |
|---|---|
| **PCM** (telemetry) | High-rate streaming; decommutation of frames/sub-frames |
| **Ethernet** | Network-delivered data |
| **UART** (serial) | Lower-rate serial links |
| **MIL-STD-1553** (MuxBus) | Avionics command/response bus; BC/RT/MT modes |
| **ARINC-429** | Avionics data bus (label-based words) |
| **Digital I/O** | Discrete signals |

Display data **formats**: "decimal, hex, literal, degrees, ASCII, binary, time,
and user-defined".

**Synchronization:** "on-line synchronization of all inputs with up to **1 µs**
accuracy" — a hard real-time, hardware-timestamp requirement (IRIG/PTP-class).
**(inferred: the 1 µs is anchored in hardware, not software.)**

---

## 4. Visualization surface

- "Up to **100 display pages**" of live graph units, and "**1000 designated
  pages** of users selected parameters" for review.
- Graph unit types: **tables, bars, X/t graphs, X/Y graphs, gauges, stacks,
  strings, messages**.
- **Alarm/exceedence coloring** on values.
- **Zoom** (time and axis).
- Menu-driven operation; four UI screenshots are referenced on the site.

This is a **data-dense, high-refresh** UI — the defining technical challenge of
the modernization (see [06 — Visualization layer](06-visualization-layer.md)).

---

## 5. Outputs & integrations

- Dump displayed information to **ASCII or binary** files.
- **Copy pictures** (chart images to clipboard/file).
- Upload data to **EXCEL** and **MATLAB**.
- **LAN** network data transfer.

**Recorder integration** — IDE is the analysis front-end for PLR recorders:
- High-Definition Video/Data Recorder
- Digital Video/Data Recorder
- Telemetry Video/Data Recorder
- PCM Recorder

**(inferred)** This implies IDE must read PLR's proprietary **recording file
formats** and likely correlate **video** with parameter data — a first-class
backward-compatibility requirement (see [09 — Recording & playback](09-recording-and-playback.md)).

---

## 6. Derived non-functional requirements

The site does not state numbers, but the feature set implies these NFRs. They
are **assumptions to confirm** with PLR (see
[16 — Discovery questions](16-discovery-questions.md)) and they drive the
architecture.

| NFR | Implied requirement | Why it matters |
|---|---|---|
| Throughput | Sustained high sample rates across many channels (PCM can be Mbit/s-class) | Drives the zero-copy ingestion pipeline & GPU charting |
| Latency | Live display "concurrent with recording" | Acquisition must be decoupled from rendering |
| Sync | 1 µs cross-channel | Hardware timestamps preserved end-to-end |
| Scale | ≤1000 pages, many parameters/page | Page virtualization, on-demand rendering |
| Determinism | No dropped samples while recording | Bounded back-pressure, no GC stalls on the hot path |
| Fidelity | Bit-exact decommutation & calibration | Parity tests vs the legacy app |
| Compatibility | Read legacy setups & recordings | Importers/readers as first-class components |

---

## 7. Capability → modernization map (traceability)

Every existing capability must land somewhere in the new design. This table is
the master checklist for **parity**.

| Existing capability | New home | Doc |
|---|---|---|
| DSH / channel / message / parameter model | `IDE.Core` domain model | [08](08-core-engine.md) |
| Calibration & display formatting | `IDE.Core` calibration + format services | [08](08-core-engine.md) |
| Compound parameters (expressions) | `IDE.Core` expression engine (compiled) | [08](08-core-engine.md) |
| Multi-bus acquisition + 1 µs sync | `IDE.Acquisition.Native` (wrapped) + pipeline | [07](07-data-acquisition-interop.md) |
| Real-time display | `IDE.App.*` + commercial GPU charting | [06](06-visualization-layer.md) |
| Playback (step/speed/search/zoom) | `IDE.Recording` playback controller | [09](09-recording-and-playback.md) |
| Recording (concurrent) | `IDE.Recording` writer (MMF/async I/O) | [09](09-recording-and-playback.md) |
| Alarms / exceedence / triggers | `IDE.Core` alarm & search engine | [08](08-core-engine.md) |
| Data Dump (ASCII/binary/Excel/MATLAB) | `IDE.Export` | [11](11-module-implementation-guide.md) |
| Display pages / subsets | `IDE.App.Core` page system + layout | [06](06-visualization-layer.md) |
| Legacy setup & recording files | Importers/readers | [09](09-recording-and-playback.md) |

---

## 8. What we do **not** yet know

These gaps are intentional and tracked in
[16 — Discovery questions](16-discovery-questions.md). The biggest unknowns:

1. **Native driver/SDK availability** — can we wrap existing C/C++ drivers, or
   must we re-integrate vendor SDKs? (Linchpin of the whole approach.)
2. **Exact data rates & channel counts** — sizes the pipeline and the charting
   spike.
3. **Legacy file formats** — setup files and recordings (and video container).
4. **Codebase size & state** — LOC, modularity, build system, documentation.
5. **Cross-platform need** — Windows-only (WPF) vs also Linux (Avalonia).
6. **AI appetite & data-governance** — what, if anything, can leave the lab.

---

### Next
→ [02 — Modernization strategy](02-modernization-strategy.md)
