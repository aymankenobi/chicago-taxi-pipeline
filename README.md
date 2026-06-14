# Chicago Taxi Analytics Pipeline

A production-ready data analytics pipeline built on Google Cloud Platform, transforming the Chicago Taxi Trips public dataset into actionable business insights using **BigQuery**, **Dataform**, and **Looker Studio**.

## 📊 Dashboard

**[View Live Dashboard](https://datastudio.google.com/reporting/ff77bcef-5eff-4b41-b958-144317363858)**

## Architecture

```
Source (BigQuery Public Data)
│   bigquery-public-data.chicago_taxi_trips.taxi_trips
│
├── Staging Layer
│   └── stg_taxi_trips          — Cleaned & standardised trips (132M+ rows)
│
├── Reference Layer
│   └── ref_us_holidays         — US federal holidays 2013–2025
│
├── Intermediate Layer
│   └── int_taxi_shifts         — Shift detection via gap analysis
│
├── Marts Layer
│   ├── mart_taxi_tips          — Q1: Tip performance by taxi/month
│   ├── mart_taxi_overworkers   — Q2: Overwork shift summaries
│   ├── mart_daily_trips        — Q3: Daily trips with holiday flag
│   └── mart_bonus_insights     — Q4: Revenue leakage & peak demand
│
└── Visualisation
    └── Looker Studio Dashboard (4 pages)
```

## Tech Stack

| Component | Tool |
|---|---|
| Data Warehouse | Google BigQuery |
| Transformation & Orchestration | GCP Dataform (SQLX) |
| Visualisation | Looker Studio |
| Version Control | GitHub |

## Design Principles

- **Separation of transformation and presentation logic**: Marts contain business-ready aggregations without hardcoded filters (no `LIMIT 100`, no date restrictions). Filtering, sorting, and ranking are handled in Looker Studio, giving end users full flexibility.
- **Layered architecture**: Source → Staging → Intermediate → Marts. Each layer has a single responsibility.
- **Data quality assertions**: Built into Dataform config blocks — non-null checks and uniqueness constraints validate data on every pipeline run.
- **Schema documentation**: Every column in every model includes a description, pushed to BigQuery metadata for discoverability.
- **Reference data separation**: US holidays are maintained as a standalone seed table, not hardcoded in query logic.

## Data Cleaning (Staging)

The staging layer applies the following filters to the raw dataset (~196M rows → ~132M clean rows):

| Rule | Rationale |
|---|---|
| `taxi_id IS NOT NULL` | Cannot attribute trips without a taxi identifier |
| `trip_start_timestamp IS NOT NULL` | Required for time-based analysis |
| `trip_end_timestamp IS NOT NULL` | Required for shift detection |
| `trip_seconds > 0` | Zero-duration trips are invalid (cancelled or system errors) |
| `trip_seconds < 86400` | Trips longer than 24 hours are data errors |
| `trip_miles >= 0` | Negative distances are invalid |
| `fare >= 0` | Negative fares are invalid |
| `tips >= 0` | Negative tips are invalid |
| `trip_total > 0` | Zero-total trips provide no analytical value |
| `trip_start < trip_end` | End time must be after start time |

**Data quality finding**: The raw dataset contains duplicate `unique_key` values. This was identified during assertion testing and documented. The uniqueness assertion was removed from the staging model since the duplication originates in the source data.

---

## Analytical Questions

### Q1: Top 100 Tip Earners (Last 3 Months)

**Definition**: Taxi IDs ranked by total tips earned, filtered to the last 3 months of available data.

**Assumptions & Methodology**:
- **Cash tips are not recorded** in this dataset. Only credit card payments capture tip amounts. Therefore, the analysis is filtered to `payment_type = 'Credit Card'` only. This is a critical assumption — the "top tip earners" ranking reflects credit card tipping behaviour, not total tipping across all payment methods.
- **"Last 3 months"** is defined as the 3 calendar months preceding the most recent trip date in the dataset. The dataset's latest date is **2023-12-31**, so the analysis window is **October–December 2023**.
- The mart (`mart_taxi_tips`) is aggregated at the **taxi × month** grain with no hardcoded date filters or row limits. The Looker Studio dashboard applies year/month dropdown filters and sorts by `total_tips DESC` to show the top earners.
- **Avg Tip Per Trip** and **Tip Percentage** are calculated as Looker Studio calculated fields (`SUM(total_tips) / SUM(total_trips)` and `SUM(total_tips) / SUM(total_fare)`) to avoid the statistical error of averaging pre-aggregated averages across months.
- Not every taxi operates in every month. A driver who worked only October and November is still ranked fairly by their actual total — they are not penalised for missing December. The question asks "who earns more" (a total), not "who earns most efficiently" (a rate).

**Dashboard**: Page 1 — filterable table sorted by total tips.

---

### Q2: Top 100 Overworkers

**Definition**: Taxi IDs that work the most hours without taking at least 8-hour breaks, with regularly long shifts.

**Assumptions & Methodology**:
- **Shift detection** uses gap analysis: a new shift starts when there is a gap of **480+ minutes (8 hours)** between the end of one trip and the start of the next for the same taxi. This is implemented as a window function in `int_taxi_shifts.sqlx`.
- A **long shift** is defined as any shift with total elapsed duration exceeding **10 hours**. This threshold reflects realistic taxi driver behaviour — a standard shift is 8–12 hours, so 10+ hours indicates extended work.
- The intermediate layer (`int_taxi_shifts`) detects every shift for every taxi. The mart (`mart_taxi_overworkers`) aggregates per taxi: total shifts, long shift count, average/max duration, active driving hours, and long shift ratio.
- **No hardcoded LIMIT or HAVING** in the mart. The dashboard sorts by `total_long_shifts DESC` and displays the top entries.
- **Long shift ratio** (`total_long_shifts / total_shifts × 100`) measures how regularly a driver works extended hours — a high ratio indicates a systemic pattern, not a one-off.

**Dashboard**: Page 2 — table sorted by total long shifts.

---

### Q3: Holiday Impact on Trips

**Definition**: Do US public holidays increase or decrease taxi trip volumes?

**Assumptions & Methodology**:
- **8 US federal holidays** are tracked per year (New Years Day, MLK Day, Presidents Day, Memorial Day, Independence Day, Labor Day, Thanksgiving, Christmas Day) from 2013 to 2025.
- Holidays are maintained in a **standalone reference table** (`ref_us_holidays`), joined to daily trip data via `trip_date = holiday_date`. This is more maintainable than hardcoding dates in query logic.
- The comparison uses **average daily trips** (not totals) to account for the imbalance between ~350 non-holiday days and ~8 holiday days per year.
- The dataset covers 2013–2023, yielding **88 holiday observations** across 11 years.

**Finding**: Holidays show a **~40% decrease** in average daily trips compared to non-holidays (20K vs 33.3K average trips per day). This aligns with expectations — fewer commuters, reduced business travel, and office closures on federal holidays reduce taxi demand. However, individual holidays vary: Labour Day and Memorial Day show higher volumes (long weekend travel), while Christmas and Thanksgiving show the steepest declines.

**Dashboard**: Page 3 — bar chart comparing Holiday vs Non-Holiday average daily trips, plus a table of all holiday dates with trip counts and revenue.

---

### Q4: Bonus Insights

#### Insight 1: Revenue Leakage from Cash Payments

**Business Value**: Cash payments do not record tips, creating a blind spot in revenue tracking. Credit card trips average ~20% tip rates. With millions of cash trips annually, the untracked tip revenue represents a significant data gap. This insight supports a business case for **promoting digital payment adoption** — not just for convenience, but for revenue visibility and driver compensation transparency.

**Data Points**:
- Credit Card tips total: **$237.9M** (recorded)
- Cash tips total: **$413K** (almost certainly under-reported, likely near zero in reality)
- Mobile tips total: **$6.9M**
- The gap between Credit Card and Cash tip totals demonstrates the scale of invisible revenue.

#### Insight 2: Peak Demand–Supply Gap

**Business Value**: Trip volume peaks between **5–7 PM** (8.9–9.4M trips), coinciding with evening commute hours. The early morning trough (4–6 AM, ~1.4M trips) represents the lowest demand. This demand curve enables **fleet optimisation**: deploying more cabs during 4–8 PM peak and reducing idle fleet during 3–6 AM. Matching supply to demand improves driver earnings, reduces passenger wait times, and increases fleet utilisation.

**Data Points**:
- Peak hour: **6 PM** (9.4M trips)
- Lowest hour: **5 AM** (1.4M trips)
- Peak-to-trough ratio: **6.7x** — demand varies nearly 7-fold across the day

**Dashboard**: Page 4 — bar chart showing tip totals by payment type (revenue leakage) and bar chart showing trip volume by hour (peak demand).

---

## Project Structure

```
chicago-taxi-pipeline/
├── definitions/
│   ├── sources/
│   │   └── src_taxi_trips.sqlx         # Source declaration
│   ├── staging/
│   │   └── stg_taxi_trips.sqlx         # Data cleaning & standardisation
│   ├── reference/
│   │   └── ref_us_holidays.sqlx        # US holiday calendar
│   ├── intermediate/
│   │   └── int_taxi_shifts.sqlx        # Shift detection logic
│   └── marts/
│       ├── mart_taxi_tips.sqlx         # Q1: Tip performance
│       ├── mart_taxi_overworkers.sqlx  # Q2: Overwork analysis
│       ├── mart_daily_trips.sqlx       # Q3: Holiday impact
│       └── mart_bonus_insights.sqlx    # Q4: Bonus insights
├── includes/
├── workflow_settings.yaml
├── .gitignore
└── README.md
```

## How to Run

1. Clone this repo
2. Set up a GCP project with BigQuery and Dataform APIs enabled
3. Create a Dataform repository and link to this repo
4. Create BigQuery datasets: `staging`, `intermediate`, `reference`, `marts`, `dataform_assertions` (location: `US`)
5. Grant the Dataform service account roles: `BigQuery Data Editor`, `BigQuery Data Viewer`, `BigQuery Job User`
6. Create a development workspace and execute all actions
7. Connect Looker Studio to the mart tables in BigQuery

## Author

Muhammad Ayman Abd Rahman
