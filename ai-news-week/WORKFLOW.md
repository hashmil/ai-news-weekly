# AI News Weekly — 9:16 TikTok Video Workflow

End-to-end guide to reproduce a 60-second editorial-magazine-style vertical video summarizing the top 5 AI news stories of the week. Voice narration, karaoke captions, per-scene color wipes, halftone backgrounds, editorial typography.

**Give this file to an LLM along with a target topic or week's news and it should be able to rebuild the video from scratch with new content.**

---

## Output spec

- **Format:** MP4, H.264 + AAC
- **Resolution:** 1080 × 1920 (9:16 vertical)
- **Frame rate:** 30 fps
- **Duration:** 60 s exact
- **Scenes:** cover (4 s) + 5 stories (10 s each) + outro (6 s)
- **Voice:** ElevenLabs TTS, British narrator (Russell — Dramatic British TV, voice id `NYC9WEgkq1u4jiqBseQ9`)
- **Captions:** word-level karaoke synced to whisper transcript, saffron keyword highlight per group

---

## Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| Node / npx | Run `hyperframes` CLI | Node 18+ |
| `hyperframes` | Compile + render composition | `npx hyperframes` (no install needed) |
| Python 3 | Caption grouping script | system python |
| `ffmpeg` / `ffprobe` | Stitch audio + verify MP4 + extract frames | `brew install ffmpeg` |
| `firecrawl` CLI | Web search for news | already installed; auth via `firecrawl --status` |
| ElevenLabs API key | TTS | env var `ELEVENLABS_API_KEY` in `.env` |
| `curl`, `jq` | API calls + JSON parsing | standard |

---

## Directory layout (target)

```
ai-news-week/
├── index.html                       root composition
├── DESIGN.md                        palette/typography/motion spec
├── WORKFLOW.md                      this file
├── .env                             ELEVENLABS_API_KEY=... (chmod 600, gitignored)
├── .gitignore
├── hyperframes.json                 project config (scaffolded)
├── meta.json                        project metadata (scaffolded)
├── CLAUDE.md, AGENTS.md             agent guidance (scaffolded)
├── compositions/
│   ├── cover.html
│   ├── story-1.html ... story-5.html
│   ├── outro.html
│   └── captions.html
├── scripts/
│   ├── narration.txt                master script (7 sections)
│   └── s1.txt ... s7.txt            per-scene prompts for TTS
├── media/
│   ├── el/s1.mp3 ... s7.mp3         raw ElevenLabs outputs
│   └── voice.mp3                    stitched 60-s voiceover
├── transcript.json                  whisper word-level timings
└── renders/
    └── ai-news-week.mp4             final output
```

---

## Step 1 — Find the top 5 news stories

Goal: identify 5 high-impact, recent, punchy AI stories from the past ~7 days.

```bash
firecrawl search "top AI news this week <MONTH YEAR> major announcements OpenAI Anthropic Google" \
  --scrape --limit 5 \
  -o /tmp/ai-news-search.json --json
jq -r '.data.web[] | "\(.title)\n  \(.url)\n  \(.description)\n"' /tmp/ai-news-search.json
```

Then pull headlines from the first roundup-style article:

```bash
jq -r '.data.web[0].markdown' /tmp/ai-news-search.json > /tmp/roundup.md
grep -oE '^\*\*\[[^]]+\]' /tmp/roundup.md | sed 's/^\*\*\[//' | sed 's/\]$//'
```

**Story selection criteria** (keep 5):

1. Released within the past 7 days (check dates)
2. High punch — dramatic numbers, new capability, policy shift
3. Diverse categories — cybersecurity, models, agents, infrastructure, consumer
4. Recognizable actors — OpenAI, Anthropic, Google, Visa, Meta preferred
5. Has at least one **metric** you can render big (version number, dollar amount, gigawatts, percentage)

**Verify each story with a follow-up firecrawl search** (`firecrawl search "<story-title> <source>"`) so the source URL and metric are real. Do not hallucinate.

Example final picks for Apr 17 2026:

| # | Title | Metric | Source | Category |
|---|---|---|---|---|
| 1 | Anthropic ships Claude Opus 4.7 | **4.7** | axios.com | models |
| 2 | OpenAI gives Codex computer-use | SEE · MOVE · CLICK | techcrunch.com | agents |
| 3 | Visa launches AI agent payments | AUTONOMOUS | visa.com | payments |
| 4 | Broadcom/Google 3.5 GW Anthropic deal | **3.5 GW** | cnbc.com | infrastructure |
| 5 | Gemini adds interactive 3D answers | 3D · INTERACTIVE | blog.google | consumer |

Swap any as user preferences or breaking news dictates.

---

## Step 2 — Scaffold the project

```bash
cd /path/to/parent
npx hyperframes init ai-news-week
cd ai-news-week
mkdir -p media media/el scripts compositions
echo "ELEVENLABS_API_KEY=sk_xxx" > .env && chmod 600 .env
printf ".env\n.env.*\n.hyperframes/\nnode_modules/\n" > .gitignore
```

---

## Step 3 — Write the narration

Write 7 scene scripts to `scripts/s1.txt` through `scripts/s7.txt`. Rules:

- **Plain English** (generalist audience), no niche jargon
- **Cover (s1):** ≤10 words, hook
- **Stories (s2–s6):** ~22–28 words, 1–3 sentences each; lead with actor + verb; end with implication
- **Outro (s7):** ≤10 words, CTA
- **Prefer spelled-out numbers** ("three point five" not "3.5") so TTS reads cleanly, but keep "A.I." with periods for pronunciation
- **Test timing:** each story must fit ~10 s with the chosen voice. Russell British narrator reads slower than US voices — aim for 8.5–9 s per 22 words. Long scripts can be sped up later with ffmpeg `atempo`.

Also concat into a master `scripts/narration.txt` with labeled sections for human review.

---

## Step 4 — Generate voice audio (ElevenLabs)

One `curl` per scene. Load key from `.env`.

```bash
source .env
VOICE_ID="NYC9WEgkq1u4jiqBseQ9"
for n in 1 2 3 4 5 6 7; do
  TEXT=$(cat scripts/s${n}.txt)
  curl -s -X POST "https://api.elevenlabs.io/v1/text-to-speech/${VOICE_ID}?output_format=mp3_44100_128" \
    -H "xi-api-key: $ELEVENLABS_API_KEY" \
    -H "Content-Type: application/json" \
    -o media/el/s${n}.mp3 \
    -d "$(python3 -c "import json,sys; print(json.dumps({
      'text': sys.argv[1],
      'model_id': 'eleven_multilingual_v2',
      'voice_settings': {
        'stability': 0.5,
        'similarity_boost': 0.75,
        'style': 0.4,
        'use_speaker_boost': True
      }
    }))" "$TEXT")"
done

# Verify durations
for n in 1 2 3 4 5 6 7; do
  D=$(ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 media/el/s${n}.mp3)
  echo "s${n}: ${D}s"
done
```

**Timing budget:**

| Scene | Slot | Max length |
|-------|------|-----------|
| s1 (cover) | 0–4 s | ~3.7 s |
| s2–s6 (stories) | 10 s each | ~9.5 s |
| s7 (outro) | 6 s | ~4.5 s |

**If any clip overshoots its slot**, regenerate with a faster speed or apply `atempo` during stitching (step 5). Avoid `atempo > 1.15` — voice starts to sound compressed.

---

## Step 5 — Stitch voice track

Combine 7 clips into one 60-second MP3 with precise offsets. Apply `atempo=1.10` to any clip that overshoots its slot.

```bash
# Example: s3 + s5 need speed-up
ffmpeg -y \
  -i media/el/s1.mp3 -i media/el/s2.mp3 -i media/el/s3.mp3 \
  -i media/el/s4.mp3 -i media/el/s5.mp3 -i media/el/s6.mp3 -i media/el/s7.mp3 \
  -filter_complex "\
    [2:a]atempo=1.10[s3f]; \
    [4:a]atempo=1.10[s5f]; \
    [0:a]adelay=0|0[a1]; \
    [1:a]adelay=4000|4000[a2]; \
    [s3f]adelay=14000|14000[a3]; \
    [3:a]adelay=24000|24000[a4]; \
    [s5f]adelay=34000|34000[a5]; \
    [5:a]adelay=44000|44000[a6]; \
    [6:a]adelay=54000|54000[a7]; \
    [a1][a2][a3][a4][a5][a6][a7]amix=inputs=7:duration=longest:normalize=0,apad=whole_dur=60[out]" \
  -map "[out]" -t 60 -c:a libmp3lame -b:a 192k media/voice.mp3

ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 media/voice.mp3
# Must print 60.000000
```

- `adelay=N|N` — left + right channel delay in ms
- `amix=normalize=0` — preserve levels (avoids quieter output)
- `apad=whole_dur=60` — pad to exact 60 s

---

## Step 6 — Transcribe for captions

```bash
npx hyperframes transcribe media/voice.mp3 --model small.en
# Writes transcript.json with word-level {text, start, end}
```

`small.en` is fine for English. For multilingual: `--model small` + `--language <code>`.

---

## Step 7 — Generate caption groups

Python grouping rules:

- **Max 4 words per group**, break on sentence-ending punctuation OR gap > 0.35 s
- **Merge orphan 1-word groups** into previous if close (< 0.5 s)
- **Drop whisper artifacts** (final single-word "you" type errors)
- **Manual corrections**: fix known mishears (e.g. whisper hears "Codex" as "codecs" — replace)
- **Trim overlapping group ends** to avoid simultaneous-visible
- **Assign highlight keyword per group**, priority list (proper nouns first): `Opus 4.7, Opus, Codex, 3.5, Anthropic, OpenAI, Visa, Gemini, Google, 3D, wallet, autonomously, cursor, click, benchmarks, gigawatts, country, poke, visual, behalf, operator, stories, minute, Follow, matter, AI, five`

Output a JS array of `{ start, end, hl, words: [{t, s, e}] }` inline into `compositions/captions.html`.

---

## Step 8 — Visual identity (DESIGN.md)

Lock colors, type, motion BEFORE writing compositions. Write to `DESIGN.md`:

```markdown
## Palette
- Cream   #F5EDE0   light paper bg
- Ink     #1A1A1A   dark bg / primary text
- Vermillion #FF4A1C primary accent (cover)
- Electric #2B4BFF  secondary accent
- Saffron  #FFC93C  accent for dark bgs ONLY
- Mauve    #D4A5E8  accent for dark bgs ONLY
- Forest   #0F4D3A  deep accent (sparing)

## Contrast rules (non-negotiable)
- Cream bg → ink text; accents vermillion/electric/forest
- Ink bg → cream text; accents saffron/mauve/vermillion
- Electric bg → cream text; accent saffron ONLY (never vermillion — they vibrate)
- Vermillion bg → cream text; accents ink and saffron only

## Typography
- Space Grotesk 700 — display, headlines, metrics, brand
- Inter Tight 200 italic — one italic phrase per headline (accent color)
- JetBrains Mono — metadata, tags, page numbers, captions, UI labels

## Scene bg rhythm (no two identical in a row)
cover:vermillion → s1:cream → s2:ink → s3:electric → s4:cream → s5:ink → outro:ink
```

---

## Step 9 — Scene timing + track layout

`index.html` hosts 9 clips on these tracks:

| Clip | Track | Start | Duration |
|------|-------|-------|----------|
| cover | 1 | 0 | 4 |
| story-1 | 1 | 4 | 10 |
| story-2 | 1 | 14 | 10 |
| story-3 | 1 | 24 | 10 |
| story-4 | 1 | 34 | 10 |
| story-5 | 1 | 44 | 10 |
| outro | 1 | 54 | 6 |
| captions | 3 | 0 | 60 |
| voice (audio) | 4 | 0 | 60 |

**Strict back-to-back on track 1** — no overlap (the lint catches that). Captions + audio on separate tracks.

**Body bg = `#1A1A1A`** in `index.html` — safety net if a frame ever leaks through.

---

## Step 10 — Sub-composition architecture

Every scene is a separate file under `compositions/` wrapped in a `<template>`, with one `[data-composition-id]` div inside.

**Critical attributes on the root div:**

```html
<div data-composition-id="story-1"
     data-width="1080" data-height="1920"
     data-start="0" data-duration="10">
```

`data-start` / `data-duration` on the sub-comp root tell the compositor the scene's internal timeline length.

**Non-negotiable rules** (from hyperframes skill):

1. Timeline created paused: `gsap.timeline({ paused: true })`
2. Registered: `window.__timelines["<comp-id>"] = tl`
3. Only visual properties animated (no `visibility`/`display` tweens)
4. No `Math.random()` / `Date.now()` / `repeat: -1` anywhere
5. No `<br>` in body copy — let text wrap via `max-width`
6. Animate a wrapper, not video dimensions

---

## Step 11 — Per-scene layout (stories)

Vertical padded container, full-bleed, editorial spacing. Pattern reused across story-1..5:

```
┌─── .sX-root (padding: 140 70 120) ──────┐
│  .sX-meta-top   ENTRY 0X OF 05 · source │  ← JetBrains Mono 20px, 2px bottom border
│  ─────────────────────────────────────  │
│                                         │
│  [bg halftone + bg accent + ghost #]    │  ← z-index 0, ambient
│                                         │
│  .sX-tag        CATEGORY · SUBCATEGORY  │  ← pill, 1.5px border, JB Mono
│  .sX-headline                           │  ← Space Grotesk 700 / Inter Tight italic
│  .sX-path       source.url              │
│  .sX-metric     BIG NUMBER or WORDS     │  ← accent color, Space Grotesk 700
│  .sX-metric-tag ● small context         │
│  .sX-callout    [stripe] WHAT IT MEANS  │  ← accent stripe left, tinted bg
│                 italic 1-sentence note  │
│                                         │
│  .sX-foot       ● · · · ·       0X/05   │  ← page dots + counter
└─────────────────────────────────────────┘
```

**Variation between scenes** — change per-scene so no two look identical:

- Structure of the italic phrase inside the headline (prefix / middle / suffix)
- Metric type (number with count-up / word list / single keyword)
- Which color is accent (vermillion / saffron / mauve / electric)
- Extra decorative (streak, circle, ring, dot) in corners / header zone

Cover + outro use a different layout (full spread, no callout).

---

## Step 12 — Scene entrance wipe (color reveal)

Every story + outro gets a full-screen colored `<div class="sX-wipe">` as FIRST child inside the root. CSS:

```css
.sX-wipe {
  position: absolute;
  inset: 0;
  background: #CONTRAST_COLOR;
  z-index: 999;
  transform-origin: right center;  /* or left, alternates per scene */
}
```

Timeline:

```js
tl.to('.sX-wipe', { scaleX: 0, duration: 0.52, ease: 'power4.inOut' }, 0);
tl.set('.sX-wipe', { display: 'none' }, 0.52);
```

**Wipe colors — must contrast both outgoing AND incoming scene bg:**

| Scene | Bg | Wipe color | Origin |
|-------|----|-----------|--------|
| cover | vermillion | ink panel (slide off top, `scaleY: 0`) | top |
| story-1 | cream | electric blue `#2B4BFF` | right |
| story-2 | ink | saffron `#FFC93C` | left |
| story-3 | electric | ink `#1A1A1A` | right |
| story-4 | cream | vermillion `#FF4A1C` | left |
| story-5 | ink | vermillion `#FF4A1C` | right |
| outro | ink | mauve `#D4A5E8` | left |

Alternate origin side per scene so the wipe direction changes — reads as editorial "turning the page".

---

## Step 13 — Scene exit — prevent bleed

**This is critical.** The hyperframes framework does NOT hide expired sub-comp iframes by default; they render through the next scene. You WILL see text from scene N behind scene N+1 unless you force it.

**Fix:** add at the end of every sub-comp timeline (before the `__timelines` registration):

```js
// Fade root then deterministic hard kill
tl.to('[data-composition-id="story-1"]',
      { opacity: 0, duration: 0.18, ease: 'power2.in' }, 9.82);
tl.set('[data-composition-id="story-1"]',
      { opacity: 0, visibility: 'hidden' }, 10.0);
```

Do this for cover (at 3.82 / 4.0), all 5 stories (at 9.82 / 10.0), and captions (via the `tl.set(groupEl, { opacity: 0, visibility: 'hidden' }, group.end)` pattern described in caption rules).

The outro is the final scene — it already fades everything to black in the last 0.6 s, which doubles as its exit.

---

## Step 14 — Animation palette (amping up)

Apply these to every scene for a consistent editorial-but-lively feel. Keep ALL entrances via `gsap.from()` — never animate into opacity: 0 mid-scene.

**Letter/word splitting** (headlines + large metrics):

```js
document.querySelectorAll('[data-split]').forEach(el => {
  const txt = el.textContent;
  el.textContent = '';
  for (const ch of txt) {
    const s = document.createElement('span');
    s.className = 'chr';
    s.textContent = ch === ' ' ? '\u00A0' : ch;
    el.appendChild(s);
  }
});
// then: tl.from('.sX-headline .chr', { y: 70, rotation: 3, opacity: 0,
//                                        duration: 0.5, ease: 'back.out(2)',
//                                        stagger: 0.025 }, 0.55);
```

**Cover title** — use word-level split (`data-split-words`) not char-split so title wraps at word boundaries if needed, never mid-word.

**Number count-up:**

```js
const counter = { val: 0 };
tl.fromTo('#sX-num', { scale: 0.4, opacity: 0 },
          { scale: 1.1, opacity: 1, duration: 0.55, ease: 'back.out(2.4)' }, 1.5);
tl.to(counter, {
  val: 4.7,
  duration: 1.1,
  ease: 'power3.out',
  onUpdate: () => { document.getElementById('sX-num').textContent = counter.val.toFixed(1); }
}, 1.5);
tl.to('#sX-num', { scale: 1, duration: 0.4, ease: 'power2.out' }, 2.05);
```

**Varied eases per scene** — at least 3 different of: `power3.out`, `expo.out`, `power2.out`, `back.out(1.6..2.4)`.

**Callout stripe grows before text fills:**

```js
tl.fromTo('.sX-callout-stripe', { scaleY: 0 }, { scaleY: 1, duration: 0.45, ease: 'power3.out' }, 2.2);
tl.from('.sX-callout',      { x: -30, opacity: 0, duration: 0.5, ease: 'power3.out' }, 2.25);
tl.from('.sX-callout-label',{ y: 14, opacity: 0, duration: 0.35, ease: 'power2.out' }, 2.45);
tl.from('.sX-callout-body', { y: 18, opacity: 0, duration: 0.45, ease: 'power2.out' }, 2.55);
```

**Foot dots** stagger pop + active dot pulses continuously.

**Ghost rank number** (`01`–`05`) drifts slowly throughout scene + pulses scale subtly.

---

## Step 15 — Background pattern animations

Each scene gets two background layers at `z-index: 0`, behind everything. Both oversized (`inset: -60px` or more) so drift doesn't reveal edges.

**1. Halftone dots** (shared pattern, tinted per scene):

```css
.sX-bg-halftone {
  position: absolute;
  left: -60px; top: -60px; right: -60px; bottom: -60px;
  pointer-events: none;
  background-image: radial-gradient(circle,
    rgba(TINT,0.07-0.12) 2px, transparent 2.5px);
  background-size: 34px 34px;  /* 36 for dark bgs, 40 for vermillion */
  z-index: 0;
}
```

```js
tl.fromTo('.sX-bg-halftone', { opacity: 0 },
          { opacity: 1, duration: 0.8, ease: 'power2.out', overwrite: 'auto' }, 0.25);
tl.to('.sX-bg-halftone', { x: ±34, y: ±34, duration: 14, ease: 'none' }, 0);
```

Drift direction varies per scene.

**2. Per-scene accent pattern** (each scene unique):

| Scene | Pattern | Implementation |
|-------|---------|----------------|
| cover | concentric rings pulsing | large border-radius circle with shadow rings |
| story-1 | vertical vermillion columns drifting right | `repeating-linear-gradient(90deg, ...)`, tween `x: 90` |
| story-2 | diagonal saffron stripes | `repeating-linear-gradient(45deg, ...)`, tween `x: -54, y: -54` |
| story-3 | horizontal ink ticker bars scrolling up | `repeating-linear-gradient(0deg, ...)`, tween `y: -140` |
| story-4 | horizontal forest contour lines | `repeating-linear-gradient(0deg, ...)`, tween `y: 48` |
| story-5 | multi-ring orbit rotating | nested `border-radius: 50%` w/ `::before`/`::after`, tween `rotation: 360` |
| outro | counter-drifting mauve diagonals | `repeating-linear-gradient(-45deg, ...)`, tween `x: -60, y: 60` |

Keep alpha 0.06–0.12. These must read as atmosphere, never as content.

---

## Step 16 — Decorative element placement (avoid content overlap)

Horizontal stripes, small dots, rings go in the **header zone** (y 232–256 px), between the meta-top border (~y 220) and the body content (~y 420+). NEVER place decoratives at `top: 42%` or `top: 58%` — they will slice through body text.

Safe zones:
- Top band: `top: 232–260px` (after meta, before body)
- Top corner dots: `top: 220–300px` (right or left)
- Bottom-left or bottom-right large circles/rings: `bottom: -200 to -280px` (peek from corner)

---

## Step 17 — Captions sub-composition

`compositions/captions.html` — track 3, full 60 s, overlays every scene.

Position band (avoid scene content + foot):

```css
.cap-container { position: absolute; left: 0; right: 0; bottom: 300px; }
.cap-group {
  position: absolute; left: 50%; bottom: 0; transform: translateX(-50%);
  width: 920px; opacity: 0; visibility: hidden;
  font-family: 'JetBrains Mono'; font-weight: 800; font-size: 58px;
  line-height: 1.15; letter-spacing: -0.01em; color: #F5EDE0;
  text-transform: uppercase;
  display: flex; flex-wrap: wrap; justify-content: center; gap: 0 14px;
}
.cap-word {
  display: inline-block; opacity: 0.42; transform-origin: center;
  text-shadow:
    4px 4px 0 #1A1A1A, -2px -2px 0 #1A1A1A,
    2px -2px 0 #1A1A1A, -2px 2px 0 #1A1A1A,
    0 0 22px rgba(26,26,26,0.62);
}
.cap-word.hl { color: #FFC93C; }
```

Per-group timeline:

```js
GROUPS.forEach((g, i) => {
  const sel = '#cg-' + i;
  const showAt = Math.max(0, g.start - 0.06); // 60 ms early to hide whisper drift

  tl.set(sel, { visibility: 'visible' }, showAt);
  tl.fromTo(sel, { opacity: 0, y: 22, scale: 0.88 },
                 { opacity: 1, y: 0, scale: 1, duration: 0.18, ease: 'back.out(2.2)' },
                 showAt);

  // Per-word pop (dim 0.42 → bright 1.0 with small scale overshoot)
  g.words.forEach((w, wi) => {
    const wsel = '#cg-' + i + '-w-' + wi;
    const wStart = Math.max(showAt, w.s - 0.04);
    tl.fromTo(wsel, { opacity: 0.42, scale: 1 },
                    { opacity: 1, scale: 1.08, duration: 0.12, ease: 'back.out(3)' }, wStart);
    tl.to(wsel, { scale: 1, duration: 0.14, ease: 'power2.out' }, wStart + 0.12);
  });

  const outStart = Math.max(showAt + 0.18, g.end - 0.10);
  tl.to(sel, { opacity: 0, y: -12, scale: 0.98, duration: 0.10, ease: 'power2.in' }, outStart);
  tl.set(sel, { visibility: 'hidden', opacity: 0 }, g.end); // hard kill (required by lint)
});
```

`hl` words get `.hl` class → saffron color. Priority list in Step 7.

---

## Step 18 — Lint

```bash
npx hyperframes lint
```

Must be **0 errors**. Warnings that are acceptable:
- `overlapping_gsap_tweens` between different elements that share a CSS prefix (false positives on bg layer fromTo + drift `to`)

Fix real issues:
- `overlapping_clips_same_track` → either sequential timing or different tracks
- `audio_src_not_found` → stitch voice.mp3 first (Step 5)
- `root_composition_missing_data_start|data_duration` → add to sub-comp root div
- `gsap_css_transform_conflict` → replace CSS `transform: translateY/X(-50%)` with positional CSS (`top: calc(50% - h/2)`)

---

## Step 19 — Render

```bash
npx hyperframes render --fps 30 --quality standard --output renders/ai-news-week.mp4
```

Takes ~90 s on 16-core M-series. For draft iteration: `--quality draft` (faster, lower bitrate). Final share: `--quality high`.

Confirm output:

```bash
ffprobe -v error -show_entries format=duration,size:stream=width,height,codec_name -of default renders/ai-news-week.mp4
# duration=60.000000
# width=1080, height=1920, codec=h264
```

---

## Step 20 — Visual verification (MANDATORY before sharing)

Extract frames at key timestamps and visually inspect each — never trust lint alone.

```bash
mkdir -p /tmp/vcheck
# mid-scene frames
for t in 1.5 7 17 27 37 47 57; do
  ffmpeg -y -ss $t -i renders/ai-news-week.mp4 -frames:v 1 -vf "scale=720:1280" /tmp/vcheck/t${t}s.jpg 2>/dev/null
done
# boundary frames (wipe should cover here)
for t in 4.1 14.1 24.1 34.1 44.1 54.1; do
  ffmpeg -y -ss $t -i renders/ai-news-week.mp4 -frames:v 1 -vf "scale=540:960" /tmp/vcheck/boundary_${t}s.jpg 2>/dev/null
done
open /tmp/vcheck/
```

**Verify every frame:**

- [ ] Cover headline one line ("THIS WEEK"), "in A.I." correctly inlined
- [ ] No scene-N content leaking behind scene-(N+1) mid-scene
- [ ] Boundary frames show solid wipe color (≠ black)
- [ ] Captions visible, not overlapping content
- [ ] Decorative lines in top header zone, not across body
- [ ] Metric numbers readable, count-ups complete
- [ ] Audio in sync (scrub the actual video file)

If anything is broken, fix the composition and re-render. Don't claim done without this check.

---

## Customization for new content (reproducibility)

To rebuild for a new week's news:

1. **Re-run Step 1** — search for new top 5 stories for the target week
2. **Update `scripts/s2.txt` – `s6.txt`** with new scripts (keep s1 / s7 as template)
3. **Update each `compositions/story-N.html`:**
   - `.sN-meta-top` source domain
   - `.sN-tag` category
   - `.sN-headline` (swap bold/italic/bold2 spans — vary structure!)
   - `.sN-path` source url subpath
   - `.sN-metric` / `#sN-num` with count-up target
   - `.sN-callout-body` one-sentence generalist takeaway
4. **Regenerate voice** (Step 4) — re-run the curl loop
5. **Re-stitch** (Step 5) — check new durations, adjust `atempo` if needed
6. **Re-transcribe** (Step 6) — whisper on the new `voice.mp3`
7. **Re-generate caption groups** (Step 7) — re-run the Python script, update highlight keyword list for new proper nouns
8. **Re-inline captions** into `compositions/captions.html`
9. **Lint + render + visually verify** (Steps 18–20)

Keep palette, typography, scene rhythm, animation patterns CONSTANT across issues — that builds brand.

---

## Common pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| Scene N text visible behind scene N+1 | framework doesn't auto-hide iframes | Add `tl.to(root, opacity: 0) + tl.set(visibility: hidden)` at scene end (Step 13) |
| Black flash at scene boundary | scene wipe not covering + framework gap | Each scene has an entrance wipe (Step 12) with opaque bg |
| Cover title wraps mid-word ("THIS WEE / K") | char-split + non-breaking spaces in title | Use `data-split-words` (word-level), reduce font to 180 px |
| Caption overlaps scene callout | container bottom too high | Set `.cap-container { bottom: 300px }` — between body content (~y 1470) and foot (~y 1770) |
| TTS clip overshoots 10 s | British voice reads slow | Apply `atempo=1.10` in ffmpeg stitch (never > 1.15) |
| Decorative stripe crosses body text | `top: 50%` placement | Move to `top: 248px` (header safe zone) |
| Whisper mishears proper noun | small.en model limits | Post-process transcript: replace "codecs" → "Codex" before grouping |
| Pattern drift reveals edges | bg layer at exact screen size | Extend bg layer `inset: -60px` so drift stays within oversized canvas |
| Render takes forever / fails | missing kokoro-onnx (if using Kokoro TTS), or 502 from HF | Install `kokoro-onnx soundfile` via pip OR switch to ElevenLabs (we use EL) |

---

## Tools / credits

- Framework: **HyperFrames** (`@hyperframes`, heygen-com/hyperframes) — HTML+GSAP → MP4
- TTS: **ElevenLabs** `eleven_multilingual_v2`
- ASR: **Whisper small.en** via `hyperframes transcribe`
- News source: **Firecrawl CLI**
- Visual brief: the editorial-magazine prompt at https://gist.githubusercontent.com/hashmil/1ff5c5a36b5665b470496b01906ac798 — palette, typography, structure rules all derived from that

---

## Acceptance checklist

- [ ] `npx hyperframes lint` → 0 errors
- [ ] `ffprobe renders/ai-news-week.mp4` → 60.00 s duration, 1080×1920
- [ ] Frame at every scene mid-point is clean (no bleed)
- [ ] Frame at every scene boundary shows wipe color (no black)
- [ ] Audio ends cleanly before 60 s; no cut-off words
- [ ] Captions sync within ±100 ms of voice
- [ ] Every highlight keyword in captions is visually distinct
- [ ] Outro fades to black smoothly over last 0.6 s
- [ ] File size under 25 MB (TikTok upload limit headroom)
