# AI News Weekly — 9:16 TikTok video generator

A 60-second editorial-magazine-style vertical video summarising the week's top 5 AI stories. Voice-narrated in British English (ElevenLabs), karaoke captions synced to whisper transcripts, per-scene colored wipe transitions, drifting halftone backgrounds, Space Grotesk + Inter Tight + JetBrains Mono typography.

Built with [HyperFrames](https://hyperframes.heygen.com) — HTML + GSAP timelines compiled to MP4.

![format](https://img.shields.io/badge/format-1080%C3%971920-orange) ![duration](https://img.shields.io/badge/duration-60s-orange) ![codec](https://img.shields.io/badge/codec-H.264-orange)

---

## What's inside

```
ai-news-week/
├── index.html                root composition (scene routing + audio + captions)
├── DESIGN.md                 palette / typography / motion spec
├── WORKFLOW.md               full reproduction guide, 20 steps
├── compositions/
│   ├── cover.html
│   ├── story-1.html ... story-5.html
│   ├── outro.html
│   └── captions.html         word-level karaoke captions
├── scripts/                  narration source (per-scene .txt files)
├── media/
│   ├── el/                   per-scene ElevenLabs mp3s
│   └── voice.mp3             stitched 60-s voiceover
├── transcript.json           whisper word-level transcript
└── renders/
    └── ai-news-week.mp4      final output
```

---

## Quick start

```bash
# 1. Clone
git clone https://github.com/hashmil/ai-news-weekly.git
cd ai-news-weekly/ai-news-week

# 2. Add your ElevenLabs key
echo "ELEVENLABS_API_KEY=sk_your_key_here" > .env
chmod 600 .env

# 3. Lint + render
npx hyperframes lint
npx hyperframes render --fps 30 --quality standard --output renders/ai-news-week.mp4
```

Output lands in `ai-news-week/renders/ai-news-week.mp4`. Upload to TikTok.

### Preview in browser (no render)

```bash
npx hyperframes preview
```

---

## Rebuilding for a new week

Full 20-step recipe in [`ai-news-week/WORKFLOW.md`](ai-news-week/WORKFLOW.md). Summary:

1. **Source news** — `firecrawl search "top AI news this week" --scrape --limit 5` — pick 5 with strong metrics
2. **Write scripts** — one plain-English script per scene in `scripts/s1.txt` – `s7.txt` (~22–28 words per story)
3. **Generate voice** — one ElevenLabs request per scene via `curl`, voice `NYC9WEgkq1u4jiqBseQ9` (Russell — Dramatic British TV)
4. **Stitch audio** — `ffmpeg` with `adelay` offsets, `amix`, `apad` to exact 60 s (apply `atempo=1.10` to clips overshooting 10 s)
5. **Transcribe** — `npx hyperframes transcribe media/voice.mp3 --model small.en`
6. **Group captions** — Python: max 4 words per group, break on punctuation / 0.35 s gaps, assign saffron highlight per proper-noun priority list, inline into `captions.html`
7. **Update compositions** — swap headline / metric / callout / source in each `story-N.html` (keep palette, typography, rhythm CONSTANT)
8. **Lint + render** — `npx hyperframes lint && npx hyperframes render`
9. **Visually verify** — `ffmpeg` extract frames at each scene midpoint + each boundary, open and eyeball

---

## Design system

| Token | Hex | Usage |
|-------|-----|-------|
| Cream | `#F5EDE0` | light paper bg / primary text on dark |
| Ink | `#1A1A1A` | dark bg / primary text on light |
| Vermillion | `#FF4A1C` | cover bg, accent on light bgs |
| Electric | `#2B4BFF` | secondary accent |
| Saffron | `#FFC93C` | accent for dark bgs ONLY |
| Mauve | `#D4A5E8` | accent for dark bgs ONLY |
| Forest | `#0F4D3A` | deep accent (sparing) |

**Typography pairing** (one expressive + one delicate + one utility):

- **Space Grotesk 700** — all display, headlines, metrics, brand
- **Inter Tight 200 italic** — one italic phrase inside each headline (accent color)
- **JetBrains Mono** — metadata, tags, captions, UI labels

Full contrast rules + scene bg rhythm in [`ai-news-week/DESIGN.md`](ai-news-week/DESIGN.md).

---

## Scene structure

| # | Scene | Duration | Bg | Wipe color | Metric |
|---|-------|----------|-----|-----------|--------|
| 1 | Cover | 4 s | vermillion | ink panel slides off top | — |
| 2 | Story 1 — Opus 4.7 | 10 s | cream | electric blue from right | count-up `4.7` |
| 3 | Story 2 — Codex computer-use | 10 s | ink | saffron from left | SEE · MOVE · CLICK |
| 4 | Story 3 — Visa AI wallet | 10 s | electric | ink from right | AUTONOMOUS |
| 5 | Story 4 — 3.5 GW compute pact | 10 s | cream | vermillion from left | count-up `3.5 GW` |
| 6 | Story 5 — Gemini 3D answers | 10 s | ink | vermillion from right | 3D · INTERACTIVE |
| 7 | Outro | 6 s | ink | mauve from left | fade-to-black |

Every scene ships with:
- Giant ghost rank number (0.10–0.13 opacity)
- Category pill tag
- Editorial headline mixing Space Grotesk bold with one Inter Tight italic phrase in accent
- Source path
- Metric (number with count-up, or word list)
- "WHAT IT MEANS" callout with left accent stripe
- 5 page dots + `0N ÷ 05` counter

---

## Caption style

Word-level karaoke, JetBrains Mono 800, cream with 4 px ink stroke + soft shadow for readability on any scene bg. Each word sits dim (`opacity: 0.42`) until its exact timestamp, then pops to bright (scale 1.08 → 1 with `back.out(3)`). Keyword of the group is rendered in saffron.

Priority list for keyword selection (specific proper nouns > generic words):

```
Opus 4.7, Opus, Codex, 3.5, Anthropic, OpenAI, Visa, Gemini, Google, 3D,
wallet, autonomously, cursor, click, benchmarks, gigawatts, country,
poke, visual, behalf, operator, stories, minute, Follow, matter, AI, five
```

---

## Known pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| Scene N text visible behind scene N+1 | framework doesn't auto-hide expired sub-comp iframes | `tl.to(root, opacity: 0) + tl.set(visibility: hidden)` at each scene end |
| Black flash at scene boundary | no incoming wipe | Each scene has a full-screen colored `.sX-wipe` div at z-999 that slides off over 0.52 s |
| Cover title wraps mid-word | char-split + non-breaking spaces | `data-split-words` (word level), font 180 px, flex baseline row for "in A.I." |
| Caption overlaps scene callout | band too high | `cap-container { bottom: 300px }` — between body content (~y 1470) and foot (~y 1770) |
| TTS clip overshoots slot | British voice reads slower than US | `ffmpeg` `atempo=1.10` on the long clip (never > 1.15 — starts sounding compressed) |
| Decorative line crosses body text | `top: 50%` | Move to header safe zone `top: 232–260 px` |

Full table in [`ai-news-week/WORKFLOW.md`](ai-news-week/WORKFLOW.md).

---

## Requirements

- Node 18+ (`npx` runs hyperframes CLI)
- Python 3.8+ (caption grouping script)
- `ffmpeg` + `ffprobe` (audio stitching, frame extraction)
- ElevenLabs account + API key (TTS)
- [firecrawl](https://firecrawl.dev) CLI (web search for news)

---

## Security

- `.env` is gitignored — API keys never committed
- No secrets in composition HTML / `WORKFLOW.md` (only placeholders like `sk_xxx`)
- Voice id (`NYC9WEgkq1u4jiqBseQ9`) is public ElevenLabs shared-voice ID, not a secret

If you fork, add your own key to `.env` locally. Do not commit it.

---

## Credits

- **Editorial visual brief** — [gist by @hashmil](https://gist.github.com/hashmil/1ff5c5a36b5665b470496b01906ac798)
- **HyperFrames** — HTML → MP4 framework by HeyGen
- **ElevenLabs** — `eleven_multilingual_v2` TTS, voice "Russell — Dramatic British TV"
- **Whisper** — word-level transcription (`small.en`)
- **Firecrawl** — web search + scrape

---

## License

MIT.
