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
   - fetch focus_areas from Notion Skill Focus DB → cache in run context
   - fetch competitors with Tier+Tier Status from Notion Competitors DB
     → filter to Tier Status="active", group by Tier (top/mid/watch)
   - fetch existing signals from Notion Signals DB created in last 90 days
     → build seen-set: {url_hash, (company+title+date)_hash}
   - send Telegram "🟡 started" with focus areas summary + total competitors count

2. Spawn sub-agents using the Task tool — MANDATORY, not optional.
   Split active competitors into batches:
     - all top tier (7 companies) → 2 batches of 3-4 each
     - all mid tier (7 companies) → 2 batches of 3-4 each
     - all watch tier (5 companies) → 1 batch (lighter scan)
   Total: ~5 sub-agents in parallel.
   Each sub-agent:
     - Receives: its batch of competitors + their Tier + scan-depth rules
     - Scans according to source-process.md (depth varies by tier)
     - Filters out candidates whose URL is in the seen-set BEFORE returning
     - Returns: compact JSON [{competitor, signals:[{title,date,url,source,raw_excerpt}, ...]}]
     - Target: ≤500 tokens per sub-agent response

3. Main session: aggregate sub-agent returns
   - Cross-source dedup (same event from multiple sources → merge URLs into one signal)
   - Re-verify nothing slipped past dedup (defensive — sub-agents should have already filtered)

4. Classify + score (one pass, classifier reads scoring-rubric.md)
   - For each unique candidate: assign type, compute score
   - Drop everything < 3 (IGNORE bucket)
   - Map score → Bucket field: 8-10 HIGH, 5-7 MED, 3-4 LOW

5. Output
   - Write all signals (Bucket ∈ {HIGH, MED, LOW}) to Notion Competitor Signals DB
   - Telegram push for each HIGH (one message per signal)
   - Send Telegram "✅ done — N signals (H HIGH, M MED, L LOW)"
   - If 0 signals after dedup: write one quiet-run row + send "✅ quiet run — no new signals" to Telegram

6. On any failure
   - Telegram "⚠️ partial: <what failed>"
   - Update issue status accordingly
```

**Critical sub-agent rule**: do NOT process competitors inline in the main session. The Task tool (sub-agents) is the only way to keep context under control and to actually cover all configured competitors. A run that only scans 3 of 7 top-tier companies is a failure even if the issue is marked done.

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

### 4. Dedup against existing Notion signals

We do NOT keep a separate sqlite seen-set. The Notion Competitor Signals DB itself is the seen-set.

**At preflight (Step 1):**

```
POST /v1/data_sources/$NOTION_SIGNALS_DB_ID/query
Body: {
  "filter": {
    "property": "Detected",
    "date": {"after": "<90 days ago ISO>"}
  },
  "page_size": 100
}
```

(paginate via `next_cursor` if needed)

Build two in-memory sets from results:
- `seen_urls` = lowercased URL strings
- `seen_titles` = lowercased (Title + Company-page-id + Date) tuples

**In each sub-agent (Step 2):**

Pass `seen_urls` and `seen_titles` to the sub-agent. The sub-agent must filter candidates against both sets BEFORE returning them to main session. A candidate matches if:
- its URL (lowercased, query-string stripped) is in `seen_urls`, OR
- the (title-lowercased, company-id, date) tuple is in `seen_titles`

Matched candidates are dropped silently — they were already reported in a prior run.

**Why this matters**: in the first production run we wrote "Roche acquires PathAI for $1.05B". Without dedup, every subsequent run would re-classify, re-score and re-Telegram that same event for the next 90 days. With dedup, we get only NEW events per run.

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

### Notion "Competitor Signals" DB — exact property names (match these!)

| Property name | Type | Notes |
|---|---|---|
| `Title` | title | one-line event description (this is the page TITLE property) |
| `Company` | relation → Competitors DB | array of competitor page IDs |
| `Type` | select | one of: `M&A`, `Funding`, `Market`, `Executive`, `Regulatory`, `Product`, `Clinical`, `Org` |
| `Date` | date | event date from source (ISO YYYY-MM-DD) |
| `Summary` | rich_text | 2–3 sentences: what happened + why it matters for the company |
| `URL` | url | canonical source URL |
| `Source` | select | one of: `website`, `news`, `crunchbase`, `sec`, `fda`, `linkedin`, `industry-press`, `other` |
| `Score` | number | 0–10 from rubric |
| `Bucket` | select | one of: `HIGH`, `MED`, `LOW`, `info` (computed: 8-10=HIGH, 5-7=MED, 3-4=LOW, info for quiet-run) |
| `Status` | select | `new` (default) — Kristina updates later to starred/ignored/actioned |
| `Run ID` | rich_text | Paperclip run_id for traceability (NOTE: exact name `Run ID` with space, not `Run`) |
| `Detected` | created_time | auto-set by Notion, do not write |

**Property names are case-sensitive and must match exactly.** Use these literal strings as keys in the `properties` payload when calling `POST /v1/pages`. Common mistakes that cause `validation_error`: writing `Run` instead of `Run ID`, omitting `Bucket`, or skipping `Title` (which is the required title property).

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
