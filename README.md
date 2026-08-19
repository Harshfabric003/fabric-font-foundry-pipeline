# Font & Foundry Data Quality Lakehouse

A Microsoft Fabric medallion-architecture pipeline that ingests messy, multi-source
font metadata and reconciles it into clean, canonical font family and foundry
records using fuzzy matching and confidence scoring.

## Problem

Font metadata across vendors, marketplaces, and internal catalogs is rarely
consistent — the same font family shows up with typos, abbreviated style names,
inconsistent casing, and missing foundry attribution. Reconciling this manually
is slow and error-prone. This project automates that reconciliation and flags
anything it can't confidently resolve, rather than guessing.

## Architecture

Bronze -> Silver -> Gold medallion layers, built entirely in Microsoft Fabric
(Lakehouse, PySpark notebooks, Delta tables), orchestrated end-to-end with a
Fabric Data Pipeline, and served through a Power BI Direct Lake dashboard.

```
Google Fonts API ---\
Vendor export CSV -----> Bronze (raw Delta tables)
Foundry reference CSV -/
                              |
                              v
                    Silver (cleaning + fuzzy matching + confidence scoring)
                              |
                              v
                    Gold (confidence summary, foundry coverage,
                           unresolved records, avg confidence by style)
                              |
                              v
                    Power BI dashboard (Direct Lake)
```

### Orchestration

The three notebooks are chained into a single Fabric Data Pipeline
(`font_foundry_pipeline`) with sequential "on success" dependencies — each
stage only runs if the previous one completes without error.

`01_bronze_ingest` -> `02_silver_clean_match` -> `03_gold_aggregate`

Configured with a daily schedule via Fabric's built-in Schedule trigger
(left disabled in this portfolio project to avoid unnecessary compute usage).

<img width="1302" height="924" alt="image" src="https://github.com/user-attachments/assets/079f6b7d-64e2-4205-b9d6-d01dd98cc89d" />


## Layers

### Bronze — raw ingestion
- **Google Fonts API** — pulled live via `requests` inside a Fabric notebook,
  no manual download step, ~1,500+ real font records
- **Vendor export CSV** — hand-crafted, deliberately messy (typos, abbreviated
  styles, CamelCase names, version suffixes) to simulate a real vendor feed
- **Foundry reference CSV** — canonical family-to-foundry lookup table used as
  the matching target

### Silver — cleaning and fuzzy matching
- Regex-based name cleaning: CamelCase splitting, version-suffix stripping,
  style-word removal
- Fuzzy matching against the canonical reference using `rapidfuzz` (`WRatio`
  scorer, `default_process` normalizer)
- Confidence tiering: >=90 high confidence, 75-89 medium confidence, <75
  unresolved

### Gold — aggregation
- `gold_confidence_summary` — record counts and percentages by match tier
- `gold_foundry_coverage` — matched font counts per foundry
- `gold_unresolved_records` — records needing manual review, sorted by
  near-miss score
- `gold_avg_confidence_by_style` — average confidence broken out by font style

## Results

| Match tier | Records | Percentage |
|---|---|---|
| High confidence | 38 | 90.5% |
| Medium confidence | 2 | 4.8% |
| Unresolved | 2 | 4.8% |

18 distinct foundries matched. The 2 unresolved records
(`XYZ Custom Display`, `Unknown Vendor Font`) were deliberately unmatchable
test cases with no canonical counterpart — confirming the pipeline correctly
flags genuinely unresolvable names rather than forcing a low-quality match.

<img width="884" height="833" alt="Screenshot 2026-08-19 at 14 03 54" src="https://github.com/user-attachments/assets/3b1d02c7-a86b-42ad-a69e-12185ba6e409" />


## Known limitations and debugging notes

- **Case-sensitivity bug (found and fixed):** `rapidfuzz` does not lowercase
  strings by default. Early runs compared lowercase cleaned names against
  Title Case canonical names, which suppressed scores across the board —
  most severely on short names (e.g. "DM Sans" scored 60 instead of ~100).
  Fixed by adding `processor=default_process` to the `extractOne` call, which
  raised the high-confidence rate from 9.5% to 90.5%.
- **Compound style words** (e.g. "SemiBold", "ExtraBold") can get incorrectly
  split by the CamelCase-splitting step, leaving an orphan word ("semi",
  "extra") that the style-word strip list has to separately account for.
- Typos that remove an entire trailing word's worth of signal (e.g.
  "Lato-Ligh" for "Lato Light") can still score below the confidence
  threshold even when a human would recognize the match — this is treated as
  a correct, conservative outcome rather than a bug.

## Tech stack

Microsoft Fabric (Lakehouse, Notebooks, Data Pipelines, Power BI Direct Lake),
PySpark, pandas, rapidfuzz, Python, Git/GitHub.

## Repo structure

```
fabric-font-foundry-pipeline/
├── README.md
├── data/raw/
│   ├── vendor_export.csv
│   └── foundry_reference.csv
├── notebooks/
│   ├── 01_bronze_ingest.ipynb
│   ├── 02_silver_clean_match.ipynb
│   └── 03_gold_aggregate.ipynb
└── docs/
    ├── pipeline_run_success.png
    └── powerbi_dashboard.png
```
