# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file presenter deck: `fortiaigate.html` (~2,500 lines) containing all CSS, HTML, and JS
inline. It is a live-demo sales/technical talk — "Securing AI Agents" — built around FortiAIGate
and the OWASP Top 10 for LLMs. There is no build step, no package manager, no tests, no
dependencies except the Google Fonts `Inter` stylesheet. `assets/` holds Fortinet SVG icons and
product screenshots referenced by relative path.

## Commands

```bash
# Preview (relative asset paths work over file:// too, but serve to match Pages)
python3 -m http.server 8000    # then open http://localhost:8000/fortiaigate.html
```

Deploy: pushing to `main` triggers `.github/workflows/pages.yaml`, which generates an
`index.html` redirect to `fortiaigate.html` and publishes the repo root to GitHub Pages.
Nothing is generated at build time beyond that redirect — edit `fortiaigate.html` directly.

## Architecture

Everything after `<script>` (~line 776) is a small data-driven renderer. The DOM in `<body>` is a
fixed skeleton of container divs; all content is rendered into it from JS data structures.

### Coordinate systems (two nested fixed canvases)

- `#deck` is a fixed **1920×1080** design canvas. `fitDeck()` sets `--deck-scale` to
  `min(vw/1920, vh/1080)` so the authored layout is never stretched or clipped; the body
  background fills the letterbox/pillarbox bars so they blend.
- Inside an act, `#diagramStage` is a fixed **960×540** logical stage. `fitStage()` sets
  `--diagram-scale`. Node `x`/`y` in `NODES`, zone rects in `getZones()`, and badge positions in
  `BADGES` are all in these 960×540 logical units.
- `updateConns()` draws connector lines into an SVG with `viewBox="0 0 960 540"`, converting
  live `getBoundingClientRect()` values back into logical units by dividing by the rendered
  scale. Any layout change that moves nodes needs `updateConns()` re-run (it is already wired to
  `render()`, `resize`, and `fullscreenchange`).

### The flow

`FLOW` is the linear presenter order — the single source of truth for navigation and the top
timeline. Each entry is either `{type:'act', n}` (renders the diagram view via `render()`) or
`{type:'slide', id}` (renders `SLIDES[id].html` into `#slideLayer`). The title screen
(`#introOverlay`) sits before `FLOW[0]`; the closing summary (`#summaryOverlay`) and the "Try it"
screen sit after the last entry.

Adding a slide means adding an entry to both `SLIDES` and `FLOW` — the timeline, nav buttons, and
clicker all derive from `FLOW`.

### Clicker sub-steps

Navigation is two-level: `flowIdx` (which stop) and `subStepIdx` (position within that stop).
`clickerNext()`/`clickerPrev()` walk sub-steps first, then advance the stop. `maxSubStep(i)` and
`applyStop()` define per-stop behavior:

- `ACT_STEPS[n]` lists an act's steps — `{panel:'nodeId'}` opens that node's detail panel exactly
  as a click would, `{sim:'soc'|'socMesh'|'attack'|'code'}` launches a full-screen auto-playing
  example overlay.
- The `agency` slide has hardcoded sub-steps 0–4 (compare → gen demo → back → agent demo → back).
- Mouse interaction must keep `subStepIdx` in sync — see `agencyPick()`/`agencyReturn()` for the
  pattern.

Nav is driven by presentation remotes: PageDown/PageUp/Space/Backspace plus arrows. `F` toggles
browser fullscreen, `P` toggles projector mode. All handled in one `keydown` listener that
branches on which overlay is currently up.

### Diagram content model

- `ACTS` — four acts, each with `accent`, copy (`subtitle`, `sellerTip`, `narration`) and a
  `tagLabel`. `ACT_COLORS`/`ACT_RGB` index by act.
- `NODES` — every diagram element across all acts in one array, tagged with `act`. Visibility is
  cumulative (`n.act <= curAct`) except acts 2 and 3 nodes, which are hidden once act 4 (the
  governed state) is reached. Special node kinds, each with its own branch in `renderDiagram()`:
  `isProxyBox` (the act-4 gateway block, whose `proxyItems` pills open other nodes' panels),
  `isHidden` (detail nodes reachable only via those pills), `isEdgeAnchor` (invisible connector
  endpoints), `optional` (renders an "opt" badge).
- `CONNS` — `[fromId, toId, act, color]`; act-4 connectors route through the `n9L`/`n9R` anchors.
- Node bodies are HTML strings. `g('term')` wraps a term in a glossary tooltip sourced from
  `GLOSS`; the tooltip is re-rendered into a body-level `#floatTip` via delegated hover so it
  escapes overflow clipping. Adding a term means adding it to `GLOSS` and calling `g()`.
- `ICONS` — inline Lucide-style 24×24 SVG strings built by `_ic()`. Use these, not emoji.

### Theming

Three orthogonal modes, all driven by CSS custom properties on `:root` / `html`:
dark (default), `.light-mode`, and `html.projector-mode` (high-contrast for washed-out projectors,
with a `.projector-mode.light-mode` variant). New surfaces must read the design tokens
(`--border-*`, `--surface-*`, `--text-*`, `--shadow-*`) rather than hard-coding colors — projector
mode works by overriding those tokens, and anything hard-coded needs a manual override added to
the `html.projector-mode` block. Brand colors live in `C` (JS) and `--ft-*` (CSS).

### Simulations

Overlay-based, timer-driven demos: the agency gen/agent comparison, the SOC agent sims
(`SOC_SIMS`, `runSocSim()`), the SOC mesh view, and the live code-completion sim (`runCodeSim()`).
They share cancellation infrastructure — `agencyTimers` + `aT()`, `clearAgencyTimers()`,
`clearSocStreamLoop()`, `clearAct3Pulse()`. Any new timed animation must register its timers so
navigating away (`goFlow()` clears them) doesn't leave a sim playing over the next stop.

## Conventions

- Keep it a single self-contained file — no new files, no bundler, no external JS/CSS.
- Content and layout are data, not markup: change `ACTS`/`NODES`/`SLIDES`/`FLOW`, not the static
  `<body>` skeleton, unless adding a genuinely new container.
- Copy is presenter-facing and carefully worded; `seller`/`sellerTip` fields are talk-track notes
  shown in panels. Preserve the OWASP LLM code references (LLM01/LLM02/LLM05/LLM06/LLM10) and the
  2025 Top 10 framing when editing copy.
- Commit messages are short imperative summaries of the deck change (e.g. "Add in-deck fullscreen
  toggle (button + F key)").
