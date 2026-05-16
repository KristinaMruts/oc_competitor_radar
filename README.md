# Competitive Radar Skills

Versioned skills for AI agents running on Paperclip. These skills define **methodology** — how agents scan competitive intelligence, classify events, score signals. Runtime parameters (focus areas, source tiers) come from Notion, not from this repo.

## Three-layer architecture

```
┌────────────────────────────────────────────────────────────────┐
│ Layer 1 — agent code (Python, FastAPI)                          │
│   Repo: <private agent runtime repo>      │
│   Deploys: git push → Railway autoredeploy                      │
│   Defines: HTTP endpoints, source fetchers, classifier code,    │
│            sub-agent orchestration, sqlite state                │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Layer 2 — methodology (this repo, public)                       │
│   Repo: KristinaMruts/oc_competitor_radar                            │
│   Deploys: git push → Paperclip refreshes Company Skills        │
│   Defines: event taxonomy, scoring formula, scan procedure,     │
│            pipeline structure, token economy rules              │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Layer 3 — runtime config (Notion, private)                      │
│   DBs: "Skill Focus", "Source Tiers", "Competitor Signals"      │
│   Edits: via Notion UI by PM                                    │
│   Defines: current focus areas with boosts, competitor tiers    │
│   Audit: Notion built-in revision history per row               │
└────────────────────────────────────────────────────────────────┘
```

**Why three layers?**

Each layer changes on a different cadence and by a different person:

| Layer | Changes | Who edits | Cadence |
|---|---|---|---|
| Code | Bug fixes, new features | engineer | irregular |
| Methodology | Scoring tweaks, new rules | PM + engineer | weekly-monthly |
| Config (Notion) | Focus shifts, new competitors, tier changes | PM | weekly |

Mixing them caused our previous monitoring tasks to break repeatedly: a methodology tweak required code redeploy, a focus shift required code redeploy, secrets ended up versioned with logic. This separation fixes that.

## Repo layout

```
README.md                         this file
competitive-intel/
  SKILL.md                        pipeline + token economy + Notion config refs
  scoring-rubric.md               event types, score formula, thresholds, anchors
  source-process.md               how to scan each source type + extractor pattern
oss-radar/                        Phase 4 placeholder
code-promise-tracker/             Phase 4 placeholder
```

## How Paperclip uses these skills

- This repo is imported as a **Company Skills** source in Paperclip
- Paperclip's `companySkillService` watches the repo and propagates changes — no manual sync
- Agents see skill `name + description` initially; full `SKILL.md` body loads when the agent decides the skill is relevant
- Supplementary `.md` files load lazily on reference

## Runtime config (Notion)

Two Notion databases hold what changes most often:

### `Skill Focus` DB

Defines current strategic priorities. Read by the agent at runtime.

| Column | Type | Notes |
|---|---|---|
| Area | text | short tag, e.g. "breast cancer", "IHC scoring" |
| Status | select | `active` (boost), `penalize` (dampen), `watch` (log only) |
| Boost | number | typically −2 to +3 |
| Notes | text | rationale, last review |

### `Source Tiers` DB

Classifies tracked competitors by depth of scanning. Read by the agent at runtime.

| Column | Type | Notes |
|---|---|---|
| Competitor | relation → Competitors DB | |
| Tier | select | `top` (full pass), `mid` (news only), `watch` (probe), `drop` (skip) |
| Status | select | `active`, `dropped` |
| Notes | text | rationale for tier |

## How to change things

**Change focus:** edit Notion `Skill Focus` row. Next run uses the new value. Notion history shows who/when.

**Change a competitor's tier:** edit Notion `Source Tiers` row. Same.

**Change scoring methodology:** edit `scoring-rubric.md` here. Commit with reason. Push. Next run uses updated rubric.

**Change scan procedure:** edit `source-process.md` here. Commit, push.

**Change pipeline flow:** edit `SKILL.md` here. Commit, push.

**Change agent code:** edit Python in the the agent runtime repo. Push. Railway redeploys.

## License

Methodology is generic enough to share — repo is public. The Company-specific configuration lives in Notion (private). The agent infrastructure (HTTP endpoints, Notion integration, sqlite state) lives in the private the agent runtime repo.

If you find this useful for your own competitive monitoring needs, feel free to adapt — but the file paths and Notion DB structure are wired to this specific setup.
