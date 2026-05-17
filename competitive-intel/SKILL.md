---
name: Competitive Intelligence Monitor
key: competitive-intel
description: |
  Run a competitive intelligence sweep across a configured list of competitor companies
  in digital pathology AI. Detect product launches, regulatory approvals, M&A, executive
  changes, partnerships, clinical trials, funding. Use this skill when the agent is
  triggered for scheduled or manual competitive monitoring. Focus parameters and source
  tier classifications are fetched from Notion at runtime, not hardcoded here.
version: 0.2.0
metadata:
  cadence: twice weekly (Wed + Fri 09:00 local)
  config_sources:
    focus_db: Notion "Skill Focus" (id from env NOTION_FOCUS_DB_ID)
    tiers_db: Notion "Source Tiers" (id from env NOTION_TIERS_DB_ID)
---

# Competitive Intelligence Monitor

## Mission

Surface signals from a competitor set, filtered by current strategic focus, scored by event type. Output a categorized digest. **Not a news aggregator** — a filter producing 2-5 actionable signals per run.

## Configuration sources (runtime)

Two things vary between runs and must be **fetched at runtime**, not embedded:

1. **`focus_areas`** — from Notion DB `$NOTION_FOCUS_DB_ID`
   Columns: `area` (text), `status` (active/penalize/watch), `boost` (number), `notes` (text)
   These drive scoring boosts and dampenings (see `scoring-rubric.md`).

2. **`source_tiers`** — from Notion DB `$NOTION_TIERS_DB_ID`
   Columns: `competitor` (text), `tier` (top/mid/watch/drop), `status` (active/dropped), `notes` (text)
   These drive how deeply each competitor is scanned (see `source-process.md`).

3. **`signals_output`** — write to Notion DB `$NOTION_SIGNALS_DB_ID`
   See "Output format" section for column schema.

If either input DB is unreachable → halt and report. Do not proceed with stale assumptions.

## Notion API access (critical — read before any Notion call)

The Paperclip-managed runtime has **Node.js** but **no `curl` / `jq`**. Use Node's built-in `fetch`.

**All Notion API calls must use the data-source API (2025-09-03) — not the legacy databases API.**

The env vars `NOTION_FOCUS_DB_ID`, `NOTION_TIERS_DB_ID`, `NOTION_SIGNALS_DB_ID` are named "DB_ID" for historical reasons but actually contain **data source IDs**. They go in the `data_sources` URL path.

### Reading rows from a DB

```
POST https://api.notion.com/v1/data_sources/{id}/query
Headers:
  Authorization: Bearer $NOTION_API_KEY
  Notion-Version: 2025-09-03
  Content-Type: application/json
Body: {"page_size": 100, "filter": {...optional...}}
```

### Creating a row (e.g. write a signal)

```
POST https://api.notion.com/v1/pages
Headers: (same as above)
Body: {
  "parent": {"data_source_id": "$NOTION_SIGNALS_DB_ID"},
  "properties": {...see Output format below...}
}
```

### Do NOT use these endpoints — they will return 404/400

- ❌ `GET /v1/databases/{id}` (legacy)
- ❌ `POST /v1/databases/{id}/query` (legacy)

If Notion returns an error message saying "Make sure the relevant pages and databases are shared with your integration" — **don't trust that text**. With the legacy endpoint that error is misleading for data-source-API DBs. Verify by trying the `/v1/data_sources/{id}/query` endpoint first. If THAT also fails with 404, only then conclude there's a real access problem.

## Pipeline

```
1. Preflight (≤30s)
   - fetch focus_areas from Notion → cache in run context
   - fetch source_tiers from Notion → cache in run context
   - load seen-URL set from sqlite (last 90 days)
   - send Telegram "🟡 started" with focus areas summary

2. Spawn sub-agents (per batch of 3-4 competitors)
   See "Token economy" section below — this is critical.
   Each sub-agent receives: tier-appropriate sources + this skill's methodology.
   Returns: compact JSON list of raw signal candidates.

3. Aggregate + dedup
   - Merge sub-agent returns
   - Filter out URLs in seen-set
   - Cross-source dedup (same event from multiple sources → merge)

4. Classify + score (one pass)
   - For each candidate: assign type, compute score per `scoring-rubric.md`
   - Drop everything < 3 (IGNORE bucket)

5. Output
   - Write all signals ≥ 3 to Notion "Competitor Signals" DB
   - Telegram push for signals scoring ≥ 8 (HIGH)
   - Update seen-URL set in sqlite
   - Send Telegram "✅ done" with HIGH/MED/LOW counts

6. On any failure
   - Telegram "⚠️ partial: <what failed>"
   - Always write run-status to runs.log
```

## Token economy — critical for cost control

This pipeline can spend $1-2 per run at Sonnet 4.5 prices if done naively. With the rules below it stays at $0.50-0.80. **Follow these or runs become expensive.**

### 1. Sub-agents per batch (mandatory)

Do NOT process 14+ competitors in one session. The shared context will balloon past 100k tokens and either crash or exceed budget. Instead:

- Split competitor list into batches of **3-4 companies**
- For each batch, spawn a sub-agent with its **own fresh context**
- Sub-agent's job: scan its batch → return compact JSON `{competitor, signals: [...]}` (target: ~500-800 tokens per sub-agent response)
- Main session receives N short JSON returns instead of accumulating 14 × 50k tokens of raw HTML

### 2. Tier-based scan depth

Don't scan every competitor with full source set. Use `source_tiers` from Notion:

| Tier | What to scan | Why |
|---|---|---|
| `top` | website (`/news`, `/press`, `/blog`), Google News, Crunchbase, ClinicalTrials.gov, industry press | Direct competitors, full pass |
| `mid` | Google News + own website `/news` only | Less direct, lighter touch |
| `watch` | Single Google News query `"<name>" pathology AI` | Probe only — flag if moves into our space |
| `drop` | Skip entirely | Removed from active monitoring |

This alone saves 30-50% of WebFetch cost vs scanning everyone fully.

### 3. Compact extractor pattern

When fetching a source, **don't pass raw HTML to the classifier**. Instead:

- Extractor sub-agent reads raw page
- Emits compact JSON: `{title, date, summary_one_line, url, source_type}`
- Classifier sees only this compact form — ~200 tokens per candidate instead of 5-50KB of HTML

### 4. Dedup before classification

Cheap dedup happens **before** any LLM scoring:

- Hash (URL OR normalized title) against sqlite seen-set
- If hash exists → skip silently, never reaches classifier
- Only unseen candidates pay the classifier token cost

### 5. Prompt caching

The agent's system prompt + this skill body + `scoring-rubric.md` are **identical across runs**. Anthropic API supports prompt caching with `cache_control` markers:

- Mark stable blocks (system + methodology) as `cache_control: {type: "ephemeral"}`
- Cache hits on these blocks cost ~10× less than fresh input
- Place dynamic content (focus_areas snapshot, candidate list) AFTER cached blocks

This requires the agent's code to set cache markers correctly — runtime concern, but mention it here as a requirement.

### 6. Don't reload skills mid-run

Paperclip loads SKILL.md once per agent run. Do not redundantly re-fetch supplementary skill files (`scoring-rubric.md`, `source-process.md`) inside sub-agents — pass relevant pieces in compact form via sub-agent's input prompt instead.

## Supplementary files (lazy-loaded when needed)

- **`scoring-rubric.md`** — event taxonomy + score formula + thresholds + anchor examples. Load when classifying.
- **`source-process.md`** — how to scan each source type, anti-patterns, extractor format. Load when in scan phase.

## Output format

### Notion "Competitor Signals" DB (one row per signal)

| Column | Type | Notes |
|---|---|---|
| Company | relation → Competitors DB | |
| Type | select | M&A / Funding / Market / Executive / Regulatory / Product / Clinical / Org |
| Date | date | from source |
| Title | text | one-line event |
| Summary | rich text | 2-3 sentences: what + why it matters |
| URL | url | canonical source |
| Source | select | website / news / crunchbase / sec / fda / linkedin / other |
| Score | number | 0-10 from rubric |
| Status | select | new (default) |
| Run | text | run_id for traceability |

### Telegram push (HIGH only, score ≥ 8)

**API call** (Paperclip env has no `curl` — use Node.js `fetch`):

```
POST https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage
Content-Type: application/json
Body: {
  "chat_id": "{TELEGRAM_CHAT_ID}",
  "text": "<message body — see template below>",
  "disable_web_page_preview": false
}
```

Do NOT set `parse_mode` — send as plain text. Telegram's markdown parser is strict and breaks on special characters in competitor names / titles.

If `TELEGRAM_BOT_TOKEN` or `TELEGRAM_CHAT_ID` env vars are missing → skip Telegram silently (Notion is primary sink). Log "telegram_skipped: not configured" in run summary.

**Message body template** (plain text, Russian where natural):

```
🔴 Competitor signal — <YYYY-MM-DD>

<Company>: <one-line event>
Type: <type>  Score: <X>/10
Why it matters: <one sentence in Russian>
Source: <URL>
Notion: <Notion row URL>
```

### Run status (Telegram) — same `sendMessage` API endpoint

Always sent at run start and end:

```
🟡 Monitor started: <N> competitors, window <D> days
Focus active: <list of active focus areas>
```

```
✅ Done — <S> signals (<H> HIGH, <M> MED, <L> LOW)
Digest: <Notion DB URL filtered to today>
```

Or on partial failure:

```
⚠️ Partial — <what worked, what didn't>
```

## Constraints

- **Never invent.** If a source returned nothing, log it. Don't guess.
- **Never duplicate.** sqlite dedup happens before classifier.
- **Russian-language output** for Telegram and Notion summaries. URLs and proper nouns stay in English.
- **Per-agent monthly budget** is a hard cap set in Paperclip — respect it.

## Failure modes

| Failure | Response |
|---|---|
| Notion API auth fails (401/500) | Halt. Telegram "🔴 Notion auth failed". |
| `Skill Focus` DB empty | Halt. Telegram "🔴 No focus configured — set up Notion DB first". |
| Single source fails (timeout/404) | Skip source, continue. Log as "X unavailable" in run digest. |
| Anthropic rate-limit (429) | Wait 30s, retry once. Second fail → halt. |
| Telegram notify fails | Continue. Notion is primary sink. Log to runs.log. |
| All sub-agents fail | Halt with "🔴 All scout sub-agents failed — check Railway logs". |

## See also

- Methodology: `scoring-rubric.md` (this directory)
- Scan procedure: `source-process.md` (this directory)
- Focus config: Notion DB (set in env, not in this skill)
- Tiers config: Notion DB (set in env, not in this skill)
