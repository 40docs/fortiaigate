# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file presenter deck: `fortiaigate.html` (~2,700 lines) containing all CSS, HTML, and JS
inline. It is a live-demo sales/technical talk — "Securing AI Agents" — about GenAI workloads
built on AWS, framed against the OWASP Top 10 for LLMs. The `main` branch holds the earlier
SOC-centric version (tag `v1-soc-storyline`).

**Voice:** the deck talks about the workload, not about a third party. No "your customer's data",
no "the AI they ship" — it's "the agent", "the application", "the AWS environment". The shared-
customer framing is something the presenter says out loud; it stays out of the copy. `seller` /
`sellerTip` fields are presenter notes and may address the room directly ("ask the team…"), but
they follow the same rule. Second-person "they/their" is reserved for actual third parties in the
story: attackers, end users, guardrail products, and quoted research. There is no build step, no package manager, no tests, no
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

### The flow (and the fork)

`FLOW` is the linear presenter order — the single source of truth for navigation and the top
timeline. Each entry is either `{type:'act', n}` (renders the diagram view via `render()`) or
`{type:'slide', id}` (renders `SLIDES[id].html` into `#slideLayer`). The title screen
(`#introOverlay`) sits before `FLOW[0]`; the closing summary (`#summaryOverlay`) and the "Try it"
screen sit after the last entry.

`FLOW` is assembled, not hardcoded: `FLOW_HEAD` (foundation + fork slides) + the selected entry of
`FLOW_PATHS` + `FLOW_TAIL` (product slides, shared). The `fork` slide is the choose-your-own-
adventure stop — its two cards call `choosePath('data'|'models')`, which runs `buildFlow()` and
jumps to the first stop of that path. The two paths are deliberately different problems:

- `models` — **the public → your customer's models** (inbound). The built-out path: acts 1–4 plus
  the OWASP slide.
- `data` — **your customer's data → public models** (outbound). One overview slide (`dataSoon`)
  covering that conversation. Only the `models` path is walked out in full, but nothing in the
  deck says so — the fork gives both paths equal weight on purpose, and the outbound slide reads
  as a finished summary rather than a placeholder. Keep it that way when editing.

Adding a slide means adding an entry to `SLIDES` and to the right `FLOW_*` array — the timeline,
nav buttons, and clicker all derive from `FLOW`.

### Clicker sub-steps

Navigation is two-level: `flowIdx` (which stop) and `subStepIdx` (position within that stop).
`clickerNext()`/`clickerPrev()` walk sub-steps first, then advance the stop. `maxSubStep(i)` and
`applyStop()` define per-stop behavior:

- `ACT_STEPS[n]` lists an act's steps — `{panel:'nodeId'}` opens that node's detail panel exactly
  as a click would, `{sim:'promise'|'mesh'|'attack'|'code'}` launches a full-screen auto-playing
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
- Three columns inside the 960×540 stage carry the whole story: the public zone on the left
  (`p*` nodes, x≈105–205), the customer's AWS workload on the right (`a*` nodes, x=850 and the
  app/agent column at x=445), and **the empty middle column at x=620** — the slot where the control
  point should be. That slot is re-used act to act: act 2 fills it with blind spots (`b*`), act 3
  with incidents (`r*`), act 4 with the gateway (`gw*`). Moving those x-coordinates breaks the
  whole point of the layout.
- `NODES` — every diagram element across all acts in one array, tagged with `act`. Visibility is
  cumulative (`n.act <= curAct`) with two exceptions in `renderDiagram()` that implement the
  middle-column hand-off: act-2 nodes hide once act 3 arrives, act-3 nodes once act 4 does.
  Special node kinds, each with its own branch in `renderDiagram()`: `isProxyBox` (the act-4
  gateway block, whose `proxyItems` pills open other nodes' panels), `isHidden` (detail nodes
  reachable only via those pills), `isEdgeAnchor` (invisible connector endpoints), `optional`
  (renders an "opt" badge), `simBtn` (declares the example button `openPanel()` renders).
- `CONNS` — `[fromId, toId, act, color]`; act-4 connectors route through the `gwL`/`gwR` anchors.
- Node bodies are HTML strings. `g('term')` wraps a term in a glossary tooltip sourced from
  `GLOSS`; the tooltip is re-rendered into a body-level `#floatTip` via delegated hover so it
  escapes overflow clipping. Adding a term means adding it to `GLOSS` and calling `g()`.
- `ICONS` — inline Lucide-style 24×24 SVG strings built by `_ic()`. Use these, not emoji.

### Theming

**Light is the deck's default.** The `.light-mode` class ships on `<html>` in the markup, and a
tiny pre-paint script in `<head>` removes it if the presenter's stored choice (`fag-theme` in
localStorage) is dark — so there's no flash on load. The toggle writes that key and shows the icon
for the mode it will switch *to*. Dark is therefore the `:root` palette with the class absent.

Three orthogonal modes, all driven by CSS custom properties: `:root` (dark values), `.light-mode`
(the default overrides), and `html.projector-mode` (high-contrast for washed-out projectors, with
a `.projector-mode.light-mode` variant).

**Always read the tokens** (`--border-*`, `--surface-*`, `--text-*`, `--shadow-*`) rather than
hardcoding a color. A hardcoded value does not flip with the theme *and* cannot be reached by
projector mode, which works by overriding those same tokens — that combination is what produced
the pile of one-off `.light-mode` patches this file used to carry. Brand colors live in `C` (JS)
and `--ft-*` (CSS); white text on a brand-colored fill (`color:#fff` on a button) is correct in
both themes and needs no patch.

Note the class name reads backwards from its behavior: `.light-mode` is present by default and
*removing* it gives dark. Inverting that properly (light in `:root`, a `.dark-mode` override)
would change the specificity of ~59 rules, which can't be verified here without a browser, so it
was left alone deliberately.

### Simulations

Overlay-based, timer-driven demos: the agency gen/agent comparison, the support-agent sims
(`AGENT_SIMS`, `runSim()` — `promise` does the job, `attack` gets injected), the "scale this to
production" view (`renderScaleView()`), and the builder's-IDE sim (`runCodeSim()`) showing how the
tool the agent abuses got written. Element ids still say `soc*` (`#socSim`, `#socInner`) from the
previous storyline; the functions no longer do.
They share cancellation infrastructure — `agencyTimers` + `aT()`, `clearAgencyTimers()`,
`clearSocStreamLoop()`, `clearAct3Pulse()`. Any new timed animation must register its timers so
navigating away (`goFlow()` clears them) doesn't leave a sim playing over the next stop.

## Conventions

- Keep it a single self-contained file — no new files, no bundler, no external JS/CSS.
- Content and layout are data, not markup: change `ACTS`/`NODES`/`SLIDES`/`FLOW`, not the static
  `<body>` skeleton, unless adding a genuinely new container.
- Copy is presenter-facing and carefully worded; `seller`/`sellerTip` fields are talk-track notes
  shown in panels, written for an AWS seller talking to a customer building on AWS. Preserve the
  OWASP LLM code references (LLM01/LLM02/LLM05/LLM06/LLM07/LLM08/LLM10) and the 2025 Top 10
  framing when editing copy.
- No browser or test runner is available in this environment, so verify changes by extracting the
  `<script>` block to a file and running it in Node against a stub `document`/`window` (elements
  need `classList`, `style.setProperty`, `innerHTML`, `appendChild`, `querySelector(All)`,
  `getBoundingClientRect`), with `setTimeout` stubbed to a no-op to freeze animations. `eval` the
  deck source and the test code together so the deck's `const`s are in scope, then walk every
  `FLOW` stop and sub-step on both paths, call `openPanel` on every node, run each example, and
  assert that every `ACT_STEPS` panel id, `proxyItems.nodeId`, `CONNS` endpoint, and flow slide id
  resolves. That catches essentially every breakage this file is prone to.
- Commit messages are short imperative summaries of the deck change (e.g. "Add in-deck fullscreen
  toggle (button + F key)").
