# OSS Radar Source Process

8 source categories. Each gets its own sub-agent in Step 2 of the pipeline.

**Default scan window:** last 4 days (matches twice-weekly cadence — Mon scans Thu-Sun, Thu scans Mon-Thu).

**Backfill mode:** when prompt explicitly says "backfill <N> months", widen window to last N months. Used for first-time runs or recovering from extended downtime.

For each sub-agent: pass `active_hypotheses Models tags`, `seen_urls`, `seen_names`, and the relevant section of this file below.

## Category A — HuggingFace

### Watched organizations (WebFetch each org page, look at "Models" tab + "Datasets" tab):

- https://huggingface.co/MahmoodLab — UNI, CONCH, TITAN, THREADS, Feather
- https://huggingface.co/bioptimus — H-Optimus
- https://huggingface.co/paige-ai — Virchow, PRISM
- https://huggingface.co/owkin — Phikon
- https://huggingface.co/google — Path Foundation, MedGemma
- https://huggingface.co/histai — Hibou, SPIDER, HISTAI datasets
- https://huggingface.co/kaiko-ai — Midnight FM, eva framework
- https://huggingface.co/KatherLab — various
- https://huggingface.co/microsoft — Prov-GigaPath

### General HuggingFace search:

```
WebSearch: "site:huggingface.co/models pathology histology" published in last week
WebSearch: "site:huggingface.co foundation model histopathology 2026"
```

### Attention filters:
- `pipeline_tag` ∈ {`image-segmentation`, `image-classification`, `image-feature-extraction`, `image-to-text`}
- Tags include: `medical`, `pathology`, `histopathology`, `wsi`, `cytology`
- Model card has a real "Model description" section (not auto-generated)

### Extractor output per model:
```json
{
  "name": "UNI-2",
  "organization": "MahmoodLab",
  "source": "HuggingFace",
  "url": "https://huggingface.co/MahmoodLab/UNI-2",
  "hf_url": "https://huggingface.co/MahmoodLab/UNI-2",
  "github_url": "https://github.com/mahmoodlab/UNI",
  "weights_url": "https://huggingface.co/MahmoodLab/UNI-2/resolve/main/pytorch_model.bin",
  "dataset_url": null,
  "license": "CC-BY-NC-4.0",
  "summary_raw": "Vision transformer pretrained on 100k WSI from 20 organs...",
  "parameters": "600M params, ViT-Large architecture",
  "datasets_mentioned": "MGH-private (100k WSI)",
  "openness_signals": {"code": "✅", "weights": "✅", "dataset": "❌"}
}
```

## Category B — arXiv

### Searches:

```
WebSearch: site:arxiv.org cs.CV histopathology OR "computational pathology" 2026
WebSearch: site:arxiv.org "foundation model" pathology last week
WebSearch: site:arxiv.org "vision-language" pathology 2026
```

### Filters:
- Categories: cs.CV, eess.IV, cs.LG
- Submitted within scan window
- Ignore pre-2026 papers

### Anti-patterns:
- Don't fetch arXiv PDFs (huge — 5MB+). Read the abstract page only.
- Don't include re-submissions / v2/v3 of papers already in seen-set (check by paper title, not URL)

### Look for `code coming soon` signals in abstract:
- Phrases: "code will be released", "weights to be made available", "upon publication"
- If found → openness_signals.code = "⏳ promised", add ETA if mentioned (else "unknown")

## Category C — GitHub

### Searches:

```
WebSearch: github.com topic:computational-pathology updated:>2026-04-23
WebSearch: github.com "digital pathology" OR "histology AI" stars:>10 created:>2026-04-15
WebSearch: github.com topic:histopathology pushed:>7d
```

### Filters:
- Repo has README (not empty)
- License is not GPL-only (preferably MIT / Apache 2.0 / BSD / CC-BY)
- Real code (not just paper-link or `awesome-list`)
- ≥10 stars OR comes from a known lab (MahmoodLab, Owkin, etc.)

### Extractor: read repo's README first 500 lines + LICENSE file.

## Category D — Papers with Code + Conference proceedings

### Searches:

```
WebSearch: paperswithcode.com pathology benchmark new SOTA 2026
WebSearch: "MICCAI 2026" OR "CVPR 2026" pathology accepted
WebSearch: "ICLR 2026" OR "NeurIPS 2026" pathology
```

### Look for:
- SOTA submissions on Patho-Bench, PathBench, EVA, HEST benchmarks
- Conference accepted-papers lists (often have links to code)

## Category E — Company blogs

### Watched URLs (WebFetch each, look at posts in scan window):

- https://www.bioptimus.com/news/
- https://www.owkin.com/blogs-case-studies/
- https://www.paige.ai/foundation-models
- https://www.aignostics.com/blog (Atlas FM)
- https://modella.ai (PathChat DX)
- https://www.kaiko.ai/blog

### Extractor: post title + date + summary. If post mentions GitHub/HF link, follow it for openness signals (don't classify based on blog alone — verify open code/weights).

## Category F — Aggregators + Benchmarks

### Watched URLs:

- https://grand-challenge.org/challenges/ — new competitions (filter by recent)
- https://github.com/DearCaat/Awesome-Computational-Pathology-Papers — curated paper list (check recent commits)
- https://birkhoffkiki.github.io/PathBench/ — FM leaderboard updates

### Filter for findings in scan window. New leaderboard entries = high signal.

## Category G — High-impact journals + news (proxy approach)

### Direct WebFetch FAILS on these (403 / paywall):
- ❌ cell.com
- ❌ nature.com (mostly)
- ❌ nejm.org

### Use proxies instead:

```
WebSearch: site:cell.com pathology OR histology OR "tumor microenvironment" foundation model 2026
WebSearch: site:nature.com pathology AI model 2026
WebSearch: nejm.org OR "lancet digital health" pathology AI 2026
WebSearch: site:news-medical.net pathology AI breakthrough 2026
WebSearch: site:geekwire.com pathology AI Microsoft OR Google OR Owkin 2026
WebSearch: scholar.google.com "pathology" "foundation model" 2026
```

### From news/PR articles extract: model name, organization, links to GitHub/HF (often in "Open Source Availability" block). Then follow up via HF/GitHub directly to verify openness signals.

## Category H — Microsoft Research Blog + Azure Foundry Labs + Google Health

### Watched URLs:

- https://www.microsoft.com/en-us/research/blog/?topic=ai-machine-learning
- https://labs.ai.azure.com (Microsoft Foundry Labs — GigaTIME, Path Foundation, etc.)
- https://blog.research.google/search/label/health

### Filter posts in scan window by keywords: pathology, histology, tumor, digital pathology, WSI.

## Common extractor anti-patterns

- ❌ Don't keep raw HTML in sub-agent context after parsing — extract to JSON, drop HTML
- ❌ Don't guess openness signals — verify by following links (HF page exists? GitHub repo has code?)
- ❌ Don't include findings already in seen-set (check `seen_urls` AND `seen_names`)
- ❌ Don't return long descriptions — `summary_raw` is one paragraph max, classifier writes the polished Summary
- ❌ Don't fetch paper PDFs — abstracts only (paper PDFs are 5-30MB each)
- ❌ Don't generate `summary_raw` from URL alone — actually fetch and read the page

## Output contract for each sub-agent

Each sub-agent returns a single JSON array. Empty array `[]` is valid (means nothing new in this source category).

```json
[
  {
    "name": "...",
    "source": "HuggingFace|GitHub|ArXiv|Company Blog|Conference|PapersWithCode",
    "organization": "MahmoodLab|...|Other",
    "url": "https://...",
    "hf_url": "https://huggingface.co/... or null",
    "github_url": "https://github.com/... or null",
    "weights_url": "direct link or null",
    "dataset_url": "https://... or null",
    "license": "MIT|Apache 2.0|CC-BY-NC|...|unknown",
    "summary_raw": "one paragraph from source",
    "parameters": "extracted or 'unknown'",
    "datasets_mentioned": "extracted or 'unknown'",
    "openness_signals": {
      "code": "✅|❌|⏳",
      "weights": "✅|❌|⏳",
      "dataset": "✅|❌|⏳"
    },
    "date_published": "ISO YYYY-MM-DD or null"
  }
]
```

Target ≤800 tokens per sub-agent response. If a category has 10+ new findings, only return top 5 by openness-signal richness — better to lose 5 weak findings than blow context budget.
