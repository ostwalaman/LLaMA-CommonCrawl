# WARC.WET Processing Pipeline — Test-v3.1 (NLTK + Line-Dedup + Page-Type Filters)

This README documents the **Test-v3.1** pipeline and provides a structured comparison against the **previous Markdown report** (`README_WET_Pipeline_Report.md`).  
Sources: v3.1 `wet_report.json`

---

## Executive summary

This repository contains a reproducible preprocessing pipeline for a CommonCrawl **WARC.WET** file. The pipeline:

- extracts plaintext documents from `conversion` records,
- performs line-based intra-document cleaning (nav/boilerplate trimming + repeated-line removal),
- applies **English-only filtering** (`langid`),
- computes lightweight quality metrics (non-ascii/digit/punct ratios + **NLTK stopword ratio**),
- performs **cross-document line-level dedup** (CCNet-style approximation),
- removes **page-type junk** (cookie/privacy boilerplate, e-commerce/listing pages, JS/template dumps),
- removes **exact duplicates** (MD5),
- includes near-duplicate detection (MinHash/LSH) and topic modeling (TF-IDF + NMF) for analysis.

**Key outcome (this run):**  
The WET sample is strongly multilingual; after English + quality filtering and dedup, the final dataset is **11,289 docs**. Page-type filtering shows a meaningful share of “clean” English pages are actually e-commerce/listings and other template-heavy pages.

---

## Data and environment

### Input
- WET file: `test_compression.warc.wet`

### Primary outputs (v3.1)
- `wet_raw_extracted.jsonl` — extracted docs after minimum length filter  
- `wet_cleaned_filtered.jsonl` — cleaned + English + quality filtered  
- `wet_line_deduped.jsonl` — cross-doc line-level dedup applied (NEW)  
- `wet_pagefiltered.jsonl` — page-type junk removed (NEW)  
- `wet_deduped.jsonl` — exact deduplicated corpus (final)  
- `wet_final.jsonl` — same as deduped (KenLM + ref-quality disabled)  
- `wet_report.json` — summary report  

---

## Methodology (Test-v3.1)

### Step 1 — WET extraction
**What happens**
- Iterate over all records using `ArchiveIterator`.
- Keep `conversion` records only.
- Decode text as UTF-8 (tolerate errors).
- Normalize whitespace.
- Drop documents shorter than **300 characters**.
- Write JSONL with: `url`, `host`, `content`.

**Results**
- Total records: **34,318**
- Conversion records: **34,317**
- Kept after length filter: **32,879**
- Dropped as too short: **1,438**

---

### Step 2 — Cleaning + English + quality filters
**Cleaning**
- Line-based filtering removes empty/very short lines and common nav fragments.
- Removes lines repeated ≥3 times within the same doc (template repetition).
- Computes repeat-line ratio proxy.

**English filter**
- Keep only documents where `langid` predicts `en`.

**Quality filters**
- Non-ASCII ratio
- Digit ratio
- Punctuation ratio
- **NLTK stopword ratio** (`word_tokenize` + NLTK stopwords)
- Repeat-line ratio threshold

**Results (from 32,879 extracted docs)**
- Dropped non-English: **19,103**
- Dropped low stopword ratio: **219**
- Dropped too repetitive: **1,003**
- Dropped digit-heavy: **53**
- Dropped too short (post-cleaning): **30**
- Dropped non-ASCII heavy: **1**
- Kept after cleaning + filtering: **12,470**

---

### Step 3 — Cross-document line-level dedup (CCNet-style approximation) — NEW
**What happens**
- Counts line document-frequency across the filtered corpus.
- Removes lines that appear across many documents (boilerplate/templates).
- Drops docs that become too short after boilerplate removal.

**Results (from 12,470 docs)**
- Kept: **12,320**
- Dropped too short after line-dedup: **150**
- Unique line hashes: **389,524**
- High-frequency lines identified: **5,920**

---

### Step 4 — Page-type junk filtering — NEW in v3.1
**Goal**  
Remove entire pages that are structurally low-value content types:
- cookie/privacy/consent boilerplate pages  
- e-commerce/listing pages (cart/checkout/shipping/sku/price heavy)  
- JS/template dumps (code-like artifacts)

**Results (from 12,320 docs)**
- Kept: **11,316**
- Dropped cookie/privacy: **61**
- Dropped e-commerce: **941**
- Dropped JS dumps: **2**

---

### Step 5 — Exact deduplication
**What happens**
- Normalize whitespace.
- Hash the text (MD5).
- Drop documents with previously seen hashes.

**Results (from 11,316 docs)**
- Exact duplicates removed: **27**
- Final deduped docs: **11,289**

---

## Results table (end-to-end, v3.1)

| Stage | Input | Output | Dropped | Retention |
|---|---:|---:|---:|---:|
| Conversion records → length filter | 34,317 | 32,879 | 1,438 | 95.81% |
| Clean + English + quality | 32,879 | 12,470 | 20,409 | 37.92% |
| Cross-doc line dedup (NEW) | 12,470 | 12,320 | 150 | 98.80% |
| Page-type junk filter (NEW) | 12,320 | 11,316 | 1,004 | 91.85% |
| Exact dedup | 11,316 | 11,289 | 27 | 99.76% |

---

## Language distribution (sanity check)
The corpus is strongly multilingual; English is a large plurality but not a majority, so dropping ~58% as non-English is expected for CommonCrawl/WET content.

---

## Near-duplicate detection (MinHash + LSH) — analysis
**Run configuration**
- Processed **3,000** documents (sampled).
- Clusters (size ≥2): **2**
- Largest cluster sizes: **[2, 2]**

**Interpretation**
Near-duplicate clusters exist but are small in this run; this is consistent with stronger upstream boilerplate removal and page-type filtering.

---

## Topic modeling (TF-IDF + NMF) — analysis
**Run configuration**
- Modeled **5,000** documents (sampled), **8 topics**.

**Observed patterns (high-level)**
- institutional/education/business content
- month/date archives
- e-commerce/cart/price signals (still present)
- JS/code-like artifacts (still present)
- cookies/privacy/account/login/terms (still present)

**Interpretation**
Even after page-type filtering, topic modeling indicates residual template content remains (especially login/account/cookies terminology). Further tuning of page filters may be warranted depending on downstream goals.

---

## Differences vs previous Markdown report

### A) Output differences (counts + new artifacts)

| Stage / Output | Previous README | Test-v3.1 | Δ (v3.1 − prev) | What this means |
|---|---:|---:|---:|---|
| Extracted after length filter (`wet_raw_extracted.jsonl`) | 32,879 | 32,879 | 0 | Same extraction/min-length behavior.  |
| Clean+English+quality (`wet_cleaned_filtered.jsonl`) | 13,436 | 12,470 | **−966** | v3.1 exposes and enforces stronger template rejection (too repetitive accounted explicitly).  |
| Cross-doc line dedup (`wet_line_deduped.jsonl`) | — | 12,320 | +12,320 | New intermediate corpus after removing boilerplate lines across documents.
| Page-type junk filtered (`wet_pagefiltered.jsonl`) | — | 11,316 | +11,316 | New intermediate corpus after dropping cookie/e-commerce/JS page types.
| Final exact dedup (`wet_deduped.jsonl`) | 13,406 | 11,289 | **−2,117** | Smaller (cleaner) final corpus due to the two new stages.  |
| Exact duplicates removed | 30 | 27 | −3 | Similar scale; not the main driver of change.  |

---

### B) What’s newly done (pipeline upgrades)

| New in v3.1 | Why it was added | What it changes vs previous |
|---|---|---|
| Cross-document line-level dedup | Topics/near-dup examples show strong templating; within-doc repetition removal isn’t enough. | Removes boilerplate that repeats across many docs and creates measurable “boilerplate prevalence” stats.
| Page-type junk filtering | Topic modeling showed cookie/cart/JS clusters; needed explicit removal of entire page categories. | Drops whole documents dominated by these patterns and quantifies contamination by type.
| More granular filter accounting | Prior README did not break out repetition-based rejections. | Makes it obvious how much data is lost to templating and why the corpus shrinks.  |

---

### C) What’s new in the analysis (vs the previous report)

#### 1) Boilerplate prevalence becomes measurable (not inferred)
| Metric | Previous README | v3.1 | New insight |
|---|---:|---:|---|
| Cross-doc boilerplate stats | Not available | `unique_line_hashes=389,524`, `high_freq_lines=5,920` | You can now track templating intensity and tune dedup thresholds based on real corpus statistics.

#### 2) Page-type contamination becomes measurable (not just “topic terms”)
| Page type | Previous README | v3.1 | New insight |
|---|---:|---:|---|
| E-commerce/listings | Topic signaled (cart/price topic) | **941 docs dropped** | Confirms a large chunk of “clean English” is still shopping/listing content.
| Cookie/privacy | Topic signaled (cookies/GDPR topic) | **61 docs dropped** | Cookie pages exist but your filter is conservative; many pages may mention cookies without being dominated by them. |
| JS/template dumps | Topic signaled | **2 docs dropped** | JS dumps are not the dominant contaminant under current detector settings. |

#### 3) Near-duplicate signal shifts in the expected direction
| Near-dup metric | Previous README | v3.1 | Meaning |
|---|---:|---:|---|
| Largest cluster sizes (sample=3k) | [3, 2] | [2, 2] | Reduced templated redundancy after new cleaning stages.  |

---

## Limitations
1. WET is plaintext-only; DOM/HTML structure is unavailable.   
2. Page-type filters are heuristic and conservative; they may miss borderline pages or (if over-tuned) drop valid content (e.g., reviews).
3. Near-duplicate detection is analysis-only in this run (no enforced removal).   
4. KenLM scoring and reference-quality classifiers are disabled.

---

## Recommended next steps
1. Tighten cookie/privacy detection only if the downstream goal is “article-like” content.
2. Consider turning near-duplicate clustering into an actual removal step (keep 1 representative per cluster).   
3. Add host concentration reporting and optional per-host caps for larger crawls.   
4. Consider KenLM as a scoring feature (not a hard filter) after calibration.
