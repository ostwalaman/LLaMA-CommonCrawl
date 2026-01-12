# WARC.WET Processing Pipeline (NLTK + Regex) — Findings & Analysis

## Executive summary
This repository contains a reproducible preprocessing pipeline for a CommonCrawl **WARC.WET** file. The pipeline:
- extracts plaintext documents from `conversion` records,
- performs **line-based boilerplate cleaning** (regex + frequency filtering),
- applies **English-only filtering** (`langid`),
- computes **NLTK-based stopword ratio** and other lightweight quality metrics,
- removes **exact duplicates** (MD5 hash),
- includes **near-duplicate detection** (MinHash/LSH) and **topic modeling** (TF‑IDF + NMF) for analysis.

The main outcome from the current run is that the input WET sample is **strongly multilingual**, and the filtered English dataset still contains significant **template/junk page patterns** (cookie banners, redirect notices, e-commerce pagination, forum templates), which motivates additional filtering.

---

## Data and environment
**Input**
- WET file: `test_compression.warc.wet`

**Primary outputs**
- `wet_raw_extracted.jsonl` — extracted docs after minimum length filter
- `wet_cleaned_filtered.jsonl` — cleaned + English + quality filtered
- `wet_deduped.jsonl` — exact deduplicated corpus
- `wet_report.json` — summary report (optional)

**Key libraries (and why)**
- `warcio`: streaming parsing of WARC/WET records (correct format handling)
- `langid`: fast language identification on noisy web text
- `nltk`: word tokenization + standard English stopwords for stopword ratio
- `re` (regex): structural cleaning (whitespace, boilerplate patterns), ratio heuristics
- `hashlib`: fast exact dedup via content hashing
- `datasketch`: MinHash/LSH for near-duplicate discovery (analysis)
- `scikit-learn`: TF‑IDF + NMF topic modeling (analysis)
- `tqdm`: progress reporting

---

## Methodology

### Step 1 — WET extraction
**What happens**
- Iterate over all records using `ArchiveIterator`.
- Keep `conversion` records only.
- Decode text as UTF‑8 (tolerate errors).
- Normalize whitespace.
- Drop documents shorter than **300 characters**.
- Write each doc as JSONL with: `url`, `host`, `content`.

**Results**
- Total records: **34,318**
- Conversion records: **34,317**
- Kept after length filter: **32,879** (95.81% of conversion)
- Dropped as too short: **1,438** (4.19% of conversion)

---

### Step 2 — Cleaning + English + quality filters
**Cleaning**
- Line-based filtering removes empty/very short lines and common nav fragments.
- Removes lines repeated ≥3 times in the same doc (template repetition).
- Computes a **repeat-line ratio** proxy.

**English filter**
- Keep only documents where `langid` predicts `en`.

**Quality filters**
- Non‑ASCII ratio
- Digit ratio
- Punctuation ratio
- **NLTK stopword ratio** (computed with `word_tokenize` + NLTK stopwords)
- Repeat-line ratio threshold (templated pages)

**Results**
Starting from **32,879** extracted docs:
- Dropped non‑English: **19,166** (58.29% of extracted)
- Dropped low stopword ratio: **223**
- Dropped digit-heavy: **51**
- Dropped non‑ASCII heavy: **3**
- Kept after cleaning + filtering: **13,436** (40.86% of extracted)

> Note: Compared to a regex-only stopword approximation, NLTK tokenization/stopwords typically changes the “low stopword ratio” rejection rate because token boundaries and stopword lists are more complete.

---

### Step 3 — Exact deduplication
**What happens**
- Normalize whitespace.
- Hash the text (`MD5`).
- Drop documents with previously seen hashes.

**Results**
Starting from **13,436** filtered docs:
- Exact duplicates removed: **30** (0.223%)
- Final deduped docs: **13,406** (99.78% retained)

---

## Results table (end-to-end)
| Stage | Input | Output | Dropped | Retention |
|---|---:|---:|---:|---:|
| Conversion records | 34,317 | 32,879 | 1,438 | 95.81% |
| Clean + English + quality | 32,879 | 13,436 | 19,443 | 40.86% |
| Exact dedup | 13,436 | 13,406 | 30 | 99.78% |

---

## Language distribution (sanity check)
A sample of **20,000** extracted documents was classified with `langid`:

Top languages:
- `en`: 8,266 (41.33%)
- `ru`: 1,281 (6.40%)
- `de`: 1,140 (5.70%)
- `es`: 1,057 (5.29%)
- `ja`: 1,037 (5.18%)
- `fr`: 992 (4.96%)
- `zh`: 902 (4.51%)
- `it`: 624 (3.12%)
- `pt`: 444 (2.22%)
- `nl`: 423 (2.11%)
- `pl`: 409 (2.04%)
- `la`: 292 (1.46%)
- `cs`: 253 (1.26%)
- `id`: 252 (1.26%)
- `vi`: 244 (1.22%)

**Interpretation**
- English is ~41.3% of the sample, so removing ~58–59% as non‑English is expected for CommonCrawl/WET content.

---

## Near-duplicate detection (MinHash + LSH)
**Goal**
Identify pages that are near-identical (templated content, pagination variants, mirrors) that exact hashing will not catch.

**Implementation**
- Build token shingles.
- Create MinHash signatures.
- Index with LSH (approximate Jaccard similarity).
- Cluster via union-find to get transitive groups.

**Run configuration (current notebook run)**
- Processed **3,000** documents (sampled for runtime).
- Near-dup edges found: **4**
- Clusters (size > 1): **2**
- Largest clusters: **[3, 2]**

**Observed clusters (examples)**
- Cluster 1 (size=3): TurboTax review pagination pages (highly templated).
  - https://turbotax.intuit.com/reviews/online/?page=9711
  - https://turbotax.intuit.com/reviews/online/deluxe/?page=11873
- Cluster 2 (size=2): University of Michigan Press contributor profile pages (shared template).
  - https://press.umich.edu/Contributors/J/Jin-Dal-Yong
  - https://press.umich.edu/Contributors/B/Boal-Dean

**Interpretation**
The near-dup clusters are dominated by **template-based pagination and profile pages**, which is typical in web corpora and is a strong signal to add template/junk filters and/or remove near-duplicates at scale.

---

## Topic modeling (TF‑IDF + NMF)
**Goal**
Provide an interpretable view of the dominant content patterns in the corpus and surface “junk” clusters.

**Run configuration (current notebook run)**
- Modeled **5,000** documents (sampled for runtime).
- Model: `TfidfVectorizer(ngram_range=(1,2))` + `NMF(n_components=8)`

**Topic patterns observed (high-level)**
- Topic 0: general site/news/business + nav terms — e.g., news, services, business, contact, home
- Topic 1: month/date archive pages — e.g., october, september, november, december, 2021
- Topic 2: cookie consent / GDPR banners — e.g., cookies, cookie, consent, gdpr, website
- Topic 3: redirect notice pages — e.g., redirect, redirect notice, previous page, visit page
- Topic 4: e-commerce/cart/price listings — e.g., cart, price, shop, sale, shipping
- Topic 5: javascript/code-heavy pages — e.g., function, var, return, document, window
- Topic 6: forums (phpBB-style) — e.g., phpbb, forum, board, login, register
- Topic 7: geography/currency entity lists — e.g., usd, eur, republic, united, island

**Interpretation**
Several topics correspond to **non-content templates** (cookie/GDPR banners, redirect notices, cart/price pages, JS code blocks, forum boilerplate). This indicates that language + basic quality filtering alone is not sufficient to produce a high-value English corpus.

---

## Limitations
1. **Heuristic quality filters are conservative**
   - They may keep English template pages (menus, review pagination, cookie banners).
   - They may drop some valid technical pages (code-heavy documentation) depending on punctuation/code density.

2. **Near-duplicate detection was run on a sample**
   - The current notebook run uses a subset (e.g., 3k docs) for runtime.
   - Running on the full corpus will likely find more clusters and remove more redundancy.

3. **WET is plaintext-only**
   - DOM structure and HTML signals are not available; boilerplate removal is necessarily approximate.

4. **No model-based quality scoring**
   - This pipeline does not include perplexity-based filtering or classifier-based quality scoring.

---

## Recommendations / next steps
1. **Add junk/template filters**
   - Remove cookie-consent pages, redirect notices, e-commerce pagination, forum templates.
   - Use topic-model clusters to define rule-based detectors.

2. **Scale near-duplicate removal**
   - Increase near-dup coverage beyond a sample.
   - Apply a retention policy (keep 1 representative per cluster).

3. **Add host concentration reporting**
   - Track top-domain dominance pre/post filtering (top1/top5/top20 shares).
   - Use it as a quality signal (template sites often dominate).

---

## Reproducibility: run order
1. Extract WET → `wet_raw_extracted.jsonl`  
2. Clean + English + quality → `wet_cleaned_filtered.jsonl`  
3. Exact dedup → `wet_deduped.jsonl`  
4. Near-dup detection (analysis)  
5. Topic modeling (analysis)  

---

## Notes for classroom use (NLTK vs regex)
The notebook intentionally uses **both**:
- **NLTK** for linguistic processing (tokenization and stopword computation).
- **Regex** for structural cleanup and quick heuristics (whitespace, nav patterns, digit/punct counting).

This separation mirrors common web-corpus preprocessing practice: NLTK provides standardized NLP primitives, while regex handles high-throughput text hygiene and template artifacts.
