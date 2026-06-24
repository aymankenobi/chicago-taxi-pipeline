# Chicago Taxi Analytics Pipeline

A production-ready data analytics pipeline built on Google Cloud Platform, transforming the Chicago Taxi Trips public dataset into actionable business insights using **BigQuery**, **Dataform**, and **Looker Studio**.

## Dashboard

**[View Live Dashboard](https://datastudio.google.com/reporting/ff77bcef-5eff-4b41-b958-144317363858)**

## Architecture

```
Source (BigQuery Public Data)
│   bigquery-public-data.chicago_taxi_trips.taxi_trips
│
├── Staging Layer
│   └── stg_taxi_trips              — Cleaned & standardised (196M → 132M rows)
│
├── Reference Layer
│   └── ref_us_holidays             — US federal holidays 2013–2025
│
├── Intermediate Layer
│   └── int_taxi_shifts             — Shift detection via gap analysis + 24hr cap
│
├── Marts Layer
│   ├── mart_taxi_tips              — Q1: Tip performance by taxi/month
│   ├── mart_taxi_overworkers       — Q2: Shift summaries per taxi
│   ├── mart_shift_distribution     — Q2: Shift duration histogram
│   ├── mart_overwork_patterns      — Q2: When overwork happens (hour/day/year)
│   ├── mart_fatigue_analysis       — Q2: First half vs second half of long shifts
│   ├── mart_daily_trips            — Q3: Daily trips with holiday flag
│   ├── mart_bonus_insights         — Q4: Revenue leakage & peak demand
│   └── mart_company_performance    — Q4: Company benchmarks & digital adoption
│
└── Visualisation
    └── Looker Studio Dashboard (4+ pages)
```

## Tech Stack

| Component | Tool |
|---|---|
| Data Warehouse | Google BigQuery |
| Transformation & Orchestration | GCP Dataform (SQLX) |
| Visualisation | Looker Studio |
| Version Control | GitHub |

## Design Principles

- **Separation of transformation and presentation logic**: Marts contain business-ready aggregations without hardcoded filters (no `LIMIT 100`, no date restrictions). Filtering, sorting, and ranking are handled in Looker Studio.
- **Layered architecture**: Source → Staging → Reference → Intermediate → Marts. Each layer has a single responsibility.
- **Data quality assertions**: Built into Dataform config blocks — non-null checks and uniqueness constraints run on every pipeline execution.
- **Schema documentation**: Every column in every model includes a description, pushed to BigQuery metadata.
- **Reference data separation**: US holidays maintained as a standalone seed table, not hardcoded in query logic.

## Data Cleaning (Staging)

The staging layer removes ~64M invalid records from the raw dataset:

| Rule | Rationale |
|---|---|
| `taxi_id IS NOT NULL` | Cannot attribute trips without an identifier |
| `trip_start_timestamp IS NOT NULL` | Required for time-based analysis |
| `trip_end_timestamp IS NOT NULL` | Required for shift detection |
| `trip_seconds > 0` | Zero-duration trips are cancelled or system errors |
| `trip_seconds < 86400` | Trips longer than 24 hours are data errors |
| `trip_miles >= 0` | Negative distances are invalid |
| `fare >= 0` | Negative fares are invalid |
| `tips >= 0` | Negative tips are invalid |
| `trip_total > 0` | Zero-total trips provide no analytical value |
| `trip_start < trip_end` | End time must follow start time |

**Data quality finding**: The raw dataset contains duplicate `unique_key` values. Identified during assertion testing, documented, and the uniqueness assertion was deliberately removed since the duplication originates in the source data.

---

## Analytical Questions

### Q1: Top 100 Tip Earners (Last 3 Months)

**Definition**: Taxi IDs ranked by total tips earned, filtered to the last 3 months of available data.

**Key Assumptions**:
- **Cash tips are not recorded.** Only credit card payments capture tip amounts (94.93% capture rate vs 0.14% for cash). Analysis filtered to `payment_type = 'Credit Card'` to avoid penalising taxis in cash-heavy areas for a data limitation.
- **"Last 3 months"** = October–December 2023. The dataset's most recent date is 2023-12-31.
- **No hardcoded LIMIT or date filter in the mart.** The mart aggregates by taxi × month. Looker Studio applies year/month filters and sorts by total_tips descending.
- **Calculated fields in Looker Studio**: `SUM(total_tips) / SUM(total_trips)` for average tip and `SUM(total_tips) / SUM(total_fare)` for tip percentage. This avoids the statistical error of averaging pre-aggregated monthly averages.
- **Ranking by total, not rate.** A driver who earned $5,000 in 2 months outranks a driver who earned $3,000 in 3 months. Not every taxi operates every month — this is expected and does not bias the ranking.

---

### Q2: Top 100 Overworkers

**Definition**: Taxi IDs that work the most hours without taking at least 8-hour breaks, with regularly long shifts.

**Shift Detection — Two Rules**:

**Rule 1 — Gap-based**: A new shift starts when the gap between consecutive trips exceeds 480 minutes (8 hours). This threshold comes from the brief ("without taking at least 8 hours break") and aligns with US DOT commercial driver rest requirements.

**Rule 2 — 24-hour cap (v2 fix)**: Even without an 8-hour gap, shifts are force-split every 24 hours using `FLOOR(minutes_since_shift_start / 1440)`. This prevents mega-shifts from drivers who take consistent sub-8-hour breaks.

**Why the 24-hour cap was needed**:
- In v1, 6.16% of all shifts (739,979) exceeded 24 hours. The worst was 3,411 hours (142 days) with only 23.4% utilisation.
- The edge case: a driver taking 6-hour breaks would never trigger the 8-hour gap rule, merging multiple days into one shift.
- After the fix: zero shifts exceed 48 hours. Max is 47.75 hours (a trip spanning the 24-hour boundary). The 0.67% between 24-48 hours are boundary artifacts from trips starting near the 24-hour mark.

**Why max is ~48 hours, not 24**: FLOOR splits on trip start times, not end times. A trip starting at hour 23.75 stays in the current sub-shift, but the trip itself can last up to 24 hours (staging max), pushing the shift end to hour 47.75. Theoretical ceiling: 23.99 + 24 = 47.99 hours. Data confirms: max = 47.75, zero above 48.

**Why 10 hours as the overwork threshold**:
1. US DOT limits commercial drivers to 11 hours of driving within a 14-hour window. 10 hours is just under this regulatory ceiling.
2. Chicago's taxi industry runs two shifts per day, each 10-12 hours. Exceeding 10 means pushing past the standard window.
3. Fatigue research shows cognitive and motor performance degrades after 10 hours of sustained work.

Note: The 10-hour threshold is pre-calculated as a `COUNTIF` in the mart. The underlying intermediate table contains all shifts at all durations with no filter. In production, this threshold would be parameterised using a Dataform variable.

**Additional Q2 Analyses**:
- **Shift distribution histogram** (`mart_shift_distribution`): Validates the fix visually — most shifts cluster under 12 hours, 24+ bucket is minimal.
- **Overwork patterns** (`mart_overwork_patterns`): Long shifts by start hour, day of week, and year. Finding: overwork peaked 2014-2016 (~1M long shifts/year), declined sharply during COVID, recovering but below pre-COVID levels.
- **Fatigue analysis** (`mart_fatigue_analysis`): Compares trip metrics in first half vs second half of long shifts. Finding: fares drop ~5.6% in the second half of 16-20 hour shifts. Drivers take shorter, cheaper trips as fatigue sets in.

---

### Q3: Holiday Impact on Trips

**Definition**: Do US public holidays increase or decrease taxi trip volumes?

**Methodology**:
- 8 US federal holidays tracked per year from 2013 to 2025 in a standalone reference table (`ref_us_holidays`).
- Daily trip data joined to holiday table via `trip_date = holiday_date`.
- Comparison uses **average daily trips** (not totals) to account for the imbalance between ~350 non-holiday days and ~8 holiday days per year.

**Finding**: Holidays show approximately 40% fewer trips on average compared to non-holidays. Christmas and Thanksgiving show the steepest declines. Labour Day and Memorial Day show minimal impact due to long weekend travel.

**Known limitations**: This comparison conflates COVID effects, seasonality, day-of-week patterns, and year-over-year decline from rideshare competition. A more rigorous approach would compare each holiday to the same day-of-week in the same month of the same year (controlled comparison). The mart data supports this analysis — `trip_date`, `trip_day_of_week`, `trip_month`, and `trip_year` are all available for a controlled self-join.

---

### Q4: Bonus Insights

#### Insight 1: Tip Data Visibility Gap (Revenue Leakage)

**Finding**: Cash payments (53% of all trips) record near-zero tips. Credit card tips total $237.9M while cash shows $413K. An estimated $298.5M in tip revenue is invisible to the system.

**How the estimate is calculated**: Credit card average tip per trip ($4.23) × number of cash trips (70.5M) = $298.5M. The assumption — cash passengers tip at similar rates — is supported by the data: every digital payment method that records tips (Credit Card, Mobile, Way2ride, Split) shows similar tip rates. The only outlier is cash, and the only difference is the recording mechanism.

**Recommendation**: Incentivise digital payment adoption for revenue visibility, driver compensation transparency, and service quality tracking. Target specific companies and time slots with the highest cash usage rates.

#### Insight 2: Cash Decline Has Plateaued

**Finding**: Cash usage dropped from ~14M trips/year (2014) to ~5M (2019), driven by rideshare competition. Since 2020, cash has stabilised. The organic decline is over — remaining cash users are resistant to change.

**Recommendation**: The blanket market shift toward digital is complete. Remaining cash users require targeted intervention — identify which companies and time windows still have the highest cash rates and focus digital payment programs there.

**Dashboard storytelling**: Four scorecards (total trips → cash share → recorded tips → estimated invisible) followed by pie chart (scale), bar chart (blind spot), and stacked bar (trend). Each visual makes one point; the story is understood without narration.

---

## Project Structure

```
chicago-taxi-pipeline/
├── definitions/
│   ├── sources/
│   │   └── src_taxi_trips.sqlx
│   ├── staging/
│   │   └── stg_taxi_trips.sqlx
│   ├── reference/
│   │   └── ref_us_holidays.sqlx
│   ├── intermediate/
│   │   └── int_taxi_shifts.sqlx
│   └── marts/
│       ├── mart_taxi_tips.sqlx
│       ├── mart_taxi_overworkers.sqlx
│       ├── mart_shift_distribution.sqlx
│       ├── mart_overwork_patterns.sqlx
│       ├── mart_fatigue_analysis.sqlx
│       ├── mart_daily_trips.sqlx
│       ├── mart_bonus_insights.sqlx
│       └── mart_company_performance.sqlx
├── includes/
├── workflow_settings.yaml
├── .gitignore
└── README.md
```

## How to Run

1. Clone this repo
2. Set up a GCP project with BigQuery and Dataform APIs enabled
3. Create a Dataform repository and connect to this repo
4. Create BigQuery datasets: `staging`, `intermediate`, `reference`, `marts`, `dataform_assertions` (location: `US`)
5. Grant the Dataform service account: `BigQuery Data Editor`, `BigQuery Data Viewer`, `BigQuery Job User`
6. Create a development workspace and execute all actions
7. Connect Looker Studio to the mart tables

## Version History

| Version | Changes |
|---|---|
| v1 | Initial pipeline — hardcoded LIMIT 100 and date filters in marts |
| v2 | Refactored — separated transformation from presentation logic, added assertions and schema docs |
| v3 | Fixed shift detection edge case (24hr cap), added fatigue analysis, improved revenue leakage visuals |

## Author

Muhammad Ayman Abd Rahman
