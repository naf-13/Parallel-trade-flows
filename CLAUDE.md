# EU Parallel Distribution Trade Flow Analysis — Project Plan

## 1. Project Summary

Build an end-to-end data engineering + data science project that extracts, models, and analyzes the EMA's public register of parallel distribution notices for centrally authorised medicines across the EU/EEA. The core deliverable is a trade-flow analysis of which countries medicines are parallel-distributed *from* and *to*, how this has changed over time, and (optional stretch) whether heavy export activity correlates with domestic shortage risk in source countries.

**Why this project:** real, regulator-published data; no invented ground truth; genuine data engineering challenge (the source isn't a clean API); produces a visually strong, explainable output; directly relevant to commercial pharma supply chain.

**Skills this builds:** web scraping / API reverse-engineering, ETL pipeline design, data modeling, network/flow visualization, (optional) basic causal/correlational analysis, packaging a project properly (Docker, docs, tests).

---

## 2. Data Source

**Primary:** EMA Register of Parallel Distribution Notices — https://iris.ema.europa.eu/registerpd/
Confirmed fields available via the UI filters:
- Product Name, Strength, Pack Size
- Notice Number, EU Number
- Parallel Distributor, Marketing Authorisation Holder, Repackagers
- Domain (Human/Veterinary)
- Pharmaceutical Form
- Status (Active / Dormant / Withdrawn / Cancelled / Suspended)
- Date of Notice, Date of Decision
- **Member State(s) of Origin**
- **Member State(s) of Destination**

**Known constraint:** this is a PowerApps/Dynamics portal — data loads via JS/backend calls, not present in raw HTML. Phase 1 is specifically about solving this.

**Optional secondary source (stretch goal, Phase 5):** a national shortage register (e.g. a country known for parallel-export-linked shortages) to test the export-volume-vs-shortage-risk angle. To be identified and validated later — not a Phase 1 dependency.

---

## 3. Phased Plan

### Phase 0 — Setup (½ day)
- Repo scaffold, README stub, virtual environment, git init
- Decide tech stack (see Section 4)
- Document the project's scope and honest limitations up front (this becomes part of your README later — start it now, add to it as you go)

### Phase 1 — Data Access (the real engineering challenge)
**Goal:** reliably extract records from the register, not just view them in a browser.

Steps:
1. Open the register in a browser with DevTools Network tab open, apply a filter, and inspect what request fires — Dynamics/PowerApps portals typically expose an OData or Web API endpoint (e.g. something under `/_api/` or similar) once a query executes.
2. If a usable API endpoint is found: replicate the request in Python (`requests`), figure out pagination and auth/session requirements.
3. If no usable API is exposed (fully JS-rendered, session-gated): fall back to browser automation (Playwright is more reliable than Selenium for modern portals) to drive the filter UI and scrape rendered results.
4. Build in politeness: rate limiting, retries, caching of raw responses so you're not hammering EMA's infrastructure during development.
5. Land raw extracted data as-is (JSON or HTML snapshots) before any cleaning — keep raw and processed data separate from day one.

**Deliverable:** a script that can pull the full (or a defined subset of) register into raw local files.

**Honest risk to flag now:** if the portal actively blocks automation (CAPTCHA, bot detection), this phase could stall. If that happens, the fallback is a smaller manually-assisted extraction (e.g., export a defined date range via the UI's own export feature if one exists) — worth checking whether the register has a built-in "export to Excel/CSV" button before assuming full automation is required.

### Phase 2 — Data Modeling & Cleaning
- Design a clean schema: one row per notice, with normalized country codes (ISO 3166) for origin/destination, parsed dates, standardized status
- Handle multi-value fields (a notice can have multiple origin *and* multiple destination states — decide whether to explode into origin-destination pairs, since that's what a flow analysis needs)
- Load into a proper store — SQLite is enough to start, Postgres if you want to demonstrate more
- Basic data quality checks (nulls, duplicate notices, date sanity)

**Deliverable:** a clean, queryable dataset + a documented schema.

### Phase 3 — Exploratory Analysis
- Volume of notices over time (by year, maybe by quarter)
- Top origin countries, top destination countries
- Net flow by country (exports minus imports) — this is the "who's a source, who's a destination" story
- Top products/therapeutic areas by parallel distribution volume (if therapeutic classification is derivable — may need to cross-reference product names against an external classification like ATC)
- Status breakdown (how many notices go dormant/withdrawn vs. stay active — tells you something about churn in this market)

**Deliverable:** a well-documented analysis notebook with real findings, not just charts for their own sake.

### Phase 4 — Visualization
- Country-to-country flow visualization — Sankey diagram or a network graph (NetworkX + a plotting layer, or a dedicated Sankey library) is the natural fit for origin→destination data
- Time-series view of flow evolution
- Optional: a simple interactive dashboard (Streamlit) so it's not just static notebook output — lets a viewer filter by year/country/product

**Deliverable:** the visual centerpiece of the GitHub repo — this is what people will screenshot.

### Phase 5 — Stretch: Shortage Correlation (optional, do only if Phases 1–4 land well)
- Identify a country with both meaningful parallel-export volume in your dataset and a public shortage register
- Test, honestly and with appropriate caveats, whether periods/products with high export volume show elevated shortage activity in that country
- This section must be framed as exploratory/correlational, not causal — be explicit about confounders (this is the same discipline we established earlier: state what the data can and can't tell you)

### Phase 6 — Packaging
- Dockerize the pipeline (extract → clean → load → analyze) so it runs end-to-end with one command
- Write a proper README: problem statement, data source and its limitations, architecture diagram, how to run it, sample outputs
- Add basic tests for the cleaning/transformation logic
- (Optional) schedule it (cron / GitHub Actions) so the dataset stays fresh — nice engineering flex, not required for v1

---

## 4. Suggested Tech Stack

| Layer | Tool | Notes |
|---|---|---|
| Extraction | `requests` or `playwright` | Depends on what Phase 1 discovers |
| Storage | SQLite → Postgres (optional upgrade) | Start simple |
| Transformation | `pandas` | Your comfort zone, fine for this scale |
| Analysis | `pandas`, `networkx` | For flow/network structure |
| Visualization | `plotly` (Sankey + interactivity) | Renders well in a README/GitHub via saved images, or live in Streamlit |
| Dashboard (optional) | `streamlit` | Fast to stand up |
| Packaging | `Docker`, `docker-compose` | One-command reproducibility |
| Testing | `pytest` | Basic coverage on transform logic |

---

## 5. Repo Structure (proposed)

```
parallel-distribution-flows/
├── README.md
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── src/
│   ├── extract/          # Phase 1 scraping/API logic
│   ├── transform/        # Phase 2 cleaning + schema
│   ├── load/              # DB loading
│   └── analysis/          # Phase 3 analysis functions
├── notebooks/
│   └── exploratory_analysis.ipynb
├── dashboard/
│   └── app.py              # Streamlit, if built
├── data/
│   ├── raw/                # gitignored
│   └── processed/          # gitignored, or small sample committed
├── tests/
└── docs/
    └── data_limitations.md # honesty section, referenced from README
```

---

## 6. Definition of Done (v1)

A stranger can clone the repo, run one command, and see:
1. The pipeline pull (or a cached sample of) the parallel distribution register
2. A cleaned dataset land in a local database
3. A generated flow visualization showing which EU countries medicines move between
4. A README that honestly explains what the data is, where it comes from, and what its limitations are

Everything past that (Phase 5 correlation, dashboard, scheduling) is enhancement, not requirement.

---

## 7. Immediate Next Step

Phase 1, Step 1: open the register in a browser, apply a filter, and inspect the network request that fires — this determines whether the rest of the project is a clean API pipeline or a browser-automation pipeline. This can't be done from within this chat (the sandbox here can't reach `iris.ema.europa.eu`), so it's the first thing to do locally before we write any code together.
