# Where to pick this up

Last touched 22 August 2026. Written while everything was still fresh, so future me
does not have to reverse engineer any of it.

---

## What this is

**Boston Neighborhood Intelligence** — a question answering engine over Boston's open
city data. Ask about safety, city responsiveness, food safety, parks or housing across
the 26 neighbourhoods, and it answers with the SQL it ran and the sources it used, or
it tells you honestly that it cannot answer.

It won **Track A ("The Engine")** at the RAG the City hackathon, The Open Accelerator,
Boston, 22 August 2026.

Built by a team of five in a single 4.5 hour window (10:45 to 15:15). The full git
history shows who committed what.

**What I personally built:** the data and SQL layer (`sql/views.sql`,
`data/DATA_DICTIONARY.md`), the semantic layer that grounds SQL generation
(`scripts/build_semantic_layer.py`, `sql/semantic_layer.md`), the query isolation layer
(`scripts/query.py`, `scripts/build_db.py`), the reference document parsing
(`scripts/parse_docs.py`), and the final presentation.

Keep that distinction visible in the README when this goes on a CV. It is a team
project and the history is public, so claiming the whole thing would be both wrong and
easy to catch. Claiming the data layer is accurate and still impressive.

---

## Why the architecture is worth keeping

This is the part that makes it a good portfolio piece rather than just another RAG demo.

**Six-stage progressive disclosure**, not a single router. Each stage removes a class of
failure before the next runs:

```
0  guard         refuse person-directed lookups BEFORE retrieval runs
1  disambiguate  resolve place names, surface ambiguity rather than guess
2  anchor        pin an explicit time window
3  route         pick one of six engines, rules only, no model call
4  execute       SQL for numbers, retrieval for meanings, abstain for gaps
5  signal        attach an explicit confidence level
```

Stages 0 to 3 are pure rules, so routing is instant and explainable. Every stage is
recorded in `Answer.trace`.

**Correctness by construction.** Generated SQL only ever runs through
`scripts/query.py`: read-only connection, `enable_external_access = false`, table
allowlist. The raw CSVs are physically unreachable, so a query that tries to route
around the cleaned tables raises a Permission Error instead of returning a wrong number.

The pitch line: *we did not tell the model the rules and hope it followed them, we made
the wrong query impossible to write.*

**Data traps fixed in SQL views, not in prompts.** A prompt full of rules is a
suggestion and a 7B model will eventually drop one. Every known trap is handled inside
`sql/views.sql` so the model can write naive SQL and still get a correct answer.

**Three distinct honest non-answers**, scored separately: `blocked` (person lookup),
`abstained` (the data does not record it), `declined` (a value judgement I will not
issue). Citations are enforced in code, so a grounded answer without one raises before
it can reach the user.

---

## The numbers that tell the story

Every one of these was measured against the committed extracts. They are the reason the
project exists.

| Question | Naive answer | Correct |
|---|---:|---:|
| Most valuable property in Boston | $999,900 | $2,448,193,300 |
| Crime incidents | 290,130 | 146,933 |
| Restaurant inspections | 896,379 | 219,732 |
| Parkland, acres | 2,327 | 5,861 |
| Crimes in Roxbury, 2025 | 9,587 | 4,543 |

The property one is the best single example: assessed values are stored as text with
commas, so `max()` sorts them alphabetically. Nothing errors. You get a confident,
citable answer that is wrong by a factor of 2,450.

Nearly half the rows in the crime file are not crimes. They are medical assists,
traffic accidents, sudden deaths and police investigations.

---

## FIX THIS FIRST

**`eval/fill_gold.py` grades the engine against a methodology the project abandoned.**

Line ~41 computes gold using `DISTRICT='B2'` as a proxy for Roxbury. That is the
district-purity crosswalk, which is only 65.9% pure. Line ~99 uses 311's own
`neighborhood` column, which contradicts itself and which `DATA_DICTIONARY.md`
explicitly says never to use.

The engine resolves geography by point-in-polygon against the BPDA boundaries at 99.7%
coverage. So **the engine is more accurate than its own gold standard**, and the
committed eval wrongly reports 0/7 on counting questions.

Verified correct values, computed from the views:

```
R02  violent crime, Roxbury 2024    636      (gold says 757)
T01  all crime, Roxbury 2024        4,489    (naive 9,097)
M05  311 Allston + Brighton         7,041    (gold says 7,005)
M04  311 rodent complaints          3,537    (already correct)
```

**The fix:** recompute gold from `boston.db`'s views (`crime_only`, `svc311`,
`property_homes`) instead of `read_csv_auto` over the raw files with district proxies.
Roughly a 20 line change. Then re-run `eval/run_eval.py` and commit the result.

This matters because the Track A rubric's exact words are *"measurably better than
naive RAG, and they can show the numbers."* Right now the committed file undersells the
system badly, and it is the first thing a technical reader will open.

---

## Everything else worth fixing, in order

1. **Fix the gold values** (above). Nothing else matters as much.
2. **Re-run the eval and commit real numbers.** The naive baseline is already proven at
   0% accuracy with 11 fabrications. Ours should look dramatically better once the gold
   is right.
3. **Run the `--no-schema-grounding` control arm.** It was never executed. This is the
   A/B measurement the rubric actually asked for, and it is a single flag:
   `python eval/run_eval.py --no-schema-grounding`.
4. **Investigate retrieval hit rate @5 showing 0%.** Almost certainly a format mismatch
   between the `retrieved_ids` the engine emits (`dataset#locator`) and what the harness
   matches against, not a real retrieval failure. Check `engine.py` against
   `eval/run_eval.py`.
5. **Remove `geopandas` from `requirements.txt`.** It is listed but never imported.
   Geography is pure DuckDB spatial. An unused heavyweight dependency looks careless.
6. **Finish the scorecard route.** It currently runs multi-metric SQL with no summary
   card layer. Either build the cards or rename the route to match what it does.
7. **Add a demo GIF or screenshots to the README.** The Starlette UI at
   `static/index.html` is genuinely nice and nobody can see it from a text README.
8. **Write the attribution section** described at the top of this file.

---

## How to get it running again

Everything local was deleted on 22 Aug 2026 to reclaim ~19 GB. Nothing is lost, it all
rebuilds.

```bash
# 1. dependencies — use Python 3.12, NOT 3.14 (no wheels for this stack)
python3.12 -m venv .venv
.venv/bin/pip install -r requirements.txt

# 2. models, about 11 GB
ollama pull qwen2.5-coder:7b     # writes the SQL
ollama pull granite3.1-dense:8b  # writes grounded prose
ollama pull bge-m3               # embeddings, multilingual

# 3. build the database, about a minute
.venv/bin/python scripts/build_db.py

# 4. check it works
.venv/bin/python -m pytest tests/ -q          # expect 179 passing
.venv/bin/python scripts/smoke_engine.py      # one question per route

# 5. run it
./run_web.sh    # demo UI   -> http://127.0.0.1:8000
./run_mcp.sh    # MCP server -> http://127.0.0.1:3000/mcp
```

Quick sanity check that the whole pipeline is alive:

```python
from engine import ask
ask("How many crimes in Roxbury in 2025?")["answer"]   # -> '4,543'
```

Total time from clone to running: about fifteen minutes, most of it model downloads.

---

## The stack

All local, no cloud APIs, open source end to end.

| Layer | What |
|---|---|
| SQL | DuckDB 1.5.5 + spatial extension, 1.67M rows, no sampling |
| Retrieval | ChromaDB (dense) + rank_bm25 (sparse), fused by Reciprocal Rank Fusion, k=60 |
| Models | `qwen2.5-coder:7b` SQL, `granite3.1-dense:8b` prose, `bge-m3` embeddings, all via Ollama |
| Interfaces | Starlette demo UI (no CDN, works offline), MCP server via fastmcp |
| Analysis | scikit-learn, KNN recovered 17,349 missing 311 ZIP codes at 98.31% |
| Documents | pdfplumber, openpyxl, parsed once and committed |
| Testing | pytest, 179 tests |

Sparse search is not decoration. A figure like `4200000` sits in a CSV field with no
currency symbol or label, so dense retrieval has no semantic neighbourhood to find it
in. BM25 finds it on the digits.

---

## Data

Nine sources from Analyze Boston plus six of the city's own reference documents. Every
geocoded record resolves to the same 26 BPDA neighbourhood polygons by point-in-polygon
in pure SQL, 290,130 crime rows assigned in under six seconds at 99.7% coverage.

The reference PDFs are in `docs/refdocs/` if they were copied across, otherwise they are
a pain to re-source from data.boston.gov. Worth keeping.

One trap worth remembering: the city's own 311 data dictionary documents 28 columns and
**only 10 of them exist in the actual file**. The schema was built from the data, not
the documentation. Never feed that PDF into SQL generation.

---

## How to frame this on a CV

Lead with the correctness delta, not the architecture. The architecture is what makes
it good, but the delta is what makes someone care.

Something like:

> Built the data and SQL layer for a RAG engine over Boston open data that won Track A
> at the RAG the City hackathon. Found and fixed four classes of silent data corruption
> (text-typed currency columns, violation-grain vs inspection-grain, non-crime rows in
> the crime file, geometry-less boundary files) that caused a naive pipeline to report
> Boston's most valuable property as $999,900 instead of $2.4 billion. Encoded every
> fix in SQL views and locked the raw files out of the query path, so wrong queries
> error instead of returning wrong numbers.

That paragraph works because every claim in it is checkable in the repo.
