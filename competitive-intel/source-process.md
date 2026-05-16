# Source Process — how to scan, what to extract

Methodology for the scan phase. **Which** competitors are scanned and **what depth** comes from Notion's "Source Tiers" DB at runtime. This file describes the **how**.

## Source taxonomy

| Source | What it gives | Cost (tokens) | Reliability |
|---|---|---|---|
| **Company website `/news`, `/press`, `/blog`, `/newsroom`** | Authoritative press releases | Medium (5-15KB raw) | High |
| **Google News query** | Broad coverage, current | Medium (5KB per query) | Medium — verify |
| **Crunchbase profile** | Funding, people, key dates | Medium | High |
| **SEC EDGAR (for public competitors)** | 8-K material events | Low (filtered) | Highest |
| **ClinicalTrials.gov** | Trial registrations | Low (structured) | High |
| **Industry press** (AuntMinnie, Pathology News, Med-Tech Innovation, digitalpathologyplace) | Specialized coverage | Low (RSS-style) | High |
| **National regulator registry** (FDA 510(k), CE database, Roszdravnadzor, NMPA) | Authoritative regulatory events | Low (structured) | Highest |
| **LinkedIn company posts** | Self-promotion, hire announcements | Low (via Google search) | Low — fragmented |

## Depth per tier

`source_tiers` from Notion classifies each competitor. The scan logic per tier:

### Tier `top` — full pass

For each competitor in this tier:
1. WebFetch company website main page → look for press/news section
2. WebFetch `/news`, `/press`, `/blog`, `/newsroom` (try variants, don't crash on 404)
3. Google News: `"<company name>" (news OR announcement OR launch)` with date filter
4. Google search: `"<company name>" site:linkedin.com/company/<slug>/posts`
5. Crunchbase: WebFetch `crunchbase.com/organization/<slug>` for funding + people
6. ClinicalTrials.gov: search by sponsor name
7. Regulator-specific queries:
   - FDA: `"<company name>" site:accessdata.fda.gov`
   - CE: `"<company name>" CE mark IVD`
   - Roszdravnadzor: `"<company name>" Росздравнадзор регистрация`
8. Industry press: `"<company name>" site:auntminnie.com OR site:pathology-news.com OR site:digitalpathologyplace.com`

### Tier `mid` — news only

For each competitor in this tier:
1. WebFetch own `/news` or `/press` page (skip if doesn't exist, don't probe variants)
2. Google News: `"<company name>"` with date filter
3. Skip Crunchbase deep dive (unless an event triggers — e.g., funding signal in news → then do Crunchbase verification)
4. Skip ClinicalTrials.gov
5. Skip LinkedIn

### Tier `watch` — thin probe

For each competitor in this tier:
1. Single Google News query: `"<company name>" pathology AI`
2. If 0 results in date window → skip silently (don't include in digest)
3. If hits → bump treatment to `mid` for this run, flag `[WATCHLIST-HIT]` in digest line

### Tier `drop` — skip

Skip entirely. Do not include in any scan.

## Window of relevance

Default scan window: **from `last_run_at` to now**. Typically 3-4 days for twice-weekly cadence.

If `last_run_at` is null (first run after deploy) → use **last 14 days**.

If user explicitly asks for backfill → use specified window (max 90 days; longer windows risk drowning in old noise).

## Compact extractor pattern (mandatory)

When scanning a source, do NOT return raw HTML to the next pipeline stage. Use an extractor sub-step:

**Extractor input:** raw HTML/text from source (5-50KB)
**Extractor output:** compact JSON, ~200-300 tokens per candidate:

```json
{
  "company": "Mindpeak",
  "title": "Mindpeak receives FDA 510(k) clearance for HER2 IHC scoring",
  "date": "2026-05-12",
  "url": "https://mindpeak.ai/news/fda-clearance-her2",
  "source_type": "company-press",
  "event_summary_oneline": "FDA 510(k) clearance for HER2 IHC AI scoring product",
  "raw_excerpt": "<150-word relevant excerpt only — not full page>"
}
```

The classifier downstream receives this compact form. **Raw HTML never reaches scoring step.**

## Anti-patterns (don't do these)

| Anti-pattern | Why it's bad |
|---|---|
| Fetching same URL multiple times in one run | Wasted tokens; cache the fetch |
| Asking classifier to read raw HTML | Burns 10× more tokens than needed |
| Scanning every competitor with full source set regardless of tier | Defeats the tier system, +50% cost |
| Including marketing fluff in event_summary | Inflates classifier input, doesn't help scoring |
| Re-fetching `/news` from competitor whose website was 404 last 3 runs | Persistent failures should be cached as failures with TTL |
| Returning multiple signals about the same event from one source | Dedup within source before passing to classifier |

## Source-fail handling

If a source returns 4xx/5xx/timeout:
- **First failure for this source this run:** retry once after 5s
- **Second failure:** mark source unavailable for this run, continue with others
- **3 consecutive runs of same failure:** include `[STALE SOURCE]` flag in digest, suggest tier review
- Don't halt the run because one source is down. Other sources should still produce signals.

## When in doubt

If unsure whether a piece of text is a "signal" or noise:
- **Yes** if a competitor's PM or CEO would care about it
- **No** if it's marketing recap, blog re-share, or third-party speculation
- **Maybe** → mark as LOW, let Kristina decide via Notion status

## See also

- `scoring-rubric.md` — what to do AFTER extracting (classify + score)
- `SKILL.md` — high-level pipeline + token economy rules
