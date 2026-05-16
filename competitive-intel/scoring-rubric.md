# Scoring Rubric — event taxonomy and score formula

How to classify and score each signal. Focus-specific boosts come from Notion runtime config, not from this file.

## Event taxonomy

Every signal must be classified into exactly one type. If a signal fits multiple, pick the **primary** (most impactful).

| Type | Examples | Type weight |
|---|---|---|
| `M&A` | acquisition, merger, divestiture, asset sale | +4 |
| `Regulatory` | FDA 510(k), CE mark, national regulator (e.g., Roszdravnadzor) approval/clearance, recall, withdrawal | +4 |
| `Funding` | seed/Series A-D, IPO, secondary, bridge | +3 |
| `Market expansion` | new country launch, new clinical area, distributor deal, platform integration | +3 |
| `Executive change` | C-level / VP / Head-of-X hire, departure, board change | +3 |
| `Product` | new product launch, major version, deprecation | +2 |
| `Clinical` | trial start, trial results, peer-reviewed publication | +2 |
| `Org signals` | layoffs, mass hires (>30/quarter), office opening, certification | +1 |

## Score formula

```
score = type_weight
      + focus_boost     ← from Notion "Skill Focus" DB; max +3, max −2 (penalize)
      + tier_boost      ← from Notion "Source Tiers" DB; top=+1, mid=0, watch=0 (unless watchlist-hit, +1)
      + regional_bonus  ← +1 if directly affects a market in focus_areas with status=active and tag=region
```

**Cap:** max score = 10, min score = 0.

## Thresholds

| Score | Bucket | Action |
|---|---|---|
| **8-10** | **HIGH** | Notion (status=new) + Telegram push |
| **5-7** | **MED** | Notion (status=new) |
| **3-4** | **LOW** | Notion with summary only (no detailed analysis) |
| **0-2** | **IGNORE** | Do not write |

## Calibration anchors

These examples define the scale — when scoring, compare against these. **Numbers reflect a sample focus configuration; actual focus boosts come from Notion.**

### Anchor A — score 9 (HIGH)

> Top-tier competitor announces FDA 510(k) clearance for a product in our active focus area.
- Type `Regulatory` (+4)
- Focus area active with boost +3
- Tier `top` (+1)
- Regional: 0 (FDA is global, not regional bonus)
- **Score: 8** → HIGH

### Anchor B — score 9 (HIGH, structural)

> Top-tier competitor acquired by larger company for >$100M.
- Type `M&A` (+4)
- Focus boost: 0 (broad acquisition, not focus-specific)
- Tier `top` (+1)
- M&A structural amplifier: events that redraw competitive map are HIGH regardless of focus → bump +4
- **Score: 9** → HIGH

### Anchor C — score 5 (MED)

> Top-tier competitor hires new VP Sales.
- Type `Executive change` (+3)
- Focus: 0 (sales hire isn't tied to clinical focus)
- Tier `top` (+1)
- **Score: 4** → LOW, but bumped to MED by rule: exec changes at top-tier always get baseline MED

### Anchor D — score 3 (LOW)

> Top-tier competitor publishes blog post about minor product version update.
- Type `Product` (+2) — incremental
- Focus: marginal match (+1)
- Tier `top` (+1)
- Dampening: marketing tone (−1)
- **Score: 3** → LOW

### Anchor E — score 0 (IGNORE)

> Competitor sponsors industry conference booth.
- Type: not classifiable (Org? Marketing?)
- Conference attendance excluded by convention
- **Score: 0** → don't write

### Anchor F — score 8 (HIGH, regional)

> Top-tier competitor signs distribution deal for a focus-active region.
- Type `Market expansion` (+3)
- Focus area `<region>` active (+3)
- Tier `top` (+1)
- Regional bonus: +1
- **Score: 8** → HIGH

## Type-specific amplifiers (rules that override pure formula)

Some event types reshape the competitive landscape and should score HIGH even when focus doesn't directly match:

- **M&A** involving a top-tier competitor → always HIGH (bump to ≥ 8)
- **IPO** of any tracked competitor → always HIGH
- **FDA / Regulatory withdrawal/recall** of a tracked competitor → always HIGH
- **Mass layoff** (>20% of headcount) of a top-tier → always MED at minimum

## Tie-breaking when one event has multiple sources

If the same event surfaces from 3 different sources (e.g., company press + Google News + industry blog):
- Use the **earliest** date among them (likely the original)
- Pick the URL from the highest-authority source: regulator > company press > industry press > generic news > LinkedIn aggregate
- Sum all URLs in a single Notion row, don't create duplicates

## Quiet-run protocol

If after a full pass no signal scores ≥ 3:
- Write one entry: `type=info, title="Quiet run YYYY-MM-DD", summary="No significant competitor activity in window"`
- Do NOT push Telegram (quiet = good news, don't be noisy)

## Editing this rubric

Changes to weights, thresholds, or amplifiers directly affect what reaches output. Be deliberate:
- Lowering thresholds → more Telegram noise
- Raising thresholds → may miss important signals
- Changing type weights → reshapes prioritization

When unsure, score last month's signals against the new rubric (manual backtest) before merging.
