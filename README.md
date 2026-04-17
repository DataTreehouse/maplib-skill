# maplib Skill — Benchmark Report

**Date**: 2026-04-17  
**Executor model**: Claude Opus 4.6  
**Method**: 3 test prompts, each run with and without the skill (6 runs total). Outputs graded programmatically against 8 assertions per test case (24 assertions total).

## Summary

| Metric | With skill | Without skill | Delta |
|--------|-----------|---------------|-------|
| Pass rate | **100%** (24/24) | **29%** (7/24) | **+71 pp** |
| Time | 20.9s | 19.7s | +1.2s |
| Tokens | 30,470 | 25,611 | +4,859 |

The skill's overhead is ~5k tokens and ~1 second per run — the cost of loading and following the skill context. In exchange, every baseline run hallucinated a nonexistent maplib API, while every with-skill run produced correct code.

## Test cases

### Eval 0 — DataFrame to knowledge graph + SPARQL query

**Prompt**: *"I have a Polars DataFrame of employees with columns id, name, department, hire_date. Show me how to turn it into an RDF knowledge graph with maplib, then write a SPARQL query that returns everyone in the Engineering department hired after 2020-01-01."*

| Assertion | With skill | Baseline |
|-----------|:---:|:---:|
| Imports `Model` from maplib | pass | fail |
| Uses correct `Model()` instantiation | pass | fail |
| Uses `add_template` method | pass | fail |
| Uses `map` method (not `expand`) | pass | fail |
| Uses `query` method on Model | pass | pass |
| Does NOT fall back to rdflib | pass | fail |
| Does NOT invent fake maplib classes | pass | fail |
| Uses OTTR template with `ottr:Triple` | pass | fail |

**Baseline failure mode**: Invented `Mapper`, `RDFStore`, and `Graph` imports from maplib, then silently fell back to raw rdflib with manual triple construction in a for-loop.

### Eval 1 — SHACL validation flow

**Prompt**: *"I have an RDF graph of people with age values. Some entries have ages over 150 which are clearly wrong. Using maplib, show me how to write a SHACL shape that catches this, validate the graph, and print the violations."*

| Assertion | With skill | Baseline |
|-----------|:---:|:---:|
| Imports `Model` from maplib | pass | fail |
| Uses `add_template` for data shape | pass | fail |
| Uses `validate()` method | pass | pass |
| References `report.conforms` | pass | fail |
| Calls `report.results()` | pass | fail |
| Defines SHACL shape with `sh:NodeShape` | pass | pass |
| Uses `sh:maxInclusive` for age bound | pass | pass |
| Does NOT invent fake maplib classes | pass | fail |

**Baseline failure mode**: Invented `maplib.KnowledgeGraph()` with a `.parse()` method and a `.validate()` that returns a list of violation objects with `.focus_node`, `.shape`, `.message` attributes — none of which exist.

### Eval 2 — Join two DataFrames in a knowledge graph

**Prompt**: *"I have two Polars DataFrames: products (id, name, price) and reviews (product_id, score, reviewer). Using maplib, build a knowledge graph where reviews link to products, then write a SPARQL query to find products with an average review score above 4.0."*

| Assertion | With skill | Baseline |
|-----------|:---:|:---:|
| Imports `Model` from maplib | pass | fail |
| Defines template for Product | pass | fail |
| Defines template for Review | pass | fail |
| Calls `map` twice (one per template) | pass | fail |
| SPARQL uses `AVG` aggregate | pass | pass |
| SPARQL uses `GROUP BY` | pass | pass |
| SPARQL uses `HAVING` with > 4.0 | pass | pass |
| Does NOT invent fake maplib classes | pass | fail |

**Baseline failure mode**: Invented `MapLibGraph` and `MapLibStore` classes with an `add_triple()` method, manually iterating over DataFrame rows in a for-loop instead of using templates.

## Assertions explained

Each test case was graded against 8 programmatic checks, divided into two categories:

**Correct API usage** — Does the generated code use the actual maplib API? This includes importing `Model`, calling `add_template()`, `map()`, `query()`, and `validate()` correctly. These assertions exist because LLMs without the skill consistently hallucinate plausible-sounding but nonexistent classes and methods.

**Architectural correctness** — Does the code follow maplib's template-based paradigm? This means using OTTR templates with `ottr:Triple` patterns, passing DataFrames through `map()` rather than iterating row-by-row, and using SPARQL for graph queries. The baselines consistently fell back to manual triple construction (rdflib-style for-loops), which defeats the purpose of maplib entirely.

## Observations

The baselines are not just slightly wrong — they are fundamentally broken. Each one invents a different fantasy API (`Mapper`/`RDFStore`, `KnowledgeGraph`, `MapLibGraph`/`MapLibStore`), none of which exist in maplib. This is consistent with the fact that maplib is a relatively niche library and LLMs have limited training data on it.

The skill eliminates this problem completely. All three with-skill runs produce code that follows the canonical `Model` → `add_template` → `map` → `query` workflow, uses correct OTTR template syntax, and returns Polars DataFrames from SPARQL queries.

The only trade-off is ~5k additional tokens per invocation (the cost of loading the skill into context), which adds roughly 1 second of wall time. For a library where the baseline is 100% hallucinated APIs, this is a straightforward win.
