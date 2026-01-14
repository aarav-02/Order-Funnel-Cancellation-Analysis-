# Order Funnel Cancellation Analysis

A funnel analysis project that uses SQL (BigQuery) to prepare event-level data and Excel to visualize and analyze user progression through a purchase funnel. The goal is to create a clear funnel chart for the top 3 countries (by total event volume) using six event types and provide actionable insights.

---

## Table of contents
- [Project overview](#project-overview)
- [Dataset](#dataset)
- [Funnel definition (events)](#funnel-definition-events)
- [Methodology (SQL + Excel)](#methodology-sql--excel)
  - [SQL steps (summary)](#sql-steps-summary)
  - [Example SQL pattern](#example-sql-pattern)
  - [Excel steps (visualization & analysis)](#excel-steps-visualization--analysis)
- [Results & analysis](#results--analysis)
- [Repository structure](#repository-structure)
- [How to reproduce / Runbook](#how-to-reproduce--runbook)
- [Findings & recommendations](#findings--recommendations)
- [Next steps](#next-steps)
- [Author / Contact](#author--contact)
- [License](#license)

---

## Project overview
Funnel analysis is used to measure how users move through a defined linear journey (e.g., discovery → product view → cart → checkout → purchase). This project:
- Prepares event data (one unique event per user per funnel step).
- Filters to the top 3 countries by event count.
- Computes per-step counts and conversion/drop-off rates.
- Produces funnel charts and further analysis in Excel.

---

## Dataset
Source: `raw_events` table hosted in a BigQuery project (project and dataset used in your environment).

The `raw_events` table contains event-level records with at least:
- `user_pseudo_id` (identifier for a user/session)
- `event_name` (e.g., `page_view`, `view_item`, ...)
- `event_timestamp` (when the event occurred)
- geo information such as `country` or `user_country` (column names may vary)
- other event params (item ids, revenue, etc.)

Tip: When referencing fields, adapt names to match your table schema.

---

## Funnel definition (events)
This analysis uses the following 6 ordered events (adjust names to match the dataset):
1. `page_view`
2. `view_item`
3. `add_to_cart`
4. `add_shipping_info`
5. `add_payment_info`
6. `purchase`

These represent the common purchase funnel steps (discovery → product view → cart → shipping → payment → purchase).

---

## Methodology (SQL + Excel)

### SQL steps (summary)
1. Deduplicate events by user and event type:
   - Keep the first occurrence of each event per `user_pseudo_id` to avoid multiple visits overcounting the funnel.
   - Use `ROW_NUMBER()` partitioned by `user_pseudo_id, event_name` ordered by `event_timestamp`.
2. Filter data to only the six funnel events.
3. Determine the top 3 countries by total number of events (or by unique users) and restrict the analysis to those countries.
4. Aggregate counts per event step per country.
5. Rank the events to preserve funnel order and produce an output table suitable for charting:
   - Columns: `country`, `event_name`, `event_order`, `users_count`.
6. Export the aggregated results (CSV) for visualization in Excel.

### Example SQL pattern
Replace column/table names and dataset/project identifiers as appropriate for your BigQuery environment:

```sql
-- Example: aggregate first occurrence per user and event, then funnel counts for top 3 countries
WITH first_events AS (
  SELECT
    user_pseudo_id,
    event_name,
    event_timestamp,
    country,
    ROW_NUMBER() OVER (PARTITION BY user_pseudo_id, event_name ORDER BY event_timestamp) AS rn
  FROM `PROJECT.DATASET.raw_events`
  WHERE event_name IN ('page_view','view_item','add_to_cart','add_shipping_info','add_payment_info','purchase')
),
unique_events AS (
  SELECT user_pseudo_id, event_name, event_timestamp, country
  FROM first_events
  WHERE rn = 1
),
country_totals AS (
  SELECT country, COUNT(*) AS total_events
  FROM unique_events
  GROUP BY country
),
top_countries AS (
  SELECT country
  FROM country_totals
  ORDER BY total_events DESC
  LIMIT 3
),
funnel_counts AS (
  SELECT
    ue.country,
    ue.event_name,
    COUNT(DISTINCT ue.user_pseudo_id) AS users_count
  FROM unique_events ue
  JOIN top_countries tc USING(country)
  GROUP BY ue.country, ue.event_name
)
SELECT
  country,
  event_name,
  users_count,
  CASE event_name
    WHEN 'page_view' THEN 1
    WHEN 'view_item' THEN 2
    WHEN 'add_to_cart' THEN 3
    WHEN 'add_shipping_info' THEN 4
    WHEN 'add_payment_info' THEN 5
    WHEN 'purchase' THEN 6
    ELSE 999 END AS event_order
FROM funnel_counts
ORDER BY country, event_order;
```

Notes:
- Use `COUNT(DISTINCT user_pseudo_id)` if you want unique users per step.
- If `country` is null for some events, consider filtering or marking them as "Unknown".

### Excel steps (visualization & analysis)
After exporting the aggregated SQL results (CSV):
1. Import the CSV into Excel.
2. Create a PivotTable:
   - Rows: `event_order` (or `event_name`)
   - Columns: `country`
   - Values: `users_count` (Sum)
3. Create funnel visualizations:
   - Use Excel’s built-in Funnel chart (if available) or a stacked bar chart with values displayed as percentages.
   - Alternatively, compute % of previous step and overall conversion and visualize as shape/area charts.
4. Add columns for:
   - Step-to-step conversion rate (users at step N / users at step N-1)
   - Overall conversion rate (users at final step / users at first step)
   - Drop-off rate per step
5. Annotate the charts with key insights: where the largest drop-offs occur, country comparisons, and potential causes.

---

## Results & analysis
The detailed analysis, charts, and actionable insights are available in the included Excel workbook (see repository file: `funnel_analysis_results.xlsx` or similar). The workbook contains:
- Raw aggregated table for the top 3 countries
- Pivot tables and charts
- A short analysis sheet describing key takeaways and recommended experiments to improve conversion

(If the workbook is not yet in the repo, run the SQL, export results, and produce the workbook following the steps above.)

---

## Repository structure
- README.md — this file
- sql/
  - funnel_query.sql — (recommended) BigQuery SQL used to aggregate funnel counts
- reports/
  - funnel_analysis_results.xlsx — Excel workbook with charts and analysis
- data_exports/
  - funnel_top3_countries.csv — aggregated CSV produced by SQL for charting
- notebooks/ (optional)
  - analysis.ipynb — if you prefer to use Python for visualization

Adjust filenames and folders as you add files to the repo.

---

## How to reproduce / Runbook
Prerequisites:
- BigQuery access to the project containing `raw_events`.
- Excel (or Google Sheets) for visualization.

Steps:
1. Update `sql/funnel_query.sql` with your project and dataset identifiers.
2. Run the query in BigQuery and save the output as CSV.
3. Open `funnel_analysis_results.xlsx` or create a new Excel workbook and follow the Excel steps to create funnels and compute conversion/drop-off rates.
4. Commit the SQL and resulting workbook to the repository.

Optional automation:
- Use a scheduled BigQuery job to refresh the aggregated CSV daily.
- Automate chart updates in Google Sheets using the CSV as a data source.

---

## Findings & recommendations (example template)
- Identify the funnel step with the highest drop-off (e.g., `add_shipping_info` → `add_payment_info`).
- Compare conversion rates across the top 3 countries:
  - Country A: Stronger product discovery but lower checkout conversions.
  - Country B: High cart add-to-purchase rate — consider replicating UX elements.
- Recommendations:
  - A/B test checkout flow simplifications.
  - Investigate payment method availability per country.
  - Add reminders or email flows for users who reach add_to_cart but do not complete purchase.

Fill these with specific numbers from your Excel analysis.

---

## Next steps
- Expand analysis to more countries or segment by device / channel.
- Analyze time-to-convert (time between funnel steps).
- Correlate funnel performance with marketing campaign or product changes.
- Add retention/cohort analysis to complement funnel view.

---

## Author / Contact
- Author: aarav-02
