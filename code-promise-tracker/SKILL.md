---
name: Code Promise Tracker
key: code-promise-tracker
description: |
  Weekly check on papers/announcements marked [CODE_PROMISED] in the Notion AI Models
  Monitor DB — verify whether authors actually released the code/weights they promised.
  Updates status from "code promised" to "released" / "overdue" / "abandoned".
  PLACEHOLDER — full migration in Phase 4.
version: 0.2.0-placeholder
metadata:
  status: placeholder
  full_spec_pending: Phase 4
---

# Code Promise Tracker — placeholder

Full migration in Phase 4 after Competitive Intelligence and OSS Radar are validated on Paperclip.

## Logic to migrate

The mac task `code-promise-tracker-weekly` (Wed 10:13 local) reads Notion `AI Models Monitor` entries marked with `[CODE_PROMISED: <ETA or "unknown">]` in Notes. For each:

1. Re-check the paper's referenced GitHub repo / HuggingFace org
2. Classify current status:
   - **Released:** code/weights published → update entry, replace `[CODE_PROMISED]` with `[CODE_RELEASED: <date>]`
   - **Still promised:** ETA not yet reached → leave as-is
   - **Overdue:** ETA passed by 30+ days → add `[CODE_OVERDUE]` flag
   - **Abandoned:** ETA passed by 90+ days → suggest demotion to `Status=Not Relevant`
3. Telegram summary at run end

## Until Phase 4

Mac task `code-promise-tracker-weekly` remains source of truth.
