# Boston Neighborhood Intelligence

**RAG the City · Track A, The Engine · Boston, 22 August 2026**

A question answering engine over Boston's open city data. Ask it about safety, city
responsiveness, food safety, parks or housing across the city's 26 neighbourhoods, and
it either shows its working or tells you honestly that it cannot answer.

> **Every number comes from SQL over 100% of rows. Retrieval's job is to make that SQL
> correct.** Embeddings cannot count, so no figure this system reports is ever read out
> of a vector store.

---

## The problem we set out to solve

Boston's open data gives wrong answers without ever raising an error. That turns out to
be the hard part, not the retrieval.

> "What is the most valuable property in Boston?"
>
> ~~$999,900~~ → **$2,448,193,300**

Assessed values are stored as text with commas, so `max()` sorts them alphabetically.
Nothing errors. Nothing warns. You get a confident, citable, wrong number that is off by
a factor of 2,450.

It is not one bad column. We measured the same class of failure across the whole corpus.

| What you ask for | Naive answer | Actual |
|---|---:|---:|
| Crime incidents | 290,130 | 146,933 |
| Restaurant inspections | 896,379 | 219,732 |
| Parkland, acres | 2,327 | 5,861 |
| Crimes in Roxbury, 2025 | 9,587 | 4,543 |

Nearly half of the rows in the crime file are not crimes. They are medical assists,
traffic accidents, sudden deaths and police investigations. A plain `count(*)` counts
all of them.

---

## How it works

A question walks six stages. Each one removes a class of failure before the next runs,
and every stage is recorded so a demo can show how the system narrowed rather than only
what it concluded.

**0. Guard.** Refuses questions about named private individuals before any retrieval
runs. The property file really does link owner names to addresses, which is exactly why.

**1. Disambiguate.** Resolves place names, and surfaces ambiguity instead of quietly
picking one.

**2. Anchor.** Pins an explicit time window and warns when a question reaches past what
the data covers.

**3. Route.** Picks one of six engines using rules, with no model call, so the routing
decision is instant and explainable.

**4. Execute.** SQL for numbers, retrieval for meanings, an honest abstention for gaps.

**5. Signal.** Attaches a confidence level and writes it into the answer text, not into
a hidden field.

---

## The wrong query is not discouraged, it is unavailable

A prompt full of rules is only a suggestion, and a 7B model holding a dozen of them will
eventually drop one. So the raw files are locked out of the query path entirely: a read
only connection, filesystem access disabled, and a table allowlist.

```
> SELECT max(total_value) FROM property
  2,448,193,300

> SELECT max(TOTAL_VALUE) FROM read_csv_auto('property-assessment.csv.gz')
  Permission Error: file system operations are disabled
```

We did not tell the model the rules and hope it followed them. We made the wrong query
impossible to write.

---

## Honesty, three different ways

Saying nothing useful is sometimes the correct answer, and the system distinguishes
between three reasons for it.

| Case | Example | Response |
|---|---|---|
| **Blocked** | "Where does John Smith live?" | Refused before retrieval, with the reason |
| **Abstained** | "How many licences were revoked?" | The licence tables hold only active records, so it says so |
| **Declined** | "Is Roxbury safe?" | Returns the metrics, refuses to issue the verdict |

Citations are enforced in code rather than encouraged by a prompt. A grounded answer
with no citation raises before it can reach the user.

---

## Built with

Everything runs locally. No cloud APIs, open source end to end.

**Data and SQL**
DuckDB 1.5.5 · DuckDB spatial · 1.67M rows, no sampling

**Retrieval**
ChromaDB · rank_bm25 · Reciprocal Rank Fusion, k=60

**Models, all local via Ollama**
`qwen2.5-coder:7b` writes SQL · `granite3.1-dense:8b` writes prose · `bge-m3` embeddings

**Interfaces and tooling**
Starlette demo UI · MCP server · pdfplumber · openpyxl

Sparse search is not decoration. A figure like `4200000` sits in a CSV field with no
currency symbol, so dense retrieval has no semantic neighbourhood to find it in. BM25
finds it on the digits. Running both and fusing by rank is what makes numeric lookups
work at all.

---

## The data

Nine sources from Analyze Boston, plus six of the city's own reference documents parsed
once and committed. Every geocoded record resolves to the same 26 neighbourhood polygons
by point in polygon in pure SQL, 290,130 crime rows assigned in under six seconds at
99.7% coverage.

| Source | Rows |
|---|---:|
| Food Establishment Inspections | 896,379 |
| Crime Incident Reports, 2023 to 2026 | 290,130 |
| Property Assessment, FY2026 | 184,552 |
| 311 Service Requests, 2026 | 78,526 |
| Licensing and entertainment licences | 4,905 |
| Open Space | 577 |
| Neighbourhood boundaries | 26 |

---

## Where it stands

**Working.** End to end. Ask a question, get a number with the query and its citations.

**Working.** 179 tests passing.

**Working.** All three honest non-answers verified live.

**Working.** Demo UI and MCP server, both offline.

**Open.** Committed eval numbers. The harness runs and the naive baseline is proven at
0% with 11 fabrications. Our own column is still being regenerated.

A question we can answer today, end to end, in about a second:

```
? How many crimes in Roxbury in 2025?
  route   aggregate
  SQL     SELECT count(*) FROM crime_only
          WHERE neighborhood = 'Roxbury' AND year = 2025
  answer  4,543
  cites   Crime Incident Reports · 146,933 rows · query shown
```

---

*Every figure measured against the committed extracts · Entirely local models · Open
source end to end*
