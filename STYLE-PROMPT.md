# House style — portable prompt

Paste this into another Claude Code session to get work in the same style.

---

Build single-file HTML deliverables in this house style.

**Structure.** One self-contained `.html` file: inline `<style>`, inline `<script>`, no build
step, no dependencies except Google Fonts (Inter). Local assets only. Content is data, not
markup — define arrays/objects (slides, nodes, flow) and render from them; the `<body>` holds
only fixed containers. Layout on a fixed 1920×1080 canvas, centred and uniformly scaled to the
viewport (`min(vw/1920, vh/1080)`), with the page background filling the letterbox bars.

**Colour.** Brand palette: blue `#307FE2`, yellow `#FFB900`, red `#DA291C`, green `#3CB17E`,
purple `#9063CD`, teal `#2CCCD3`, silver `#A2B2C8`, grey `#75787B`. Everything else comes from
tokens — `--text-primary/secondary/dim`, `--border-subtle/default/strong`, `--surface-1/2`,
`--surface-glass-bg`, `--surface-card-bg`, `--shadow-card/lift`, `--ease-out-soft`. **Never
hardcode a colour in a rule body**: it won't flip with the theme and high-contrast mode can't
reach it. White text on a brand-coloured fill is the only exception. Light is the default (class
on `<html>`, restored pre-paint from localStorage); dark and a high-contrast projector mode are
token overrides. Verify contrast numerically — composite the real stack and compute WCAG ratios.
AA for body text; deliberately quiet elements may sit near 3:1 and get lifted in projector mode.

**Form.** Inter. Titles 23–36px, body 12.5–14px, labels 9–11px uppercase with 1.2–2.5px
letter-spacing. Cards, pills and chips with 8–20px radii, 1px token borders, glass surfaces over
the page ground. Prose caps its own measure (~840–900px) even in a wider container. Never reserve
space with fixed heights — equalise with grid/flex and pin footers with `margin-top:auto`. Icons
are inline SVG, 24×24, `stroke-width:1.75`, `currentColor`, Lucide-like. No emoji in content.

**Motion.** Animate `transform` and `opacity` only. Deterministic values — no `Math.random()`;
stagger with computed negative delays so a looping field is full on first paint. Every animation
needs a `prefers-reduced-motion` resting state.

**Voice.** Write about the subject, not about a third party — "the agent", "the application",
"the environment", never "your customer's data" or "the thing they ship". It must read as a
document someone opens alone: no coaching ("ask them…", "the line that lands here"), no sales
vocabulary (buyer, pitch, deal, qualify, consumption, on Monday). Declarative and concrete — name
real services, real numbers, real payloads; short sentences; no hype or stacked adjectives.
Credit competing and native platform capabilities accurately, then argue scope and ownership;
never build on a claim the other side can refute in one sentence. Attribute figures to their
source and say plainly when one is unverified. Define jargon in hover glossaries, not mid-sentence.

**Method.** Edit by scripted exact-string replacement that asserts the match count, so a stale
assumption fails loudly instead of silently mangling the file. Without a browser, verify anyway:
extract the `<script>`, run it in Node against a stubbed DOM, and walk every stop, panel and
animation entry point; validate SVG as XML and check geometry against the viewBox; confirm every
class used has a CSS rule. When changing a theme system, prove the baseline is untouched by
resolving the cascade per mode before and after and diffing.
