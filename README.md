# Data Cleaning & Modeling (Power Query — NYC 311)

This project uses a reproducible Power Query (M) pipeline to prepare NYC 311 service requests for analysis in Power BI. The steps focus on **data quality, portability, and performance** (smaller, faster refreshes).

---

## Pipeline Overview

flowchart TD
  A[CSV Import] --> B[Promote Headers]
  B --> C[Type Conversion]
  C --> D[Text Hygiene & Casing]
  D --> E[ZIP Normalize (5-digit text)]
  E --> F[Geo Validate (NYC bounds)]
  F --> G[HoursToClose (Closed - Created)]
  G --> H[Clamp Durations (-0.1..720h)]
  H --> I[Rolling Window (last 6 months)]
  I --> J[Drop Redundant/Empty Columns]
  J --> K[Hour Bucket + Geo Rounding]
  K --> L[Sort by Created (asc)]
  L --> M[Distinct per Composite Key]
  M --> N[Reorder Columns → FactRequests]
```


## Step-by-step

### 1) Import & Types
- Load the CSV with `Csv.Document` → `Table.PromoteHeaders`.
- Explicitly set column data types (datetimes for `Created Date` / `Closed Date`, numeric for `Latitude` / `Longitude`, text for categoricals).  
**Why:** prevents type drift and makes later calculations consistent.

### 2) Text Hygiene & Standardization
- Trim and clean text, then apply consistent casing on key dimensions:  
  - `Agency` and `Borough` → **UPPERCASE**  
  - `Complaint Type`, `Descriptor`, `City`, `Status`, `Location Type`, `Address Type` → **Proper Case**  
**Why:** reduces duplicate labels from typos/case differences; improves slicer usability.

### 3) ZIP Code Normalization
- Store `Incident Zip` as **5‑digit text** (strip non‑digits; anything not 5 digits → null).  
**Why:** ZIPs are identifiers; leading zeros matter; arithmetic is meaningless.

### 4) Geospatial Validation
- Validate NYC lat/long range: **Latitude 40.40–40.95**, **Longitude −74.30 to −73.65**; out‑of‑bounds → null.  
**Why:** removes geo errors that break maps and spatial groupings.

### 5) SLA Metric — `HoursToClose`
- Compute `HoursToClose = Closed Date − Created Date` (hours), null if either date is missing.  
- Keep only plausible durations: **−0.1 to 720 hours** (~30 days).  
**Why:** trims noise/outliers and allows tiny negatives for clock skews.

### 6) Rolling Window (Last 6 Months)
Dynamically filter to **[Today − 6 months, Today)** so the model stays small and current:
```m
Today     = DateTime.LocalNow(),
StartDate = Date.AddMonths(Date.From(Today), -6),
Filtered  = Table.SelectRows(PreviousStep,
    each Date.From([Created Date]) >= StartDate and Date.From([Created Date]) < Date.From(Today)
)
```
> Change `-6` to `-12` or `-18` to widen the window later.

### 7) Column Pruning
- Drop near‑empty or redundant fields (e.g., `Due Date` (empty), verbose geo (`Location`, State Plane X/Y), `BBL`, park/bridge/taxi columns, and street‑level detail unless needed).  
**Why:** slimmer fact table, faster visuals, simpler model. Re‑enable any column if your story needs it.

### 8) Light De‑duplication (Near‑Duplicate Tickets)
- Create an **hour bucket** from `Created Date` using a compatibility‑safe method (no `DateTime.Hour` dependency):
```m
CreatedHour = DateTime.From(DateTime.ToText([Created Date], "yyyy-MM-dd HH:00:00"))
```
- Round coordinates (`LatRound`, `LngRound`) to **5 decimals** to group close points.  
- Sort by `Created Date` ascending, then keep the first row per composite key with `Table.Distinct`:
```
keys = { Complaint Type, Borough, LatRound, LngRound, CreatedHour }
```
**Why:** treats multiple tickets for the same complaint type, in the same place, within the same hour as duplicates; keeps the earliest. Avoids column collisions from Group/Expand.

### 9) Final Column Order
Reorder essentials for a tidy **FactRequests** table:
```
Unique Key, Created Date, Closed Date, Resolution Action Updated Date,
Agency, Complaint Type, Descriptor, Status,
Borough, City, Incident Zip, Location Type, Address Type,
Latitude, Longitude, HoursToClose
```

---

## What This Enables
- **Reliable KPIs:** `Requests`, `Closed Requests`, `Avg HoursToClose`, On‑Time %.
- **Trends & Seasonality:** clean monthly/daily/hourly analysis by complaint type and borough.
- **Maps:** valid lat/long only, with reduced duplicate clutter.
- **Performance:** smaller model via the rolling window and column pruning → quicker refresh & visuals.

---

## Notes & Tweaks
- **Adjust the window:** change `-6` to `-12`/`-18` months.  
- **Re‑include columns:** remove them from the `RemoveColumns` step.  
- **Time zone:** `DateTime.LocalNow()` uses machine locale; for UTC, use `DateTimeZone.UtcNow()` then convert to date.  
- **Path portability:** consider a **parameter** (Power Query → Manage Parameters) for the CSV path so collaborators can change it without editing the query.


---

## Purpose of This Report

This report demonstrates end‑to‑end analytics skills for a data‑analytics portfolio aimed at employers. It showcases:
- **Data quality engineering** (robust Power Query pipeline, schema typing, cleaning, de‑duplication).
- **Analytical modeling** (fact table with SLA metric, Date dimension, relationships).
- **Insightful visuals** focused on operational performance, temporal patterns, and geography.
- **User experience** features (slicers, navigation, dynamic titles) suitable for decision‑makers.

> Goal: present a compact, performant, and professional analysis of NYC 311 requests that highlights practical BI capabilities for real‑world roles.

## Report Pages Implemented (v1)

### 1) Executive Overview
**KPIs**: Total Requests, Closed Requests, Open Backlog, On‑Time %, Median Hours to Close, P90 Hours to Close.  
**Visuals**: Weekly Requests trend (Requests & Closed), Top 10 Complaint Types, Bubble Map (Requests with tooltips for On‑Time % and Median Hours).  
**Usage**: Read at a glance; filter with slicers to narrow by Borough/Agency/Complaint Type and watch KPIs/trend update.

### 2) Temporal Patterns
**Visuals**:  
- **Heatmap** (Matrix) — Requests by *Weekday × Hour* with conditional‑formatting color scale.  
- **Closing Velocity** — Combo chart with *Closed per Day* (columns) and *Avg Hours to Close* (line).  
**Usage**: Identify peak hours/days, seasonality, and whether closing speed keeps up with intake.

### 3) Geo & Neighborhoods
**Visuals**:  
- **Borough Comparison** — Requests by Borough with On‑Time % as secondary line.  
- **ZIP/City Table** — Requests, On‑Time %, Median Hours with conditional formatting.  
- **Map** — Requests by latitude/longitude (cleaned, validated geos).  
**Usage**: Spot hotspots and areas with slower resolution or lower on‑time performance.

## Interactivity & Navigation

- **Slicers**: Date (via DimDate), Borough, Agency, Complaint Type. These filter all visuals to keep context consistent.  
- **Navigation Buttons**: Page navigation buttons provide quick movement across the three pages (Executive Overview, Temporal Patterns, Geo & Neighborhoods).  
- **Dynamic Titles (optional)**: Title measures can reflect current slicer selections (e.g., “Top Complaints — *[Boroughs]* | *[Date Window]*”).

## How to Use the Report

1. **Pick a date range** (defaults to rolling window from the data).  
2. **Filter by Borough/Agency/Complaint Type** to focus analysis.  
3. **Overview** for KPIs and top categories; **Temporal Patterns** to see peaks (when); **Geo** to see hotspots (where).  
4. Hover tooltips on charts and map for Median Hours to Close and On‑Time %.  
5. Use navigation buttons to switch pages quickly.

## Notes for Hiring Managers

- The dataset was cleaned with a resilient Power Query pipeline (types, text hygiene, ZIP normalization, geo bounds, SLA duration, rolling window, and de‑dup).  
- The model includes a date dimension, core SLA measures, and report‑level interactivity tuned for performance and clarity.  
- This project reflects practical BI skills: data quality, model design, DAX, UX, and storytelling.

## Future Enhancements

- Parameterized **SLA threshold** (What‑If) to interactively test 24–168 hour targets.  
- **Incremental Refresh** for scalable, low‑latency updates.  
- **Anomaly detection** on spikes by complaint type or borough.  
- **Bookmarks/Drillthrough** for agency deep‑dives or neighborhood profiles.
