# Showcase Project 02 — `finance-dashboard-powerbi`

> An executive Power BI dashboard for fee collection, dues aging, and ledger analytics — with full SQL source, DAX, and synthetic data.

---

## One-liner

A Power BI report institutions could actually use: tracks fee collection, dues aging, and concentration risk across programs — built on a star-schema SQL Server source with measures source-controlled in DAX.

---

## Why this project (recruiter angle)

- Visible — recruiters can *see* it without running code
- Demonstrates the **full BI stack**: SQL source → semantic model → DAX → visuals
- Shows business judgment (which KPIs matter, which don't)
- Bridges your dev + analytics skills in one repo

---

## The 5 questions the dashboard answers

Every page maps to one executive question:

1. **Are we collecting fees on time?** → collection-rate page
2. **Which programs / cohorts have rising dues?** → dues-aging page with drill-down
3. **Where is concentration risk?** → top-N contributors page
4. **What's the cash trend?** → time-series + forecast page
5. **Are there anomalies?** → outlier page (collections vs invoices delta)

---

## Pages & visuals (final state)

| Page | Visuals | Drill-through |
|------|---------|---------------|
| 1. Overview | 4 KPI cards + monthly trend + program bar + status donut | → page 2 |
| 2. Collections by program | Stacked bar by program & month + period slicer | → page 3 |
| 3. Dues aging | Aging bucket table (0-30/31-60/61-90/90+) + heatmap | → page 4 |
| 4. Top contributors | Pareto bar + concentration % indicator | — |
| 5. Anomalies | Variance scatter + flag list | → page 6 |
| 6. Detail | Transaction table + filters | — |

---

## KPIs (DAX)

```dax
-- Collection rate
Collection Rate :=
DIVIDE(
    [Total Collected],
    [Total Invoiced],
    BLANK()
)

-- Total Collected
Total Collected := SUM(fact_payments[amount])

-- Total Invoiced
Total Invoiced := SUM(fact_invoices[amount])

-- Dues outstanding > 90 days
Aged Dues 90+ :=
VAR Today = MAX(dim_date[date])
RETURN
CALCULATE(
    SUM(fact_invoices[balance]),
    FILTER(
        fact_invoices,
        fact_invoices[balance] > 0
        && DATEDIFF(fact_invoices[invoice_date], Today, DAY) > 90
    )
)

-- MoM growth
MoM Collection Growth :=
VAR CurrentMonth = [Total Collected]
VAR PrevMonth = CALCULATE([Total Collected], DATEADD(dim_date[date], -1, MONTH))
RETURN DIVIDE(CurrentMonth - PrevMonth, PrevMonth)

-- Top-10 contributor share
Top10 Share :=
VAR Top10 =
    TOPN(
        10,
        VALUES(dim_program[program]),
        [Total Collected],
        DESC
    )
VAR Top10Sum = CALCULATE([Total Collected], Top10)
RETURN DIVIDE(Top10Sum, [Total Collected])
```

All measures committed in `dax/measures.dax`.

---

## Data model (star schema)

```
                   ┌─────────────┐
                   │  dim_date   │
                   └──────┬──────┘
                          │
   ┌──────────┐   ┌───────┴────────┐   ┌──────────────┐
   │ dim_     │   │  fact_invoices │   │ dim_program  │
   │ student  │───┤                ├───│              │
   └──────────┘   └───────┬────────┘   └──────────────┘
                          │
                  ┌───────▼────────┐
                  │ fact_payments  │
                  └────────────────┘
```

- `dim_date`: marked as date table, 5-year span
- `dim_student`: SCD Type 2 for program changes
- `dim_program`: stable lookup
- `fact_invoices`: invoice + balance grain
- `fact_payments`: payment-line grain

---

## Folder structure

```
finance-dashboard-powerbi/
├── pbix/
│   └── finance-dashboard.pbix
├── dax/
│   ├── measures.dax                # all measures, source-controlled
│   ├── calculated-columns.dax
│   └── time-intelligence.dax
├── sql/
│   ├── schema/
│   │   ├── 001-dimensions.sql
│   │   └── 002-facts.sql
│   ├── views/                      # what Power BI connects to
│   │   ├── vw_fact_payments.sql
│   │   └── vw_fact_invoices.sql
│   └── seed/
│       └── seed-synthetic.sql      # 50k students, 24 months of data
├── theme/
│   └── finance-theme.json
├── docs/
│   ├── data-model.md
│   ├── kpi-dictionary.md
│   ├── refresh-strategy.md
│   └── img/
│       ├── p1-overview.png
│       ├── p2-collections.png
│       ├── p3-aging.png
│       ├── p4-top-contributors.png
│       └── erd.png
├── samples/
│   └── synthetic-data.csv          # fallback if SQL not available
├── .gitattributes                  # *.pbix filter=lfs
├── .gitignore
├── LICENSE
└── README.md
```

---

## Build plan (1-2 weekends)

### Weekend 1 — data + model
- [ ] Create local SQL Server DB
- [ ] Apply schema scripts
- [ ] Run seed script → 50k synthetic students, 24mo of data
- [ ] Connect Power BI Desktop to SQL views
- [ ] Build star schema in model view
- [ ] Mark date table
- [ ] Write all 12 measures

### Weekend 2 — visuals + polish
- [ ] Build 6 pages following the visual list above
- [ ] Apply custom theme (`theme/finance-theme.json`)
- [ ] Configure drill-through paths
- [ ] Export measures to `dax/` via Tabular Editor
- [ ] Take screenshots
- [ ] Write README + KPI dictionary
- [ ] Publish read-only version to Power BI Service free tier → embed link

---

## Design principles applied (interview talking points)

| Principle | What I did |
|-----------|------------|
| **One question per page** | Every page maps to one exec question |
| **Variance always shows ±** | MoM cards use up/down arrows + color |
| **Time flows left-to-right** | All trend charts oriented L→R |
| **Drill-through, not click-through** | One-way navigation paths |
| **Cognitive load budget** | Max 7 visuals per page |
| **Single semantic palette** | Brand color + 4 neutrals — no rainbow |
| **Tooltips do work** | Hover gives the "why" without crowding the visual |

---

## Cleaning the .pbix for commit

Power BI files easily leak data. Before every commit:

1. **Open the report**
2. **Home → Transform data → Data source settings** → point to synthetic dataset
3. **Refresh** until visuals all show synthetic numbers
4. **File → Options → Data Load → Privacy → Combine data across all sources** (forces re-cache)
5. **Save**
6. **Verify**: open the .pbix in a text editor's hex view OR use [pbi-tools](https://github.com/pbi-tools/pbi-tools) to extract and grep for stray strings
7. Track with LFS:
   ```bash
   git lfs install
   git lfs track "*.pbix"
   git lfs track "*.pbit"
   git add .gitattributes
   ```
8. Commit

---

## Stretch features

- Publish to Power BI Service free → embed iframe in README
- Add a row-level security (RLS) demo (role: `Director` sees all, `Program Head` sees own program only)
- AI Insights / Anomaly detection visual
- Implement bookmarks for guided tour
- Add a paginated report (Power BI paginated → PDF export)
