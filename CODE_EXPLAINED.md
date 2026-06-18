# Code Explained — IE Airlines Group 3

This document explains what each Python file does in plain language.
No prior coding knowledge needed.

---

## The Big Picture

The project has **three Python files**, each with a clear job:

```
db.py  ──pulls data from DB──►  data/*.parquet  ──►  analysis.py  ──►  app.py  ──►  dashboard on screen
```

1. **`db.py`** — talks to the database, downloads summarised data, saves it to files
2. **`analysis.py`** — reads those files and does the maths (revenues, costs, margins)
3. **`app.py`** — shows everything on screen as an interactive dashboard

You run `db.py` **once** to get the data. Then you run `app.py` every time you want to see the dashboard. `analysis.py` runs automatically in the background when the dashboard loads.

---

## db.py — The Database Connector

**What it does:** Connects to the airline's DB2 database, runs three big SQL queries, and saves the results as `.parquet` files (a compact data format, like a very efficient Excel file).

**Why not just load the raw data?** The TICKETS table has **35 million rows**. Loading that into Python would crash or take forever. Instead, we tell the database to summarise the data first (e.g. "give me total revenue per route per year") and only send back the summary — a few thousand rows instead of 35 million.

### Important functions

---

#### `make_engine()`
Creates the connection to the database.
Think of it as dialling a phone number — this function dials the DB2 server using the credentials stored in `.env`.

```python
engine = make_engine()
```

---

#### `q_ident(name)`
Wraps a table or column name in double quotes so DB2 reads it correctly.

```python
q_ident("TICKETS")  # returns  "TICKETS"
```

DB2 can get confused by names that clash with reserved words or have mixed capitalisation. This function prevents that.

---

#### `qualified_table(schema, table)`
Combines schema + table into the full name DB2 needs.

```python
qualified_table("ATTGRP3", "TICKETS")  # returns  "ATTGRP3"."TICKETS"
```

---

#### `normalize_column_names(df)`
After reading from DB2, all column names come back in UPPER_CASE. This function converts them to lowercase so the rest of the code is easier to read.

```python
# Before:  ROUTE_CODE, NET_REVENUE
# After:   route_code, net_revenue
```

---

#### `extract_revenue(engine)`
Runs the biggest query: total revenue, taxes, and ticket count **grouped by route + year + cabin class**.

This is why the result is small: instead of 35M ticket rows, we get roughly **4,000 summary rows** (59 routes × 25 years × 3 classes).

Key columns in the output:
- `route_code`, `origin`, `destination` — which route
- `yr` — the year
- `class` — B (Business), P (Premium), E (Economy)
- `net_revenue` — total ticket prices (taxes excluded)
- `total_taxes` — airport + local taxes collected
- `ticket_count` — how many tickets sold

---

#### `extract_fuel(engine)`
Calculates fuel consumption per route, grouped by aircraft model.

Formula: `flights operated × fuel burn rate (gallons/hour) × flight duration (hours)`

This is done in Python (not SQL) because multiplying large integers inside DB2 would overflow. The result is one row per route + aircraft model combination.

---

#### `extract_airports(engine)`
Loads the full airports table — only 30 rows, safe to load completely.
Used to look up city names and continents for the charts.

---

#### `main()`
Runs all three extractions in order and saves the results:
- `data/revenue.parquet`
- `data/fuel.parquet`
- `data/airports.parquet`

Run with: `python db.py`

---

## analysis.py — The Maths Layer

**What it does:** Reads the three parquet files and calculates all the metrics the dashboard needs. No database calls here — pure data processing using **Polars** (a fast data manipulation library, similar to pandas).

Everything uses the `.lazy() → .collect()` pattern, which means Polars figures out the most efficient way to run a calculation before actually running it.

### Important functions

---

#### `enrich_revenue_with_airports(revenue, airports)`
Adds continent and city name to each revenue row.

The revenue table only has `origin` and `destination` as airport codes (e.g. `MAD`, `JFK`). This function looks those codes up in the airports table and adds columns like `origin_continent` and `origin_city`.

It joins the airports table **twice** — once for origin, once for destination — because each route has two endpoints.

---

#### `apply_filters(revenue, year_min, year_max, cabin_classes, origin_continents, min_tickets)`
Applies the sidebar filters the user selects on the dashboard.

Takes the full revenue DataFrame and returns a smaller one with only the rows matching:
- The selected year range
- The selected cabin class (Business / Premium / Economy / All)
- The selected origin continent
- Routes with at least `min_tickets` tickets sold (removes noisy tiny routes)

Every chart uses this filtered DataFrame, so all charts always show the same subset of data.

---

#### `compute_kpis(revenue_filtered, fuel)`
Calculates the 7 numbers shown in the top cards on the dashboard:

| KPI | How it's calculated |
|-----|---------------------|
| Total Net Revenue | Sum of all `price` values (taxes excluded) |
| Est. Fuel Cost | Total fuel gallons × $3/gallon |
| Est. Margin % | (Net Revenue − Fuel Cost) / Net Revenue |
| Avg Tax Burden % | Total taxes / Total gross revenue |
| Total Tickets | Sum of ticket counts |
| Best Route | Route with highest (revenue − fuel cost) |
| Worst Route | Route with lowest (revenue − fuel cost) |

---

#### `route_profitability(revenue_filtered, fuel)`
Prepares data for the **scatter chart** (Section 1).

Returns one row per route with:
- `fuel_cost` → x-axis
- `net_revenue` → y-axis
- `ticket_count` → bubble size
- `origin_continent` → bubble colour
- `est_profit` → shown on hover

---

#### `tax_drain_by_route(revenue_filtered)`
Prepares data for the **tax bar charts** (Section 2).

Groups revenue by route and calculates `tax_pct` = taxes as a percentage of gross revenue. Sorted highest-to-lowest so the most tax-burdened routes appear first.

---

#### `fleet_efficiency(revenue_filtered, fuel)`
Prepares data for the **fleet bar chart** (Section 3).

Groups fuel consumption by aircraft model, joins revenue onto it, then calculates `rev_per_gallon` = net revenue ÷ total gallons burned. A higher number means the aircraft earns more for each gallon it burns.

---

#### `margin_trend(revenue, fuel)`
Prepares data for the **margin trend line chart** (Section 4).

Groups revenue by year (2000–2024). Since the fuel data has no year dimension, it allocates total fleet fuel cost across years proportionally — if year 2010 generated 4% of all-time revenue, it gets assigned 4% of total fuel cost. This is an approximation.

Returns `margin_pct` per year = (revenue − allocated fuel cost) / revenue.

---

## app.py — The Dashboard

**What it does:** Builds the visual dashboard using **Streamlit**. Reads the parquet files, calls `analysis.py` functions to get the data, and draws interactive Plotly charts.

### How it's structured

```
1. Load data (revenue, fuel, airports) from parquet — cached so it's fast
2. Enrich revenue with airport info
3. Draw sidebar filters (year slider, class, continent, min tickets)
4. Apply filters → revenue_filtered
5. Compute KPIs → show 6 metric cards
6. Section 1: Route Profitability scatter
7. Section 2: Tax Drain bar charts (side by side)
8. Section 3: Fleet Efficiency bar chart
9. Section 4: Margin Trend line charts
10. Expandable data table + CSV download button
```

### Key Streamlit concepts used

**`@st.cache_data`** — placed above each data-loading function. Means "load this file once and remember the result." Without it, Streamlit would re-read the parquet files every time the user moves a slider.

**`st.sidebar`** — everything inside this goes into the left panel. Filters live here so they don't clutter the main view.

**`st.plotly_chart(fig, use_container_width=True)`** — renders a Plotly chart and stretches it to fill the full column width.

**`st.columns(n)`** — splits the page into n side-by-side columns. Used for the KPI cards (6 columns) and the two tax charts (2 columns).

---

## Assumptions Baked Into the Code

| Assumption | Where in code | Why |
|---|---|---|
| Fuel price = $3/gallon | `analysis.py` line 14: `FUEL_PRICE_USD = 3.0` | Fixed estimate; real Jet-A varies |
| Taxes are pass-through | `extract_revenue` — taxes tracked separately, never added to margin | Airline doesn't keep tax money |
| Yearly fuel allocation is proportional | `margin_trend()` | Fuel extract has no year column; only approximation available |
| `TICKETS.PRICE` = net revenue | Used directly in all revenue sums | DB column excludes taxes by definition |

---

## How the Three Files Connect

```
db.py                    analysis.py              app.py
─────────────────────    ─────────────────────    ─────────────────────
make_engine()        →   (reads parquet files)    load_revenue()
extract_revenue()    →   enrich_revenue...()  →   revenue_enriched
extract_fuel()       →   apply_filters()      →   revenue_filtered
extract_airports()   →   compute_kpis()       →   KPI cards
main() saves             route_profitability() →   Section 1 chart
  revenue.parquet        tax_drain_by_route() →   Section 2 charts
  fuel.parquet           fleet_efficiency()   →   Section 3 chart
  airports.parquet       margin_trend()       →   Section 4 charts
```
