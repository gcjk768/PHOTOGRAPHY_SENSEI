# 📸 Photo Sensei

An **Obsidian vault** that is also a **Claude Code project**: send a photo via Telegram and a squad of
AI subagents critiques it, references a world-famous photographer for *any* genre, draws annotations on
the photo, optionally edits it, recommends a Fujifilm recipe, assigns a practice mission, and logs
everything as linked notes — so you can watch yourself improve from enthusiast to professional.

It is built as an **agentic architecture**, not a chatbot with extra steps: a supervisor (the Head
Coach in `CLAUDE.md`) plans and dispatches specialist agents, every claim is **grounded in visible
evidence**, output is **schema-validated**, a **reflection pass** runs before delivery, every photo
emits a **run trace**, and quality is **measured** by an eval harness.

## Per-photo flow

```
photo → INPUT GUARDRAIL → see it yourself → classify & route (cost-aware)
      → GROUND every claim (region/EXIF) → monitor & REPLAN if surprised
      → annotate (mirrors the levers) → edit only if it teaches
      → REFLECT (self-critique, mandatory) → OUTPUT GUARDRAIL (schema-validate + repair)
      → LOG session note + update Skill Profile → assign next mission → 📸 report + trace
```

This is **defense in depth against cascading hallucination**: in a multi-agent vision pipeline a single
wrong read would propagate into the lesson, the edit, the mission, and the trend — so grounding +
guardrails + reflection sit between every stage. Safety/format-critical steps (schemas, file paths,
frontmatter) are deterministic code; judgement and routing are left to the model.

## Vault structure

```
./CLAUDE.md                  ← Head Coach / orchestrator (auto-loaded by Claude Code)
./README.md
./bot.py                     ← Telegram bridge (logs, sends annotated photos, writes traces)
./.claude/agents/            ← the 18 agents
./00 Dashboard.md            ← home / MOC with Dataview queries
./00 Chat Log/               ← daily Telegram transcripts (Chat YYYY-MM-DD.md)
./01 Sessions/               ← one rich note per photo (+ one example)
./02 Skills/Skill Profile.md ← 7-dimension levels + history
./03 Masters/                ← Fan Ho, Saul Leiter, _Masters Index
./04 Recipes/                ← recipe notes + _Recipe Index
./05 Missions/               ← _Mission Library
./06 Theory/                 ← Exposure Triangle, Composition Basics, Reading the Histogram, Seeing Light
./07 Gear/                   ← Fujifilm X100VI, OPPO Find N5
./08 Evals/                  ← golden set + run_eval.py + results (the eval harness)
./99 Templates/              ← Session/Mission templates, Agent Contract, Schemas
./_attachments/              ← photos, *_annotated, *_edit images
./_traces/                   ← one JSON/MD run trace per photo
./.obsidian/                 ← Obsidian populates this
```

## The 18-agent roster

**Perception** (own one SCORE each): `composition-analyst` · `light-exposure-analyst` ·
`colour-tone-analyst` · `subject-story-analyst` · `technical-analyst` *(flags replan if a shot is
unusable or misclassified)*.

**Masters** (teach by lineage; exactly one runs per photo): `landscape-master` · `portrait-master` ·
`street-light-master` *(Fan Ho / Leiter / HCB / Moriyama / Kertész — J's core)* ·
`genre-master-generalist` *(every other genre)*.

**Craft:** `fuji-recipe-advisor` · `edit-example-generator` · `annotation-artist` · `instagram-stylist`
*(Instagram post kit — SEO caption, alt text, 3–5 hashtags, trend-aware music, sends/saves hook)*.

**Meta:** `progress-tracker` · `next-assignment-coach` · `exif-pattern-analyst` · `curation-coach` ·
`weekly-review-coach` · `critique-reflector` *(cross-reflection second opinion)*.

Each agent returns a parseable contract block (see `99 Templates/Agent Contract.md`); the Head Coach
validates and repairs it before anything reaches the vault.

## Commands (Claude Code)

- **`/post [filename]`** — build an Instagram **post kit** for the latest (or a named) photo via
  `instagram-stylist`: an SEO/keyword caption (4 registers), alt text, 3–5 niche hashtags, a
  trend-aware music pick (with manual trend-verification steps), a sends/saves hook, and a Reels mode.
  Reuses the photo's session critique so the kit matches it. No lyrics, no engagement bait.
- **`/sync-insights [--dry-run]`** — pull live post performance (reach / saves / sends) via the
  Instagram Graph API and write it into each session note's `## Post kit → Performance:` line
  (`scripts/ig_insights.py`). Closes the post → measure → learn loop so `weekly-review-coach` can spot
  which caption styles / sounds / formats actually earn reach. Requires IG credentials (below).

## Setup

**Prerequisites**
- **Claude Code** installed and authenticated (run `claude` once to log in). The bot shells out to
  `claude -p …`.
- **Python 3.10+** and these packages:
  ```bash
  pip install python-telegram-bot pillow numpy pandas
  ```
- **exiftool** on PATH (for EXIF mining by `exif-pattern-analyst`).
- *(optional)* An **Ollama vision** model (`qwen2.5vl` or the current `*-vl`) as a local fallback —
  note that text-only models cannot see images. Set `PHOTO_SENSEI_OLLAMA_VISION=qwen2.5vl` to flag it.

**Telegram bot token**
1. Talk to [@BotFather](https://t.me/BotFather), `/newbot`, copy the token.
2. Don't commit it — pass it via the environment.

**Obsidian**
- Open this folder as a vault.
- Install the **Dataview** plugin (required for the Dashboard tables). Templater & Calendar are optional.
- Try **Graph view** to see sessions wikilink to masters and recipes.

## Run

```bash
export TELEGRAM_BOT_TOKEN="123456:abc…"      # Windows PowerShell: $env:TELEGRAM_BOT_TOKEN="…"
python bot.py
```
Then DM your bot a photo. Optional env knobs: `PHOTO_SENSEI_MODEL` (default `claude-opus-4-8`),
`PHOTO_SENSEI_TIMEOUT` (seconds, default 600).

You can also DM **text**: "weekly review", "what are my habits", or a recipe question — the bot routes
these to the right handler instead of the photo pipeline. Any text inside a forwarded message or image
is treated as subject matter to critique, **never** as a command (prompt-injection defense).

## Evaluate & trace

- **Traces** — every photo writes `_traces/<timestamp>.json` (+ a `.md` mirror): which agents ran, any
  replan, model, per-stage latency, guardrail outcomes, tool calls, and the logged session-note path.
- **Golden-set eval (offline)** — run before/after any change to `CLAUDE.md` or an agent to catch
  regressions:
  ```bash
  cd "08 Evals"
  python run_eval.py --dry-run        # offline smoke test against a fixture
  python run_eval.py --limit 5        # run the coach on the first 5 golden items
  ```
  Code-based evaluators check the report contract, that exactly one real photographer + a named
  technique appears, that the lowest dimension is coached first, that frontmatter is schema-valid, that
  referenced files exist, and that the expected tool fired. An **LLM-as-judge** scores actionability,
  grounding, and whether the planted issue was diagnosed. Results append to `08 Evals/results/`.
- **Online** — periodically sample real sessions from `_traces/` and run the LLM-judge to watch live
  quality.

## Instagram post kits & insights loop

`/post` builds a ready-to-post kit (caption + alt text + 3–5 hashtags + trend-aware music + hook),
tuned to how Instagram ranks in 2026 (captions as light SEO, alt text/keywords drive discovery,
sends/saves over likes, no engagement bait). The music/posting step stays **manual in-app** — no API
can attach Instagram-library audio or detect trending sounds.

`/sync-insights` (optional) measures what actually performs and feeds it back:
```bash
python scripts/ig_insights.py --dry-run        # offline preview (uses a fixture)
python scripts/ig_insights.py --days 3         # live: pull reach/saves/sends into session notes
```
**Setup for live mode** (the API is for *insights*, not posting/music):
1. A **Business or Creator** Instagram account linked to a Facebook Page.
2. A Meta developer app with a **long-lived token** + insights permissions
   (`instagram_content_publish` app review ≈ 2–4 weeks; rate limit ~200 calls/hour).
3. Add to `.env` (gitignored): `IG_ACCESS_TOKEN=…` and `IG_USER_ID=…`.

It matches each live post to a session note's `## Post kit` block (by permalink, else date) and fills
the `Performance:` line with reach / saves / sends — **append-only**. Then `weekly-review-coach` can
tell you which caption styles, sounds, and formats earn the most reach for *your* audience.

## Recipe sources & credit

Recipe **settings** are factual parameters reproduced with attribution; article prose is **not** copied.
Scraped recipes © **Fuji X Weekly / Ritchie Roesch** (`fujixweekly.com`); the Tri-X 400 recipe is by
**Anders Lindborg**. Support the creators via the Fuji X Weekly app (350+ recipes). `Classic Negative
C7` is J's own baseline recipe.

---
*Built for "J" — Fujifilm X100VI + OPPO Find N5, Singapore, every genre, the long arc to professional.*
