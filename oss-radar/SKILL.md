---
name: OSS Radar — Pathology AI
key: oss-radar
description: |
  Scan open-source pathology AI ecosystem (HuggingFace, arXiv, GitHub, PapersWithCode,
  high-impact journals, Microsoft Research blog, Azure Foundry Labs) for new foundation
  models, datasets, approaches relevant to active research hypotheses configured in
  Notion. Use this skill when the agent is triggered for scheduled OSS monitoring.
  PLACEHOLDER — full spec migrated in Phase 4.
version: 0.2.0-placeholder
metadata:
  status: placeholder
  full_spec_pending: Phase 4 migration from mac scheduled task
---

# OSS Radar — placeholder

This skill is a placeholder. Full content migrates from the existing mac-based skill once the Competitive Intelligence migration pattern is validated.

## Why deferred to Phase 4

- Competitive Intelligence migration must prove the Paperclip + Notion + agent-runtime pattern works first
- Existing mac task still functional (occasional failures, not constant)
- Migration risk: oss-radar has a longer history of working entries in the Notion "AI Models Monitor" DB — moving it carries higher rollback cost

## What migrates in Phase 4

- 8 source categories (HuggingFace, arXiv, GitHub, PapersWithCode, journals, Microsoft Research, Azure Foundry, industry press)
- 19-point scoring rubric (relevance + openness + technical)
- Notion writeback to `AI Models Monitor` DB
- Telegram push for hot findings (score ≥ 8 OR open code+weights)
- Backfill mode for historical gap recovery

Will follow the same three-layer pattern as `competitive-intel`:
- methodology in this repo
- focus parameters (active research hypotheses) read from Notion at runtime
- agent code in the agent runtime

## Until Phase 4

The mac scheduled task `oss-radar-twice-weekly` remains the source of truth and continues to run. Do not invoke this placeholder via Paperclip.
