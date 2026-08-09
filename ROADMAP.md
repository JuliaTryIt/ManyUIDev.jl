# ManyUI Framework — Roadmap & Vision

This document outlines the strategic roadmap for the **ManyUI** framework to
become a fully functional, production-ready GUI toolkit for Julia. It lives in
the `ManyUIDev.jl` superproject because it spans every submodule.

## 1. Core Framework Maturation

- [ ] **Comprehensive Test Coverage & CI/CD**: Integrate Aqua.jl, CodeCov, and TestItemRunner across all sub-packages (`ManyUI`, `ManyUITUI`, `ManyUIWeb`).
- [ ] **Documentation**: Write extensive docstrings (using DocStringExtensions) and build a Documenter.jl site with tutorials, a gallery, and architectural deep-dives.
- [ ] **Error Handling**: `ErrorBoundary` exists in `ManyUI`; still to do is making a crashing widget provably unable to tear down the event loop or corrupt terminal state.

## 2. Advanced Widget Library

- [ ] **Disabled State**: Native support for the `disabled` property on all interactive widgets (`Button`, `TextInput`, `Checkbox`, `Dropdown`) to prevent interactions and apply a dimmed style.
- [x] **ProgressBar**: Support for indeterminate or determinate loading states.
- [x] **Spinner** and **Slider**: shipped (`ManyUI/src/widgets/spinner.jl`, `slider.jl`).
- [ ] **TextInput Enhancements**: Support for password mode (rendering `***`) and input filtering/masks (e.g., numeric-only).
- [ ] **Tooltips**: Support for hover tooltips on widgets to display contextual help text.
- [ ] **Modals & Dialogs**: `Popup` and `MinSizeOverlay` exist; a true `Modal` (dimmed backdrop, focus trap, button row) and the alert/confirm/file-picker helpers do not.
- [ ] **Advanced Layouts**: `Tabs`/`TabStrip` shipped. Accordions and **Splitters (draggable panes)** remain.
- [ ] **Rich Text & Data**: `DataTable` sorts (stable, index-permuting). A Markdown rendering widget and column filtering remain.
- [ ] **RawHTML Escape Hatch**: Allow injecting raw HTML/CSS strings when using the web backend (with graceful fallbacks in TUI) for custom web-specific tweaks.

## 3. Theming & Styling

- [ ] **Theme System**: there is none today — `ManyUI` has `Color`, `Style` and a CSS cascade, but no named palettes, no semantic tokens (`accent`, `warning`, `text-dim`), no registry.
- [ ] **Dynamic Theming**: Support for hot-swapping themes (e.g., Light/Dark mode transitions) at runtime, with persistence via Preferences.jl.
- [ ] **CSS Expansion**: Expand the pseudo-CSS parser to support more properties (e.g., `margin`, `z-index`, `opacity` for web).

## 4. Backend Expansion (The "Many" in ManyUI)

The overarching goal of ManyUI is to write the UI once and project it anywhere.
`TUI`, `WebTerminal` and `WebNative` are usable today.

- [ ] **Dear ImGui Backend**: `ManyUICImGui` exists but is a skeleton (~700 LOC) — it cannot yet render a non-trivial multi-pane application. Bring it to parity with `ManyUITUI`.
- [ ] **Native Desktop**: Explore bindings to native OS controls (like GTK or macOS Cocoa) or web-view wrappers (like Blink.jl or Electron).
- [ ] **Makie & Data Science Integration**: Deepen support for data-centric apps. Allow rendering ManyUI widgets over Makie.jl plots, embedding Makie plots inside ManyUI containers, and adding native, fast `DataFrame` viewing components.

## 5. Interaction & Events

- [ ] **Mouse Support**: Improve drag-and-drop, scroll-wheel handling, and hover states across all backends.
- [ ] **Focus Management**: `focus!`, `focus_next!`, `focus_prev!` and `focusable_widgets` exist. Focus *rings* driven by CSS pseudo-classes remain.

## 6. Developer Experience (DX)

- [x] **Immediate Mode API**: `ManyUI.Immediate` lets users write top-to-bottom scripts (`if button("Click me") ... end`) that compile into the formal tree under the hood.
- [ ] **Hot Reloading**: Enable Revise.jl to hot-reload widget definitions without restarting the event loop.
- [ ] **Debugger GUI**: A built-in inspector (like Chrome DevTools) to visualize the widget tree, CSS bounding boxes, and performance metrics in real-time.

## 7. Modality-Specific UX (The Anakin/Padmé Principle)

While ManyUI allows writing a unified codebase for multiple platforms, we must
ensure we **tailor the UX to each modality**.

- [ ] **Terminal Conventions**: Support standard terminal bindings natively (e.g., `Esc` or `q` to quit, `Ctrl-C` handling, `Tab` focus cycling).
- [ ] **Web Conventions**: Support browser navigation paradigms (back/forward), responsive touch targets for mobile, and URL routing mapping to UI states.
- [ ] **Graceful Degradation**: If a widget relies on advanced graphics (like Canvas), provide a clean fallback for pure TUI environments.

## 8. Ecosystem Synergies (Learning from Python's Typer, Rich, Textual)

ManyUI aims to give Julia the DX Python enjoys for terminal apps:

- [ ] **CLI Generation (Typer analog)**: Continue leveraging **Comonicon.jl** for robust, type-hinted command-line interfaces.
- [ ] **Rich Formatting & Widgets (Rich analog)**: Deepen the integration with **Tachikoma.jl** to bring highly styled text, advanced terminal layouts, and beautiful UI widgets seamlessly into ManyUI.
- [ ] **Full-Screen TUI (Textual analog)**: Position `ManyUITUI` as the premier Textual alternative in Julia — using web paradigms (CSS, DOM, Reactivity) tailored strictly for high-performance, full-screen terminal applications.

## 9. Upstream Contributions (Tachikoma.jl)

To fully integrate `Tachikoma.jl` natively (especially for `ManyUIWeb`):

- [ ] Remove process-global I/O state (`T.INPUT_IO`) in favor of context-aware session handling to allow for multi-tenant web sessions.
- [ ] Implement responsive layout constraints across core Demos so they do not crash or fail to render in small (or dynamically sizing) terminal windows (e.g. `xterm.js` during initial load).
- [ ] Provide APIs for rendering single widgets headlessly or statically without taking over the full terminal via an event loop.

---

## 10. Reference Target: Kaimon TUI Parity

*Added 2026-08-09. Gap analysis performed against `Kaimon/src/tui/` (~17 500 LOC,
Tachikoma-based) — a real, non-trivial, in-production terminal application.*

### Why this target

Feature checklists drift. A concrete application does not. The Kaimon TUI is a
10-tab full-screen supervisor for an MCP server: a live log pane, two data
tables with a detail panel, an animated per-session ECG trace, a tool-call
activity feed with a sparkline, ~15 modal configuration flows, a Markdown
history pane, a syntax-highlighted code editor, and an embedded PTY terminal.
"Could ManyUI render this?" is a sharper question than any feature list, and the
answer today is *no* — for reasons worth writing down.

The payoff if we close the gap: the same supervisor UI would also run in a
browser through `ManyUIWeb`, which Tachikoma cannot do.

### The three structural blockers

**10.1 — Inline styled text.** Kaimon builds **181** `Span`s: text runs carrying
their own style *inside* a single line. `ManyUI.Label` held a `Reactive{String}`
with one CSS-resolved style, and `TabStrip.titles` is a `Vector{String}`. Even
"**1** Server" with the digit in yellow was inexpressible.

- [x] `RichText`/`TextRun` in `ManyUI`, with `text_width`, `truncate_width` and `wrap_width` over them. Named `TextRun`, not `Span`: `ManyUITUI` already exports a `Span` (a run of changed cells in a diff patch) and two exported `Span`s would collide for anyone doing `using ManyUI, ManyUITUI`.
- [x] `Label` and `Static` carry a `RichText`; `write_richtext!` paints one in `ManyUITUI`.
- [x] `TabStrip.titles` are `RichText`, and `List`'s `format` / `Table`'s `cell` accept `TextLike`. All six row widgets got it from a single `RichText` overload of `_tc_slice!`, the painter they all funnel through.
- [x] `ManyUIWeb` projects runs as `<span>`s, so a rich list or table reaches the browser styled rather than flattened.

The wrap invariant is the load-bearing part and is worth restating: wrapping
runs the *plain* text through the existing string wrap and reattaches styling,
so `plain.(wrap_width(rt, w)) == wrap_width(plain(rt), w)` holds by
construction. Colouring a paragraph cannot reflow it, and there is only one
wrap implementation to keep correct.

**10.2 — Full-screen buffer access.** `_paint_node!` (`ManyUITUI/src/paint.jl:134`)
calls `render!(w, view(buf, inter))`: every widget receives a view **clipped to
its own content box**, and painting outside it is impossible by construction.
Kaimon relies on the opposite: `_dim_area!(buf, f.area)` dims the whole screen,
`render_resize_handles!(buf, layout)` paints over neighbouring pane borders.
This is a restructuring, not a dead end — root-level overlays are the ManyUI
answer, and they are cleaner — but it touches every screen.

- [ ] Specify and document the root-overlay pattern (screen dimming, drag handles, transient chrome) so applications stop wanting the raw buffer.

**10.3 — OSC 8 hyperlinks.** Kaimon's `file_link_style` emits
`Style(..., hyperlink=url)`, making every displayed file path clickable straight
into the configured editor. `ManyUI.Style` (`ManyUI/src/style.jl:35`) is a
12-byte isbits struct — fg, bg, attrs, mask — with no room for a link. A side
table of link IDs alongside the buffer is the likely design.

- [ ] Add a hyperlink channel to the paint pipeline without inflating `Style`.

### Prioritised backlog

**P0 — nothing renders without these**

- [x] `RichText`/`TextRun`, carried by `Label`, `Static`, `TabStrip`, `List`, `Table`, `DataTable`, `TreeView` and `Checkbox`, and projected by both the TUI and the web backends (10.1).
- [x] Border **titles**: `border_title(w)`/`border_title_align(w)` are a seam any widget can override, painted by `paint_border_title!` right after the perimeter. `Container` ships `title`/`title_align`. A caption never touches a corner, and its runs fold over the border's style.
- [ ] Border **footers**, and *interactive* titles — Kaimon's `Server Log (24) [wrap:off] [F]ollow:on` is a clickable control living in the border, which needs hit-testing on the border box rather than the content box.
- [ ] Theme system with semantic tokens (§3). Kaimon calls `Tachikoma.theme()` 26 times and `tstyle(:token)` throughout.

**P1 — the ergonomics this class of app needs**

- [ ] `Splitter` — draggable panes with mouse handles and persisted geometry. 8 of Kaimon's 10 tabs depend on it.
- [ ] `Modal` — dimmed backdrop, focus trap, button row. Kaimon has ~15 such flows.
- [ ] `Gauge`, `Sparkline`, `StatusBar`, `ProgressList` — all cheap once P0 lands.
- [ ] CSS `:focus` pane rings. This is the clearest win: Kaimon's **63** hand-written `_pane_border`/`_pane_title` call sites collapse into a stylesheet rule.

**P2 — rich content**

- [ ] `MarkdownPane` (the `Markdown` stdlib supplies the AST).
- [ ] `CodeEditor` with syntax highlighting (JuliaSyntax.jl, or port Tachikoma's tokenizers).

**P3 — terminal-only, needs a decision**

- [ ] `TerminalWidget` + PTY: full VT emulation. Large, and meaningless under `ManyUICImGui`/`WebNative` — decide whether ManyUI owns this or delegates.
- [ ] Pixel graphics (Kitty/sixel), animation primitives (tween/spring/easing), session recording and SVG/GIF export.

### What we already have, and have better

Worth stating so the gap is not overestimated. Flex layout, the CSS cascade,
focus, mouse, `Scrollpane`/`Scrollbar`, `TreeView`, `Form`, `Checkbox`,
`RadioGroup`, `DropDown`, the `App`/`run!` loop with `set_interval!`/`call_later!`,
and `HeadlessDriver` for tests are all in place.

`Table`/`DataTable` are *stronger* than the Tachikoma equivalents: rows are held
as data, not widgets, so a frame costs the same at 100 000 rows as at 3 — which
is exactly what Kaimon's Sessions and Activity tabs want.

And much of Kaimon's bulk is accidental: hand-rolled scrolling
(`y_virtual = detail_area.y - scroll` with `in_view` guards, 15 `*_scroll` model
fields), manual focus rings, and 283 `set_string!`/`set_char!` coordinate
computations. In ManyUI those are the framework's job. A faithful port should be
substantially *shorter* than the original.

### Suggested first step

- [ ] Prototype the **Server tab alone** in ManyUI — outer frame, two blocks, a following log pane, a status bar. It is the simplest screen and it exercises every P0 item end to end.
