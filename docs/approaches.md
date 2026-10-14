# Charts from unknown-schema tool data: approach comparison

**Problem.** An agent (Python, OpenAI SDK) calls an external MCP server we don't control. A single
tool response can carry up to 100k data points in any shape. The agent must turn that into a
high-quality interactive visualization (not just bar charts: animations, linked views, drill-down
workflows).

**Constraints.** Token-efficient, good end-user UX, robust to messy data, open source where possible,
about $3 of OpenAI credit for the whole project.

## The core fact that drives the design

100k points as JSON is roughly 1.5M to 5M tokens. That is more than any context window and would cost
dollars per request. So **the raw payload can never pass through the model**. The model only needs to
know *what the data looks like*, not the data itself. Every approach below is really an answer to
"how does the full data reach the chart without going through the LLM?"

## Candidate approaches

### A. Model writes a chart spec from a preview
Tool output is truncated/sampled into context; the model writes a Vega-Lite or ECharts spec.

- **Tokens:** OK for small payloads. If the model has to inline data in the spec, cost scales with data and fails beyond a few hundred rows.
- **UX:** Fast and pleasant when it works.
- **Edge cases:** Weak. A sample hides rare keys, mixed types and the true value range. Nested JSON has to be understood by the model every time.
- **Verdict:** Fine baseline, does not meet the 100k requirement on its own.

### B. Model writes code that runs in a sandbox over the full data
The model sees a preview, writes Python (pandas + Plotly/Altair) or JS (D3), which executes against the full payload.

- **Tokens:** Output-heavy (1k to 4k tokens of code), plus retries when code errors. Medium.
- **UX:** Highest ceiling: anything is possible, including custom animations.
- **Edge cases:** The model must re-solve parsing/cleaning in code every time; failures surface as stack traces. Needs a real sandbox (code execution is a security surface). Output HTML can be huge if data is embedded naively.
- **Verdict:** Powerful but expensive, slow and flaky as the *default* path.

### C. Data stored by reference; chart tool takes a handle plus a mapping
The harness intercepts the tool result, stores it, and gives the model a handle and schema. The model calls `render_chart(handle, type="line", x="ts", y="value", color="region")`.

- **Tokens:** Lowest. A few hundred tokens regardless of data size.
- **UX:** Fast, deterministic, consistent styling.
- **Edge cases:** Good for parsing (done once, in code), but expressiveness is capped by the chart templates we write. "Custom workflows and animations" don't fit a flat parameter list.
- **Verdict:** Right data plane, too narrow a control plane.

### D. Pure auto-viz (no LLM in the loop)
Heuristics/recommender (in the spirit of Lux, Voyager, Data Formulator) pick charts from the profile.

- **Tokens:** Zero.
- **UX:** Decent defaults, but cannot follow user intent ("show how churn evolves per region over time as an animation").
- **Verdict:** Not standalone, but a useful component: it can propose candidates the model picks from.

### E. Model writes a full HTML/JS app that fetches data by URL
Generative UI: the model writes a complete page (D3/ECharts) that loads data from a local endpoint.

- **Tokens:** Highest output cost (3k to 10k tokens per page).
- **UX:** Most bespoke results; also the most variable quality.
- **Verdict:** Good as an escape hatch, too costly as the default.

## Recommended: hybrid "data plane / control plane" split

Combine C's data handling, a declarative spec richer than C's parameter list, D's candidates as hints, and B/E only as an escape hatch.

```
 MCP server ──► Interceptor ──► Normalizer ──► DuckDB (full data, by handle)
 (mock)            │                                  │
                   ▼                                  │
             Profile (~300-800 tokens)                │
                   │                                  │
                   ▼                                  ▼
 User ──► LLM ──► viz spec (no data, refs + SQL) ──► Renderer ──► self-contained interactive HTML
            ▲            │                            (downsample, validate, theme)
            └── short structured errors ◄─────────────┘
```

1. **Interceptor.** Wraps the MCP client. Every tool result goes to the data plane, never straight to the model. Small results (say under 2k tokens) can still be passed through verbatim, since that's cheaper than a round trip.
2. **Normalizer (deterministic Python).** Finds the record arrays inside arbitrary JSON (JSONPath walk), flattens nested objects (`user.address.city`), explodes or links child arrays as separate tables, detects CSV/TSV/markdown tables inside strings, coerces types (`"$1,200"`, `"45%"`, epoch vs ISO dates, `"N/A"` as null), and records what it did. Output: one or more DuckDB tables per handle.
3. **Profile for the model.** Per table: row count, columns with inferred semantic type (temporal, quantitative, nominal, geo, id, text), null %, cardinality, min/max or top values, 3 sample rows, plus detected shapes (time series, graph with nodes/edges, hierarchy, geo points). Size is fixed regardless of 10 rows or 100k. Wide tables (hundreds of columns) get a ranked subset plus a `describe_columns` tool.
4. **Optional `query(handle, sql)` tool.** Lets the model look at small aggregates before deciding (e.g. "which regions exist?"). Results are capped to a few rows.
5. **Viz spec.** The model writes an ECharts-flavoured JSON spec where data is a reference, not values: `{"dataset": {"handle": "h1", "sql": "SELECT month, region, sum(rev) ..."}, "series": [{"type": "line", "encode": {"x": "month", "y": "rev"}}], "timeline": {...}}`. ECharts' own `dataset` + `encode` model fits this exactly, LLMs know it well, and it covers animation (timeline, universal transitions), linked brushing, data zoom, treemap/sunburst/sankey/graph/geo, and large-data modes (`large`, `progressive`, WebGL via echarts-gl).
6. **Renderer (deterministic).** Validates field names and types against the profile and returns short, fixable errors (`unknown column 'revenue'; did you mean 'rev'?`) with a retry cap of 2. Executes the SQL in DuckDB, applies **downsampling by chart type** (LTTB for lines, binning/hexbin for dense scatter, top-k + "other" for categories, aggregation for bars), injects the result into a themed HTML template, and emits one self-contained file.
7. **Workflows.** A "dashboard" spec is just several specs plus interactions (filter, drill-down, linked brush) the renderer wires up. Multi-step workflows become an ordered list of views with transitions.
8. **Escape hatch.** For something the spec cannot express, the model may write a small JS function against a constrained render API (`render(el, data, echarts)`), with data still resolved by handle. Used rarely, so its token cost stays rare too.

### Why this wins on the three criteria

| | A. Spec from preview | B. Sandbox code | C. Handle + mapping | **Hybrid** |
|---|---|---|---|---|
| Tokens per chart (approx.) | 2k to 20k+, grows with data | 3k to 10k, plus retries | ~1k | **~2k to 4k, flat** |
| Works at 100k points | No | Yes | Yes | **Yes** |
| Expressiveness | Medium | Very high | Low | **High (escape hatch for the rest)** |
| Determinism / safety | Medium | Low (runs model code) | High | **High** |
| Messy-data handling | Model, every time | Model, every time | Code, once | **Code, once** |
| Latency | Low | High | Low | **Low** |

With a small OpenAI model (gpt-4o-mini / gpt-4.1-mini class), ~3k input + ~800 output tokens is a
fraction of a cent per chart, so $3 covers hundreds of test runs. Most of the system (mock server,
normalizer, profiler, renderer) can be built and tested with hand-written specs at zero API cost.

## Edge cases the mock server should throw at it

| Case | Handling in the hybrid |
|---|---|
| Records nested deep inside wrappers (`{"data": {"items": [...]}}`) | JSONPath discovery of the largest record arrays |
| Ragged keys across records | Union schema, missing becomes null, null % shown in profile |
| Mixed types in one field (`12`, `"12"`, `"N/A"`, `null`) | Majority-type coercion, failures counted in profile |
| Units and formats in strings (`"$1.2k"`, `"45%"`, `"3 days"`) | Parsers with recorded unit |
| Timestamps: ISO, epoch s/ms, `"Oct 3"` | Detection + normalization to UTC |
| CSV / markdown table embedded in a text field | Detected and parsed into its own table |
| Several unrelated arrays in one response | Multiple tables under one handle |
| Graph data (nodes + edges) | Detected shape, graph/sankey suggested |
| Hierarchy (tree of children) | Detected shape, treemap/sunburst suggested |
| Geo (lat/lon pairs, GeoJSON) | Detected shape, map suggested |
| Single number / tiny result | KPI tile, or pass through without a handle |
| Pure text, no chartable data | Profile says so; agent answers in text |
| 100k rows, high-cardinality category | Downsampling + top-k, never sent to model |
| 500+ columns | Ranked column subset + `describe_columns` tool |
| Empty response / tool error | Explicit status in profile, no chart attempted |

## Proposed build order (once a direction is chosen)

1. Mock MCP server using the official open-source `mcp` Python SDK (FastMCP), with ~10 tools covering the edge cases above, including 100k-row generators. No API cost.
2. Interceptor + normalizer + profiler, unit-tested against the mock tools. No API cost.
3. Spec schema + renderer (ECharts, DuckDB, LTTB), tested with hand-written specs. No API cost.
4. Agent loop with the OpenAI SDK, small model, ~20 scripted prompts. Log tokens and cost per run and stop at a fixed budget (e.g. $0.50 for this phase).
5. Optional: implement approach B on the same mock data as a measured comparison.

**Open-source stack:** `mcp` (Python SDK), DuckDB, pandas/pyarrow, Apache ECharts (+ echarts-gl), Jinja2 for templating, pydantic for spec validation. Mosaic (UW IDL) is a strong alternative renderer for linked views over DuckDB if ECharts' interactions turn out to be limiting.
