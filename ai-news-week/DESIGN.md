# DESIGN.md — AI News Weekly (9:16 TikTok)

## Style Prompt

Bold, fun, modern editorial — Monocle energy but playful, not austere. Full-bleed, edge-to-edge, no floating cards. Each scene is one news story rendered like a magazine page: giant ghost rank number, editorial headline mixing bold sans with a single italic accent phrase, prominent metric, colored callout strip with plain-English takeaway. Vertical 1080×1920. Narrated, with karaoke captions.

## Palette

- `#F5EDE0` Cream — light paper background
- `#1A1A1A` Ink — primary text / dark background
- `#FF4A1C` Vermillion — primary accent (cover, bold highlights)
- `#2B4BFF` Electric — secondary accent
- `#FFC93C` Saffron — accent for dark backgrounds ONLY
- `#D4A5E8` Mauve — accent for dark backgrounds ONLY
- `#0F4D3A` Forest — deep accent (sparing)

**Contrast rules (non-negotiable):**
- Cream bg → ink text. Accents: vermillion, electric, forest.
- Ink bg → cream text. Accents: saffron, mauve, vermillion.
- Electric bg → cream text. Accent: saffron ONLY (never vermillion — they vibrate).
- Vermillion bg → cream text. Accents: ink and saffron only.
- Decoratives: 0.08–0.15 opacity. Body text: min 0.82 opacity.

## Typography

- **Space Grotesk 700** — display, headlines, metrics, brand
- **Inter Tight 200 italic** — one italic phrase inside each headline (accent color)
- **JetBrains Mono** — metadata, tags, page numbers, captions, UI labels

Headlines clamp(56px, 9vw, 130px). Metrics clamp(140px, 20vw, 280px). Body clamp(22px, 2.4vw, 34px).

Every headline: bold sans + italic phrase in accent. No two headlines identical in structure.

## Motion

- Entrance via `gsap.from()` only. Durations 0.4–0.7s. Vary eases per scene — at least 3 of: `power3.out`, `expo.out`, `power2.out`, `back.out(1.6)`.
- Transitions between scenes = editorial push slide + crossfade (0.5s, `power2.inOut`). The transition IS the exit. No per-element exit tweens except outro.
- Ghost rank number: slow drift (y±20, opacity 0.08↔0.12) over scene duration.
- Metric count-up via GSAP tween + onUpdate.
- Decorative circles: breathing scale 1.0↔1.05, ~6s cycle, `sine.inOut`.

## What NOT to Do

- No gradient text, no neon glow, no cyan-on-dark.
- No floating boxed cards — full-bleed padded containers only.
- No `#000` / `#fff` — use `#1A1A1A` / `#F5EDE0`.
- No identical card grids.
- No `<br>` in body copy — let text wrap via `max-width`.
- No exit animations on scenes 1–6 (transitions handle the handoff).
