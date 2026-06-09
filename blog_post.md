# How We Cut an EDA Tool's Energy Use by Up to 64% — Without Touching Its Code

*By Keerthana Yelchuru Venkata and Sriyan Ravuri | Group 14, NWI-IMC019 Green Software, Radboud University*

---

Every day, thousands of data scientists drop a CSV file into a profiling tool and get back a beautiful report: distributions, correlations, missing-value summaries, all in one click. It's incredibly convenient. But what does that convenience actually cost — in energy, in memory, in carbon?

That was the question driving our semester project in the Green Software course at Radboud University. We chose **fg-data-profiling** (the community-maintained successor to the well-known *pandas-profiling* / *ydata-profiling* library, available at [github.com/Data-Centric-AI-Community/fg-data-profiling](https://github.com/Data-Centric-AI-Community/fg-data-profiling)) as our subject — a library with ~13,600 GitHub stars that is embedded in tens of thousands of data pipelines and notebooks worldwide.

Our finding: **49–64% of the tool's energy and runtime can be cut with fewer than ten lines of code, purely from the caller side, without changing what the report contains.**

---

## The Hidden Cost of "Just Works"

When you call `pd.read_csv("data.csv")` with no extra arguments, Pandas plays it safe. It doesn't know what's in your file, so it allocates memory conservatively:

- String columns (endpoint names, HTTP methods, regions) become Python `object` dtype — roughly **50 bytes per cell**, because each value is a full Python heap object
- Numeric columns default to `int64` or `float64` even when the values would fit in half the space

For a 20 MB CSV file, these defaults inflated the in-memory DataFrame to **20.7 MB** of Python-allocated RAM just for the load stage — and that bloated DataFrame then flowed into every subsequent computation. ProfileReport's correlation engine and histogram passes had to process the full object-column representation on every pass.

We found this by building an **Energy Processing Diagram (EnPD)** — a map of where energy flows in the pipeline — and instrumenting each stage with `tracemalloc` (Python-level memory) and `psutil` (process-level RSS). The diagram pointed clearly to the data-load layer as the dominant memory hotspot.

![Energy Processing Diagram](plots/enpd_diagram.png)
*EnPD for the baseline fg-data-profiling pipeline (20 MB input, 15 W server TDP, 365 runs/year). Red borders mark the memory hotspot (data-load layer) and the compute hotspot (correlation engine).*

---

## The Fix: Tell Pandas the Truth

The optimisation was conceptually simple: instead of letting Pandas guess, we declared exactly what the data contains.

```python
# Specify exact dtypes — no inference overhead, no object bloat
DTYPE_MAP = {
    "status_code":      "int16",    # max 65535; 75% smaller than int64
    "response_time_ms": "float32",  # 7 sig. digits is enough; 50% smaller
    "bytes_sent":       "int32",    # max ~2 GB; 50% smaller
    "method":           "category", # 4 unique strings → 1-byte integer index
    "region":           "category", # 15 unique strings → 1-byte integer index
    "endpoint":         "category", # 200 unique strings → 1-byte integer index
}

# Load only the 7 columns the report will actually analyse
# (ip_address, session_id, user_agent are flagged Unsupported/Unique anyway)
USECOLS = ["timestamp", "method", "endpoint", "status_code",
           "response_time_ms", "bytes_sent", "region"]

df = pd.read_csv(path, dtype=DTYPE_MAP, usecols=USECOLS)
```

This applies three strategies from van Gastel's *Strategies for Green Software* infographic:

- **C5 — Algorithm to data (avoid copies):** load only the 7 needed columns; never decode the 3 high-cardinality or irrelevant ones
- **D6 — Improve algorithms:** `category` dtype stores 200 unique endpoint strings once in a lookup table; each row is a single byte instead of a full Python object
- **C3 — Store less:** the smaller DataFrame also means fewer pairwise correlation calculations (21 pairs instead of 45)

---

## The Numbers

We ran the **full** end-to-end pipeline — `ProfileReport(explorative=True).to_file()`, including the correlation engine and HTML rendering — on two dataset sizes, three runs each, measuring wall time, CPU time, memory, and energy via CodeCarbon.

| Metric | 20 MB | 50 MB |
|---|---|---|
| Wall-clock time | −49.2% (9.07 → 4.61 s) | −64.1% (16.67 → 5.98 s) |
| Measured energy (CodeCarbon) | −48.7% | −63.9% |
| DataFrame load memory | **−88.3%** (20.7 → 2.4 MB) | **−88.7%** (50.4 → 5.7 MB) |
| Process peak RSS | −9.8% | −8.6% |
| Output correctness | ✓ identical | ✓ identical |

The scaling behaviour is the most striking result: the optimised variant barely grows with input size (4.6 → 6.0 s for 2.5× more data), while the baseline grows steeply (9.1 → 16.7 s). **The larger the dataset, the bigger the relative win.**

![Scaling and reduction charts](plots/full_scaling.png)
*Wall-clock time vs. input size. The optimised variant's flat-ish growth vs. the baseline's steep growth means the gap widens in the regime that matters — production data.*

One honest caveat: while the DataFrame load memory fell 88%, the full-pipeline peak process memory (RSS) only dropped ~9%. ProfileReport's correlation engine and HTML rendering allocate hundreds of megabytes independent of input dtypes — they set the memory ceiling. A load-only measurement would have hidden this. We measured the full pipeline precisely to avoid that mistake.

---

## What Does This Mean in Practice?

### Case 1 — CI/CD server (15 W TDP, European grid, 365 daily runs on 50 MB data)

| | Per run | Per year (one pipeline) | 10,000 pipelines |
|---|---|---|---|
| Energy saved | 160 J | **16.2 Wh** | 162 kWh |
| CO₂eq avoided | — | **4.8 g** | **48 kg** ≈ 300 km by petrol car |

### Case 2 — Developer laptop (25 W TDP, same grid and cadence)

| | Per run | Per year (one pipeline) | 10,000 pipelines |
|---|---|---|---|
| Energy saved | 267 J | **27.1 Wh** | 271 kWh |
| CO₂eq avoided | — | **8.0 g** | **80 kg** ≈ 500 km by petrol car |

Per-pipeline savings sound small. But fg-data-profiling runs in tens of thousands of automated pipelines. At scale, the aggregate matters — and these savings compound every single day.

---

## A Technical Deep Dive: Why Category Dtype Changes Everything

For those curious about the mechanism: a `category` column stores its unique values once in a lookup array (the "categories") and represents each row as an integer index into that array. For a column like `endpoint` with 200 unique values across 100,000 rows, the baseline `object` dtype allocates 100,000 Python string objects; the `category` dtype allocates 200 Python strings (once) plus 100,000 one-byte integers. Memory drops by roughly 50×.

More importantly, pandas groupby, value_counts, and correlation operations on `category` columns work on the integer indices rather than string comparisons — turning O(n × string_length) operations into O(n × 1). This is why wall time fell 49–64%, not just memory.

---

## Beyond the Main Optimisation

We also measured four additional levers:

- **Configuration tuning:** stacking `minimal=True` on top of the dtype fix cuts runtime to −72%, but removes the correlation matrix — a trade-off, not a free win
- **Row sampling:** profiling a 25% random sample cuts peak memory by 67% with key statistics staying within 0.5% of the full-data values
- **CSV → Parquet:** columnar format cuts load time by 95% and on-disk size by 85% — attacking the storage energy the LCA flagged as comparable to runtime energy
- **Result caching:** for repeated runs on unchanged data, a hash-based cache avoids 99.98% of compute (cache hit: 0.3 ms vs 1.6 s)

We also report one **negative result**: we expected exporting JSON instead of HTML to be faster (skipping browser rendering). It was 23% *slower* and used twice the memory. Serialising the full description object outweighs the render savings. Wrong intuitions are worth documenting.

---

## The Broader Lesson

The optimisation required no changes to fg-data-profiling itself, no algorithmic redesign, no new dependencies. It was purely about *telling Pandas the truth about the data upfront*.

This pattern recurs constantly in green software work: library defaults are tuned for safety and generality, not efficiency. When you know your data — its column types, its cardinality, which columns you actually need — you can almost always do better. If your pipeline reads CSVs and you have not specified `dtype=` and `usecols=`, you are paying a memory and energy tax you don't need to.

The change is ten lines. The savings are real and scale with every run.

---

*Written as part of NWI-IMC019 Green Software at Radboud University, 2025–2026.*

*For the full scientific report with methodology, EnPD diagrams, energy calculations, and references: see [report/report.pdf](report/report.pdf).*
