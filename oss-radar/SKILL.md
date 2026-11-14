---
name: OSS Radar — Pathology AI
key: oss-radar
description: |
  Scan open-source pathology AI ecosystem (HuggingFace, arXiv, GitHub, PapersWithCode,
  high-impact journals, Microsoft Research blog, Azure Foundry Labs, company blogs) for new
  foundation models, datasets, approaches relevant to OneCell's active research hypotheses
  (read from Notion) and DataHub Golden Set (proxied via onecell-agents Railway service).
  Score each finding 0-19 (relevance + openness + technical), write rich-analysis cards
  to Notion AI Models Monitor, push Telegram digest for hot findings (score ≥ 8 OR open
  code+weights). Use this skill when triggered for scheduled OSS monitoring (Mon+Thu 09:07 МСК)
  or manually via Assign Task.
version: 1.0.0
metadata:
  cadence: twice weekly (Mon + Thu 09:07 МСК)
  config_sources:
    hypotheses_db: Notion "Продуктовые гипотезы" (id from env NOTION_HYPOTHESES_DS_ID)
    models_monitor_db: Notion "AI Models & Approaches Monitor" (id from env NOTION_MODELS_DS_ID — read for dedup+calibration, write new findings)
    datahub_endpoint: HTTP GET to onecell-agents (url from env DATAHUB_ENDPOINT_URL)
---

# OSS Radar — Open Source Pathology AI Monitor

## Mission

Twice-weekly digest of new open-source models, papers, datasets in computational pathology. Each finding scored 0-19 against OneCell's active research hypotheses AND DataHub Golden Set. Hot findings (score ≥ 8 OR full open stack) go to Telegram; everything ≥ 4 goes to Notion AI Models Monitor with rich analysis (openness, population, clinical limits, OneCell-dataset match).

**Not a news aggregator** — a filter producing 0-5 hot findings per run, plus 5-15 medium-relevance entries in Notion for later review.

## Configuration sources (runtime)

Three things vary between runs and must be **fetched at runtime**, not embedded:

1. **`active_hypotheses`** — from Notion DB `$NOTION_HYPOTHESES_DS_ID` ("Продуктовые гипотезы")
   Filter rows where `Статус` IN (`Dev`, `Clinical review`, `Эксперимент`, `in progress`, `To do`).
   Extract `Models` multi-select values (e.g. `HER2`, `Ноттингем`, `Метастазы`, `tumor`, `плацента`, `Artefacts WSI`) — these drive scoring boosts.

2. **`datahub_snapshot`** — from `$DATAHUB_ENDPOINT_URL` (no auth header needed, internal Railway service)
   ```
   GET $DATAHUB_ENDPOINT_URL
   → {
       "annotation_types": [{"type": "...", "geometry": "...", "size_hint": "..."}],
       "tumor_loc_summary": [{"localization": "...", "n_slides": ...}],
       "fetched_at": "ISO"
     }
   ```
   These drive the `Наш датасет` field analysis (what OneCell already has annotated that matches the finding).

3. **`recent_findings`** — from Notion DB `$NOTION_MODELS_DS_ID` ("AI Models & Approaches Monitor")
   Two queries:
   - All rows where `Date Found` ≥ 90 days ago → build seen-set for dedup
   - Last 20 rows where `Status` IN (`Not Relevant`, `Relevant`, `In Use`) → calibration signals (see Step 0)

**Output:** write new findings to the SAME `$NOTION_MODELS_DS_ID` DB.

If hypotheses DB is unreachable → halt and Telegram-alert. If DataHub endpoint is unreachable → continue but write `n/a — DataHub unavailable` in `Наш датасет`. If models monitor DB is unreachable → halt (need it for dedup, can't risk duplicates).

## Notion API access (critical — read before any Notion call)

The Paperclip-managed runtime has **Node.js** but **no `curl` / `jq`**. Use Node's built-in `fetch`.

**All Notion API calls must use the data-source API (2025-09-03) — not the legacy databases API.**

The env vars `NOTION_HYPOTHESES_DS_ID` and `NOTION_MODELS_DS_ID` contain **data source IDs**. They go in the `data_sources` URL path.

### Reading rows from a DB

```
POST https://api.notion.com/v1/data_sources/{id}/query
Headers:
  Authorization: Bearer $NOTION_API_KEY
  Notion-Version: 2025-09-03
  Content-Type: application/json
Body: {"page_size": 100, "filter": {...optional...}}
```

### Creating a row (new finding)

```
POST https://api.notion.com/v1/pages
Headers: (same as above)
Body: {
  "parent": {"data_source_id": "$NOTION_MODELS_DS_ID"},
  "properties": {...see Output format below...}
}
```

### Updating an existing row (when finding has new info — e.g. weights released)

```
PATCH https://api.notion.com/v1/pages/{page_id}
Headers: (same as above)
Body: {"properties": {...changed fields only...}}
```

### Do NOT use these endpoints — they will return 404/400

- ❌ `GET /v1/databases/{id}` (legacy)
- ❌ `POST /v1/databases/{id}/query` (legacy)

If Notion returns an error message saying "Make sure the relevant pages and databases are shared with your integration" on a legacy endpoint — **don't trust that text**. Verify by trying the `/v1/data_sources/{id}/query` endpoint first.

## Pipeline

```
1. Preflight (≤60s)
   - Fetch active_hypotheses from Notion → cache Models tags + Зона
   - Fetch datahub_snapshot from $DATAHUB_ENDPOINT_URL → cache annotation_types + tumor_loc_summary
     (on failure: continue with empty snapshot, flag in run summary)
   - Fetch recent_findings (last 90 days) from Notion AI Models Monitor → build seen-set:
     - seen_urls = lowercased URLs from URL, huggingface, веса, dataset fields
     - seen_names = lowercased Name (model name match for dedup when URLs differ)
   - Fetch calibration set: last 20 rows with Status in (Not Relevant / Relevant / In Use)
     → derive per-source / per-organization boost or penalty (see Step 3 calibration)
   - Send Telegram "🟡 OSS Radar started — N hypotheses, M models in seen-set"

2. Scan (sequential sub-agents — MANDATORY, not parallel)

   Spawn sub-agents via Task tool, one per source category. Total: 8 sub-agents.

   **CRITICAL — spawn sub-agents SEQUENTIALLY, one at a time.**
   Do NOT spawn multiple Task tool calls in parallel (single message with multiple tool_use blocks).
   Anthropic API has per-minute token rate limits; parallel sub-agents will hit HTTP 429.

   Sub-agent batches:
     A. HuggingFace orgs (MahmoodLab, bioptimus, paige-ai, owkin, google, histai, kaiko-ai, KatherLab)
        + general HF search
     B. arXiv (cs.CV, eess.IV, cs.LG with histopathology / "computational pathology" / "foundation model")
     C. GitHub (topic:computational-pathology, "digital pathology" updated:>7d)
     D. PapersWithCode + MICCAI/CVPR 2026 accepted lists
     E. Company blogs (bioptimus.com/news, owkin.com/blogs-case-studies, paige.ai/foundation-models,
        aignostics.com/blog, modella.ai)
     F. Aggregators (grand-challenge.org/challenges, DearCaat/Awesome-Computational-Pathology-Papers,
        birkhoffkiki.github.io/PathBench/)
     G. High-impact journals + news (Cell/Nature via Scholar+news-medical proxies — direct WebFetch fails 403)
     H. Microsoft Research blog + Azure Foundry Labs + Google Health blog

   Each sub-agent:
     - Receives: scan window (default last 4 days), active_hypotheses Models tags, seen-set
     - Scans its source category per source-process.md
     - Filters out URLs and names already in seen-set BEFORE returning
     - Returns: compact JSON [{name, source, organization, url, hf_url, github_url, weights_url,
       dataset_url, license, summary_raw, parameters, datasets_used, openness_signals}, ...]
     - Target: ≤800 tokens per sub-agent response

   This takes ~10-15 minutes total instead of ~3-4 parallel, but avoids 429 errors.
   If a sub-agent hits 429 internally despite this, wait 60 seconds and retry once.

3. Aggregate + score
   - Cross-category dedup (same model from HuggingFace + GitHub → merge into one finding)
   - Re-verify nothing slipped past dedup
   - Apply calibration boost/penalty per source/organization (from Step 1 calibration set)
   - For each unique candidate: compute score per scoring-rubric.md (0-19)
   - Drop anything score < 4 (noise)
   - Apply hard rules from scoring-rubric.md:
     - Open code + open weights → hard min Relevance=High
     - Code-only or paper-with-promised-code → Status="To Research", Notes marker [CODE_PROMISED]
     - Closed FM from big lab → Relevance ≤ Medium, "closed" flag in Summary
   - Determine Telegram trigger: raw score ≥ 8 OR (open code AND open weights)

4. DataHub matching (for "Наш датасет" field)
   For each finding:
   - Compare finding's domain/annotation_type/localization vs datahub_snapshot
   - Generate one of: ✅ Применимо / ⚠️ Частично / ❌ Не применимо
   - If datahub_snapshot was unavailable in preflight → write "n/a — DataHub snapshot unavailable this run"

5. Write to Notion
   - For each finding (score ≥ 4): create new page OR update existing page if matched in seen-set with new info
   - All fields populated per "Output format" below — empty fields are a failure mode (Kristina has explicitly flagged "скупой анализ")

6. Telegram digest (only if Telegram-trigger count > 0)
   - Build compact digest with top 3 hot findings expanded + rest as one-liners
   - Send via direct Telegram API (no notify-telegram.sh — that's mac-only)

7. On any failure
   - Send Telegram "⚠️ partial: <what worked, what didn't>"
   - Update Paperclip issue accordingly
```

**Critical sub-agent rule**: do NOT process source categories inline in the main session. Without sub-agents the context balloons past 200k tokens (each WebFetch is 5-30KB raw). The Task tool is the only way to keep context under control.

## Token economy — critical for cost control

This pipeline can spend $2-3 per run at Sonnet 4.5 prices if done naively (8 source categories × 30+ web fetches each). With the rules below it stays at $1.0-1.5. **Follow these or runs become expensive.**

### 1. Sub-agents per source category (mandatory)

8 source categories → 8 sub-agents, sequential. Each gets fresh context, returns compact JSON.

### 2. Compact extractor pattern

Within each sub-agent: when fetching a source page, **don't keep raw HTML in context**. Extract immediately to compact JSON:
```
{name, organization, url, hf_url, github_url, weights_url, dataset_url, license, summary_one_line, parameters, datasets_mentioned, openness_signals: {code: ✅/❌/⏳, weights: ✅/❌/⏳, dataset: ✅/❌/⏳}}
```
Target ~200 tokens per candidate. The classifier in the main session sees only this.

### 3. Dedup before LLM

`seen_urls` and `seen_names` from preflight (Notion query, no extra DB). Each sub-agent receives these and filters BEFORE returning. Saves the entire scoring + Notion-write cost for already-known findings.

### 4. Prompt caching

The skill body + scoring-rubric.md + source-process.md are **stable across runs**. The agent runtime should mark them as `cache_control: {type: "ephemeral"}`. Dynamic content (hypotheses snapshot, candidate list) goes AFTER cached blocks.

### 5. Tiered Cell/Nature handling

Don't WebFetch cell.com / nature.com directly — they return 403 and waste a turn. Use Google Scholar + news-medical proxies (see source-process.md category G).

### 6. Don't reload supplementary skill files in sub-agents

Pass relevant pieces of source-process.md into each sub-agent's input prompt instead of having the sub-agent re-fetch the file.

## Supplementary files (lazy-loaded when needed)

- **`scoring-rubric.md`** — 0-19 scoring formula + hard rules + thresholds + anchor examples. Load in Step 3 (classify).
- **`source-process.md`** — 8 categories of sources with URLs, scan rules, extractor patterns, anti-patterns. Load when spawning sub-agents in Step 2.

## Output format

### Notion "AI Models & Approaches Monitor" DB — exact property names

Schema verified 2026-05-18. Property names are **case-sensitive**, including Cyrillic ones.

| Property name | Type | Notes |
|---|---|---|
| `Name` | title | Model/paper name (this is the page TITLE property) |
| `Type` | select | one of: `Model`, `Paper`, `Approach`, `Dataset`, `Benchmark` |
| `Domain` | multi_select | one or more of: `Histology`, `Cytology`, `CDI`, `Oncology`, `General PathAI`, `Artifacts` (default `Histology`) |
| `Source` | select | one of: `HuggingFace`, `GitHub`, `ArXiv`, `Company Blog`, `Conference`, `PapersWithCode` |
| `Organization` | select | one of: `MahmoodLab`, `Bioptimus`, `PAIGE`, `Owkin`, `Google`, `Microsoft`, `Lunit`, `KatherLab`, `Other` |
| `Status` | select | `New` (default for score 4-7), `Review` (score ≥ 8 OR open code+weights), `To Research` (CODE_PROMISED) |
| `Relevance` | select | `High`, `Medium`, `Low` (per hard rules in scoring-rubric.md) |
| `Date Found` | date | ISO YYYY-MM-DD (today) |
| `Parameters` | text | N parameters, training compute, architecture brief |
| `Datasets` | text | training datasets with sizes, OR `n/a (private)` |
| `URL` | url | canonical source URL (paper or repo) — property key is `userDefined:URL` in API |
| `huggingface` | url | HF URL (empty if N/A) |
| `веса` | url | direct link to weights (empty if not open) |
| `dataset` | url | dataset URL (empty if not open) |
| `Summary` | text | 2-3 sentences: what it is, what's different, key numbers (dataset/params/metrics) |
| `Notes` | text | **Rich analysis — see template below. MANDATORY to fill fully.** |
| `Наш датасет` | text | **DataHub match — see template below. MANDATORY to fill (or explicit n/a with reason).** |

**Common mistakes that cause `validation_error`:** writing `URL` instead of `userDefined:URL` in the properties payload (Notion API needs the internal key, not the display name); using `huggingface_url` instead of `huggingface`; omitting `Domain` (it has no default in Notion).

### `Notes` field — structured rich analysis (MANDATORY, no empty fields)

Use this exact template (Kristina has flagged "скупой анализ" — empty Notes is a failure):

```
[score=X/19] hypothesis_match: <tag from active_hypotheses или "—">
match: annotation=<type>, localization=<loc>, stain=<stain>
rationale: <1-2 lines — why this score>

📦 Openness:
- code: ✅ <github URL> | ❌ closed | ⏳ promised <ETA>
- weights: ✅ <HF URL> | ❌ closed | ⏳ promised <ETA>
- dataset: ✅ <URL> | ❌ private | ⏳ promised
- license: <name, commercial use Y/N>

📊 Population / training data:
- N patients / N slides / N tiles
- geo: <countries>, age: <range>, sex: <if stated>
- staining: <H&E / IHC marker / multiplex>
- bias risk: <if any — e.g. single-center, only Asian cohort>

⚠️ Clinical limitations / gaps:
- <what's NOT covered — e.g. TCT only, no conventional Pap>
- <missing classes — e.g. no AGC>
- <missing validation — e.g. no external>

🎯 OneCell relevance:
- hypothesis: <specific hypothesis name from Notion DB or "—">
- value: <how it could help — fine-tune backbone / eval set / replace existing approach>

[CODE_PROMISED: <ETA "Q3 2026" or "unknown">]    ← add ONLY if paper promises code but hasn't released
```

If any sub-section is genuinely unknowable (e.g. organization didn't disclose patient counts), write `n/a — <one-line reason>` for that bullet. Never leave a sub-section empty.

### `Наш датасет` field — DataHub matching (MANDATORY)

If DataHub snapshot was available in preflight, use the appropriate template:

✅ Applicable:
```
✅ Применимо: <annotation_type list from datahub_snapshot> (<size hints>)
   Локализация: <matching localization from tumor_loc_summary>
   Use case: <fine-tune classification head / use as eval set / compare SOTA / etc.>
   Sheet: docs.google.com/spreadsheets/d/1d_T6b4ySj9hQ_Qq8Vy1mVSK1-ZdOO2Ak2KrD5kqcZqo
```

❌ Not applicable:
```
❌ Не применимо: <reason — e.g. "model is on molecular data, our DataHub is H&E/IHC slides only">
```

⚠️ Partial match:
```
⚠️ Частично: <what works — e.g. "tumor sheet (1.1M) good for backbone fine-tune, but IHC HER2 doesn't cover their multiplex">
```

If DataHub snapshot was unavailable:
```
n/a — DataHub snapshot unavailable this run (endpoint failed). Re-evaluate manually.
```

### Telegram digest (only if Telegram-trigger count K > 0)

**API call** (Paperclip env has no `curl` — use Node.js `fetch`):

```
POST https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage
Content-Type: application/json
Body: {
  "chat_id": "{TELEGRAM_CHAT_ID}",
  "text": "<message body — see template below>",
  "disable_web_page_preview": true
}
```

Do NOT set `parse_mode` — send as plain text. Telegram's markdown parser breaks on `[`, `_`, `(` etc. that appear in paper titles and URLs.

If `TELEGRAM_BOT_TOKEN` or `TELEGRAM_CHAT_ID` env vars are missing → skip Telegram silently (Notion is primary sink). Log `telegram_skipped: not configured` in run summary.

### Telegram trigger (UNAMBIGUOUS)

A finding goes to Telegram if:
- `raw score ≥ 8` (before hard-rule modifiers to Status/Relevance), OR
- `open code AND open weights` (full open stack — always notify regardless of score)

Status/Relevance modifiers from hard rules **do not block** Telegram. If Notion gets Relevance=Medium due to closed FM but raw score=14 — it STILL goes to Telegram (with 🔒 closed marker).

### Telegram message body template (plain text, Russian where natural)

```
🔬 OSS Radar — <K> hot findings

1. <Model Name> (<Source>) score=X/19
   match: <hypothesis tag>, <annotation>, <localization>
   📦 code: ✅/❌  weights: ✅/❌  data: ✅/❌  license: <short>
   <one-line summary with key number>
   🎯 OneCell: <short value — fine-tune / eval / replace X>
   📄 paper/repo: <URL>
   📋 Notion: <Notion page URL>

2. <Model Name 2> (<Source>) score=Y/19
   ...

(up to top 3 expanded)

Plus N more in Notion: <URL of AI Models Monitor filtered to today>
Run: 2026-MM-DD HH:MM МСК
```

Top 3 hot findings are expanded as above. Remaining hot findings get one-line entries:
```
4. <Name> score=X — see Notion: <URL>
```

**Mandatory fields in each expanded block**: name, source, score, code/weights/data flags, OneCell value, Notion URL. Without these Telegram is useless — Kristina can't make a decision at a glance.

### Run status (Telegram)

Always sent at run start:

```
🟡 OSS Radar started
Active hypotheses: <list of Models tags>
Seen-set: <N> models in last 90 days
DataHub: ✅ loaded | ⚠️ unavailable
```

Always sent at end:

```
✅ OSS Radar done — <K> hot, <M> medium-relevance (total <T> findings)
Telegram digest: <yes if K>0 else "no hot findings this run">
Notion: <URL filtered to today>
```

Or on partial failure:
```
⚠️ OSS Radar partial — <what worked, what didn't>
```

## Constraints

- **Never invent.** If a source returned nothing, log it. Don't guess parameters/datasets/etc.
- **Never duplicate.** Dedup via Notion seen-set happens in every sub-agent BEFORE classification.
- **Fill every field.** Empty `Notes` or `Наш датасет` is a failure mode — write `n/a — <reason>` if data genuinely unknown.
- **Russian-language output** for Telegram and Notion `Notes`/`Summary`/`Наш датасет`. URLs, model names, organization names stay in English.
- **Per-agent monthly budget** is a hard cap set in Paperclip — respect it. Default ~$15-25/month at twice-weekly cadence.

## Failure modes

| Failure | Response |
|---|---|
| Notion API auth fails (401/500) | Halt. Telegram "🔴 OSS Radar: Notion auth failed". |
| Hypotheses DB returns 0 active rows | Halt. Telegram "🔴 OSS Radar: no active hypotheses configured". |
| DataHub endpoint fails (timeout/500) | Continue. Write "n/a — DataHub unavailable" in `Наш датасет`. Note in run summary. |
| Models monitor DB unreachable | Halt. Telegram "🔴 OSS Radar: can't read seen-set, refuse to risk duplicates". |
| Single source category fails (timeout/404) | Skip category, continue with others. Log "X unavailable" in run digest. |
| Anthropic rate-limit (429) | Wait 60s, retry once. Second fail → halt. |
| Telegram notify fails | Continue. Notion is primary sink. Log to run summary. |
| All sub-agents return 0 findings | Send Telegram "✅ Quiet run — 0 new findings". |
| Sub-agent crash (>2 retries) | Skip that source category, continue. Telegram "⚠️ category X failed". |

## See also

- Methodology: `scoring-rubric.md` (this directory)
- Scan procedure: `source-process.md` (this directory)
- Hypothesis config: Notion DB "Продуктовые гипотезы" (set in env, not in this skill)
- DataHub snapshot: served by onecell-agents Railway service at `$DATAHUB_ENDPOINT_URL`
- Code-promise tracker (separate skill): picks up `[CODE_PROMISED]` markers in Notes
