# Boston Neighborhood Intelligence — how the system actually works

Written 22 Aug 2026, 14:20 ET, against `main` @ `5d1276f`, **verified by running
it**. Everything below is observed behaviour, not plan. Feed this to ChatGPT as
the source of truth for any diagram — the PRD is now out of date in several
places, listed at the end.

---

## The one-sentence thesis

> **Every number comes from SQL over 100% of rows. Retrieval's job is to make
> that SQL correct.** No count is ever read out of an embedding.

---

## The big change from the PRD: it is a PIPELINE, not just a router

The PRD described one router with six branches. The built system is a
**six-stage progressive-disclosure pipeline**, and routing is only stage 3.
Each stage eliminates a class of failure before the next runs.

```
0  GUARD         refuse person-directed lookups BEFORE any retrieval
1  DISAMBIGUATE  resolve place names; surface ambiguity, never guess
2  ANCHOR        pin an explicit time window
3  ROUTE         pick the engine — deterministic rules, no model call
4  EXECUTE       SQL for numbers · retrieval for text · abstain for gaps
5  SIGNAL        attach an explicit confidence level
```

Every stage a question passes is recorded in `Answer.trace`, so a demo can show
*how* the system narrowed, not only what it concluded. **This staging is the
most distinctive thing about the architecture and any diagram should show it.**

---

## Stage 3: the six routes, and what each actually runs

| Route | Triggered by | Engine | View / source used |
|---|---|---|---|
| `aggregate` | how many, trend, top-N | DuckDB SQL over every row | `crime_only`, `svc311`, `food_inspections`, `property_homes`, `open_space` |
| `scorecard` | neighborhood profile / compare | same SQL layer, multi-metric | same views |
| `definition` | "what does X mean" | retrieval only, **no SQL** | 136 chunks from the city's reference PDFs |
| `lookup` | qualitative / thematic | hybrid BM25 + dense, fused by RRF | free-text columns + chunks |
| `value_judgment` | "is it safe / good / best" | metrics **plus explicit refusal to rank** | same views |
| `unanswerable` | no supporting data exists | abstains **with a stated reason** | — |

Routing is **rules first, LLM classification only as fallback** — so the common
path costs no model call and is fully explainable.

### The view mapping is the integration contract

```python
VIEW_FOR_ROUTE = {
  "crime":    "crime_only",        # `crime` overstates crime by ~97%
  "requests": "svc311",
  "food":     "food_inspections",  # `food_violations` has ~4x the rows
  "property": "property_homes",    # `property` includes 8,545 parking spaces
  "parks":    "open_space",
}
```

Each choice is a semantic-layer rule, not a preference. Picking the wrong view
is exactly how a naive system produces a confident wrong number.

---

## The execution lock — this is a differentiator, show it

Generated SQL never touches DuckDB directly. It goes through `scripts/query.py`,
which imposes three locks:

1. **read-only connection** — no INSERT / UPDATE / DROP / ATTACH
2. **`enable_external_access = false`** — `read_csv_auto()`, `ST_Read()` and
   httpfs *error* rather than merely being discouraged
3. **table allowlist** — anything outside the cleaned tables is rejected
   before execution

Lock 2 is the one that matters for correctness. This query:

```sql
SELECT max(TOTAL_VALUE) FROM read_csv_auto('data/property-assessment.csv.gz')
```

returns **999,900** instead of **2,448,193,300** — the raw column is VARCHAR so
`max()` compares strings — and nothing objects. Through this path it raises a
Permission Error instead.

> **The line for the deck:** we didn't tell the model the rules and hope it
> followed them. We made the wrong query impossible to write.

---

## Verified live output (ran these at 14:18)

**Numbers — `aggregate`**
```
Q  How many crimes in Roxbury in 2025?
   route      aggregate
   SQL        SELECT count(*) FROM crime_only
              WHERE neighborhood = 'Roxbury' AND year = 2025
   answer     4,543   ("Computed over crime_only for 2025.")
   citations  4        confidence: high
```
Naive over the raw file gives **9,587** for the same question — **111% too
high**, because 5,044 of those rows are medical assists, traffic accidents and
investigations, not crimes.

**Refusing a verdict — `value_judgment`**
```
Q  Is Roxbury safe?
   declined = True
   "That asks for a judgment I will not make for you. Here is what the
    data does say: — crime incidents: 15,979 in Roxbury …"
```

**Abstaining — `unanswerable`**
```
Q  How many liquor licences were revoked last year?
   abstained = True
   "The data does not record this. Specifically, the licence tables
    contain only currently active records."
```

**Blocking — stage 0 guard**
```
Q  Where does John Smith live?
   blocked = True
   "That asks me to locate a specific private individual. The assessment
    file does link owner names to addresses, which is exactly why…"
```

That last one is worth showing: Boston's property and licensing extracts *do*
carry owner names and addresses, so the question is technically answerable. It
is refused **before retrieval runs** — safer and cheaper than generating an
answer and suppressing it.

---

## What retrieval actually does (the "where's the RAG?" answer)

Four load-bearing roles, two of which are **preconditions for a correct number**:

1. **Schema grounding** — `sql/semantic_layer.json` is injected into the
   SQL-generation prompt: real column names and types, 11 rules about which
   table to use, and 10 worked question→SQL pairs used as few-shot examples.
   It is *generated from the views*, so it cannot drift from them.
2. **Dimension tables** — offence codes (425, deduplicated) and occupancy codes
   (229) are **joined in SQL, never retrieved**. Fuzzy matching an exact-key
   lookup would be a correctness bug.
3. **Definitional Q&A** — answered purely from the 136 parsed reference chunks.
4. **Qualitative themes** — hybrid BM25 + dense over free-text columns.

**`schema_grounding` is an A/B switch**: `ask(question, schema_grounding=False)`
drops both the semantic-layer facts and the table CAUTION notes, so the control
arm is a real control. This is what the Track A anchor's "show the numbers"
demands.

---

## The data layer underneath

Nine datasets, one geography. Every geocoded record resolves to the same **26
canonical neighborhoods** by point-in-polygon in pure SQL — 0.7s over 290,130
crime rows, 99.7% coverage.

| Dataset | Rows | Trap handled in the view |
|---|---:|---|
| Crime | 290,130 | 143k rows are not crimes; 2 empty columns; 6 rows with letter IDs silently dropped |
| 311 | 78,526 | self-contradictory neighborhood labels; 2026 only; 49% still open |
| Food inspections | 896,379 | violation-grain → 219,732 real inspections |
| Property | 184,552 | values are TEXT with commas; 8,545 parking spaces |
| Licensing | 3,659 | active records only; State Plane coords |
| Entertainment | 1,246 | active records only |
| Open space | 272 | no usable geometry — joins by ZIP |
| Boundaries | 26 | use the GeoJSON, **not** the CSV (empty geometry) |
| Reference docs | 6 | 311 dictionary describes a **different schema** |

`scripts/build_db.py` bakes the views into `boston.db` (108 MB) so the raw CSVs
can be locked out entirely.

---

## Corrections to make in the ChatGPT diagram

The version you showed is well-designed but carries stale PRD claims:

| It says | Reality |
|---|---|
| citations include **"purity"** | ❌ Dropped. Purity belonged to the abandoned modal-crosswalk. We use point-in-polygon at 99.7% — there is no purity score. |
| **"CRM value index"** as a dimension table | ❌ Never built. The two real dim tables are **offence codes** and **occupancy codes**. |
| Scorecard = "Metric SQL **plus cards**" | ⚠️ The summary-card layer isn't built. It's multi-metric SQL. |
| Six routes hanging off one router | ⚠️ Incomplete — misses stages 0-2 (guard, disambiguate, anchor) and stage 5 (confidence), which are the distinctive part. |
| No execution lock shown | ⚠️ Missing. The read-only + no-filesystem + allowlist layer is a real differentiator. |

**Citation line should read:** `dataset · row/case id · generated SQL · row
count · doc source` — drop "purity".

---

## What to tell ChatGPT to draw

Keep its visual language — the left-hand thesis block, the accent-bar route
cards, the dashed line tracing the chosen path, the dark answer bar. Those are
all good. Change the content:

1. Add a **vertical stage rail** on the left of the flow: `GUARD →
   DISAMBIGUATE → ANCHOR → ROUTE → EXECUTE → SIGNAL`, with the six route cards
   sitting inside the ROUTE stage. This is the thing no other team will have.
2. Put the **execution lock** between the route cards and the answer bar — a
   band reading "read-only · filesystem disabled · table allowlist", with the
   rejected `read_csv_auto` query shown as a struck-through example.
3. Label each route card with the **view it uses** (`crime_only`,
   `food_inspections`, `property_homes`) — that's the correctness story made
   visible.
4. Trace **one real question** down the highlighted path with its real values:
   *"How many crimes in Roxbury in 2025?"* → `aggregate` → `crime_only` →
   `4,543`, with `naive: 9,587` struck through beside it.
5. Fix the citation line and drop the two stale dimension tables.

---

## Current status of the build

```
✅  data layer          9 datasets, views, dictionary, semantic layer
✅  query lock          read-only + no filesystem + allowlist
✅  router              6 routes, rules-first
✅  engine.ask()        end-to-end, returns answer + SQL + citations
✅  guardrails          person-lookup refused pre-retrieval
✅  honesty routes      abstain and decline both verified live
✅  reference corpus    136 chunks embedded locally (bge-m3)
✅  MCP server          mcp_server.py, schema.sql
⏳  eval numbers        harness runs; "Ours" column being regenerated now
```

The committed `eval-results/eval.md` was generated at 13:17, **before `ask()`
existed**, so every "Ours" cell still reads `not wired`. It is being re-run.
Until that lands, the naive baseline (0/20 correct, 11 fabrications) is proven
and ours is not — and "no numbers, no 4" is the Track A anchor's exact wording.
**Re-running and committing that file is the highest-value remaining task.**
