# RailPulse — Power BI Operations Dashboard

This is the direct continuation of my RailPulse Azure Challenge project. I built this Power BI dashboard on top of that same Azure pipeline (Azure Function → Azure SQL → `liveboard_records`) to answer the four core operational questions: punctuality, rush-hour bottlenecks, train-class performance, and station congestion. Everything below reflects the state of the report as of my latest refresh (07/08/2026, 9h20), with the numbers pulled directly from the current data model rather than estimated.

**Published:** this report is live in my Power BI Service workspace ("Railpulse") — [live link](https://app.powerbi.com/links/hAXgx48G73?ctid=33ac9060-aa07-4cb9-8fbc-4dcf3a48b1d7&pbi_source=linkShare).

## Table of Contents

- [1. Data Model](#1-data-model)
- [2. Report Pages](#2-report-pages)
  - [2.1 On Time Rate %](#21-on-time-rate-punctuality-scorecard)
  - [2.2 Rush Hours](#22-rush-hours-rush-hour-matrix)
  - [2.3 Delay by Train Category](#23-delay-by-train-category-train-class-breakdown)
  - [2.4 Stations Congestion Map](#24-stations-congestion-map-platformstation-congestion-map)
  - [2.5 Top 10 Worst-Performing Stations](#25-top-10-worst-performing-stations)
  - [2.6 Avg Delay for Major Hubs](#26-avg-delay-for-major-hubs-cross-hub-comparison)
  - [2.7 On-Time Rate by Day of Week](#27-on-time-rate-by-day-of-week)
  - [2.8 Delay Severity Breakdown](#28-delay-severity-breakdown)
  - [2.9 Delay Concentration](#29-delay-concentration)
  - [2.10 Delay Severity by Train Category, Date and Hour](#210-delay-severity-by-train-category-date-and-hour)
  - [2.11 Micro Report](#211-micro-report-drill-down-navigation-hidden-drillthrough-target)
- [3. DAX Measures](#3-dax-measures)
- [4. Nice-to-Have Coverage](#4-nice-to-have-coverage)
- [5. Top 3 Tactical Recommendations](#5-tactical-recommendations)
- [6. What I Can Actually Do With This](#6-what-i-can-actually-do-with-this)
- [7. Final Conclusion](#7-final-conclusion)

## 1. Data Model

I connect the report to Azure SQL Database (`serv-railpulse.database.windows.net`, database `db-railpulse`) for the live tables, and to Azure Blob Storage for the two static GTFS reference tables — `routes` and `trips` are served from `railpulsechallenge97d6.blob.core.windows.net/powerbi-gtfs-static/`, authenticated with my BeCode Microsoft Entra ID account. This was the last piece I needed to make the whole pipeline cloud-native end to end.

| Table | Source | Role |
|---|---|---|
| `liveboard_records` | Azure SQL | Fact table — one row per real-time GTFS-RT departure event (**980,868 rows** as of my latest refresh) |
| `stations` | Azure SQL | Dimension — station reference data (name, lat/long). 1,856 stations total in this table, though only 598 of them actually appear in the live fact table — the rest exist in the GTFS static reference but haven't produced a real-time departure event yet during my collection window |
| `vehicles` | Azure SQL | Dimension — vehicle type/direction. 3,617 vehicles, matching exactly the number of distinct `vehicle_id` values in the fact table |
| `routes` | Azure Blob Storage (`routes.txt`) | Dimension — `train_category` (IC / S / P / L / NJ / EC / T / BUS / EXT / OTC), 1,801 routes |
| `trips` | Azure Blob Storage (`trips.txt`) | Bridge table, `trip_id` → `route_id`, 134,809 trips |
| `_Calculations` | N/A | Measures-only table |
| `Date` | Calculated table (`CALENDAR()`), marked as Date Table | Date dimension for trend visuals, related to `liveboard_records[Scheduled Date]` |

**Join coverage:** `trip_id` matches trips[trip_id] for about 86% of rows — the rest are excluded from category breakdowns rather than shown unlabeled.

**My honest read on the data window.** I've collected **12 days** of polling so far (27 Jul – 7 Aug 2026) — not much for trend analysis. The "Daily Change" KPI computes correctly, but I wouldn't call it a trend indicator yet — I frame it as "the mechanism works and is live," not "the network is improving," until several more days (or weeks, depending of my free credits) smooth out the noise.

**Date relationship fix I applied:** I ran into an issue where `scheduled_time` (which includes the time of day) wouldn't relate to my pure-date `Date` table — Power BI just refused to link them properly. My fix was to add a calculated column, `Scheduled Date`, that strips out the time and keeps just the date, then I related that to `Date[Date]` instead.

## 2. Report Pages

11 pages, in this order:

### 2.1 On Time Rate % — Must-Have: Punctuality Scorecard

Landing/exec page: two gauges (All Network / Major Hubs, each driven by its own independent slicer — `Stations Name` for the network view, a dedicated `Major Hubs` slicer restricted to the 5 flagship stations for the hub view, with cross-interactions deliberately disabled between them so neither slicer affects the other's gauge), plus Total Records, Canceled Trains, the "Daily Change" trend KPI, and the standard 3-filter slicer set.

**Conclusion:** My network-wide on-time rate sits at **78.3%** across all 980,868 records I've collected, which is a meaningful gap below the 90% objective I set out to measure against. What strikes me looking at this page is how much that single network-wide number hides — once I break it down by hub (§2.6), individual stations swing from the mid-60s to the high-80s, so "78.3%" is really an average of some fairly different local realities rather than a uniform experience across the network.

### 2.2 Rush Hours — Must-Have: Rush Hour Matrix

Combo chart (`lineClusteredColumnComboChart`): Train Volume (columns) vs Avg Delay (line) by Hour.

**Conclusion:** Volume peaks in the early-to-mid afternoon (14:00–16:00), but delay peaks later, in the 17:00–21:00 window, roughly double the midday baseline. That mismatch is the most interesting finding here — the network isn't slowest when it's busiest, it's slowest later in the day. I checked whether Nightjet services (my worst-performing category) were driving this, but they're a small enough slice of evening traffic (~5,8%) that the pattern holds without them — so the cause is still an open question for me.

### 2.3 Delay by Train Category — Must-Have: Train Class Breakdown

Combo chart: Total Delayed Minutes + Avg Delay (min) by `train_category`.

**Conclusion:** IC and S trains dominate total delay volume — about 698,000 and 487,000 delayed minutes respectively — simply because they're the categories running the most trips. But Nightjet is the real outlier on a per-trip basis: only 4,252 records in my dataset, yet an average delay of nearly **17 minutes**, several times worse than any other category. I'd flag that as the single most actionable number on this page — it's a small, low-volume service, but almost every Nightjet departure I've recorded runs meaningfully late (see §2.10).

### 2.4 Stations Congestion Map — Must-Have: Station Congestion Map

Clustered bar chart (Total Delayed Minutes by station) + Azure Map (bubble size = Total Delayed Minutes, positioned by lat/long). Canvas deliberately larger (1250×1600) to fit the map.

**Conclusion:** Congestion is heavily concentrated rather than spread evenly across my 598-station network — the three Brussels hubs (Nord, Midi, Central) alone account for close to 180,000 of my roughly 1.7 million total delayed minutes, despite representing a tiny fraction of the stations I'm tracking. Looking at the map, it's visually obvious that the network's delay problem is really a "handful of major junctions" problem rather than a diffuse, network-wide one.

### 2.5 Top 10 Worst-Performing Stations

Bar chart, Avg Delay by station, all stations.

**Conclusion:** This is where the international/border stations show up — Rosenheim (DE) tops my list at nearly 43 minutes average delay, followed by München Ost and Koblenz Hbf, all well above anything in the Belgian core network. I read this less as "SNCB is failing these stations" and more as: these are the last stops on long cross-border lines, so delay that piled up somewhere earlier on the route probably just shows up here at the end. I can't confirm that from this data alone, but it's the explanation that makes the most sense to me (worth keeping in mind before using this chart to point fingers).

### 2.6 Avg Delay for Major Hubs: Cross-Hub Comparison

Scatter chart: Avg Delay × Train Volume × Total Delayed Minutes, one bubble per major hub. 

**Conclusion:** Bruxelles-Central is clearly my weakest major hub — around 66% on-time and roughly 2.2 minutes average delay, noticeably worse than Charleroi-Central (87% on-time, ~1.1 min) and Gand-Saint-Pierre (84% on-time, ~1.3 min), which are my two best performers. Anvers-Central and Liège-Guillemins sit in between. What I take from this chart is that "major hub" doesn't mean "equally strained" — Bruxelles-Central is carrying a disproportionate share of my delay problem relative to its four peers.

### 2.7 On-Time Rate by Day of Week

Line chart, On-Time Rate % across the week.

**Conclusion:** Monday and Friday are my two weakest days, both sitting around 75–76% on-time, while Sunday is by far my strongest at nearly 86%. That pattern lines up with intuition — weekday commuter pressure at the start and end of the working week versus a much lighter, less schedule-dependent Sunday service — but I'd want more weeks of data before treating this as a stable weekly signature rather than a one-off pattern from my 12 days window.

### 2.8 Delay Severity Breakdown

Donut chart, records by severity band, fixed color palette, matched on the Micro Report bar chart using the same page.

**Conclusion:** 78.3% of my records fall in the On Time band, 13.9% are Minor delays (2–5 min), and the remaining 7.7% split between Moderate (5.7%) and Severe (2.0%). What's notable to me is that the Moderate and Severe bands — under 8% of records combined — still account for a hugely disproportionate share of total delayed minutes, which is exactly what §2.9 makes visible.

### 2.9 Delay Concentration 

Pivot table, Hour × Day of Week, values = Total Delayed Minutes.

**Conclusion:** Wednesday alone accounts for roughly **510,000 of my 1.70 million total delayed minutes** — close to a third of everything I've recorded, concentrated in a single day of the week. Drilling into the worst single cell, Wednesday at 17:00 stands out at about 48,000 delayed minutes, and that same 17:00 hour totals close to 159,000 minutes across the whole week — by a wide margin the single worst recurring time slot in my entire dataset.

### 2.10 Delay Severity by Train Category, Date and Hour

Matrix, cross-tabulating severity against category, date, and hour.

**Conclusion:** I built this page as my own drill-in reference — somewhere I can go to defend or double-check any number if it gets challenged during a walkthrough, not a headline chart on its own.

### 2.11 Micro Report: Drill-Down Navigation (hidden drillthrough target)

The only page I configured with `pageBinding: Drillthrough`, cleaned up to exactly two fields: `train_category` and `Stations Name`. Hidden from the tab bar. Contains: `Selected Station` title card, 4 KPI cards with conditional red/orange/green formatting, an Hour trend line, a day-of-week trend line, and two bar charts (Delay Severity, Avg Delay by train_category). 

**Conclusion:** This page lets me click from any chart straight to "why is this specific station or category underperforming" without leaving the report. It's genuinely useful, I can answer a question in the moment instead of stopping to build a new visual.

## 3. DAX Measures

```DAX
-- Punctuality Scorecard -----------------------------------------------------

On-Time Rate % =
DIVIDE(
    CALCULATE(
        COUNTROWS(liveboard_records),
        liveboard_records[delay_seconds] < 120,
        liveboard_records[canceled] = FALSE
    ),
    CALCULATE(
        COUNTROWS(liveboard_records),
        liveboard_records[canceled] = FALSE
    )
)

Total Records = COUNTROWS(liveboard_records)

Canceled Trains = 
CALCULATE(COUNTROWS(liveboard_records), liveboard_records[canceled] = TRUE)
-- Always 0 across all rows: the live GTFS-RT feed does not populate this field, or no train cancelled during my 12 days window...

Target_Objectif = 0.9

-- Rush Hour Matrix ------------------------------------------------------------

Train Volume = COUNTROWS(liveboard_records)
Avg Delay (sec) = AVERAGE(liveboard_records[delay_seconds])
Avg Delay (min) = AVERAGE(liveboard_records[delay_seconds]) / 60

-- Congestion / Concentration --------------------------------------------------

Total Delayed Minutes = DIVIDE(SUM(liveboard_records[delay_seconds]), 60)

-- Trend ------------------------------------------------------------------------
-- Compares yesterday to the day before (not "today", which is always a partial
-- day at refresh time and would make the delta meaningless).

Trend vs Previous Period =
VAR LatestDate = MAX('Date'[Date]) - 1
VAR CurrentRate = CALCULATE([On-Time Rate %], 'Date'[Date] = LatestDate)
VAR PreviousRate = CALCULATE([On-Time Rate %], 'Date'[Date] = LatestDate - 1)
RETURN CurrentRate - PreviousRate

Zero Baseline = 0   -- neutral target used by the "Daily Change" KPI visual

-- Drill-through detail ---------------------------------------------------------

Selected Station =
VAR StationName = SELECTEDVALUE(liveboard_records[Stations Name])
RETURN IF(ISBLANK(StationName), "No station selected", StationName)

Severe Delay Count =
CALCULATE(COUNTROWS(liveboard_records), liveboard_records[Delay Severity] = "Severe (>15min)")

-- Conditional formatting (wired to Micro Report cards) ------------------------

On-Time Rate Status Color =
VAR CurrentRate = [On-Time Rate %]
RETURN SWITCH(TRUE(),
    CurrentRate >= [Target_Objectif], "#16F755",
    CurrentRate >= [Target_Objectif] - 0.15, "#FFA500",
    "#FF0000"
)

Avg Delay Status Color =
VAR D = [Avg Delay (min)]
RETURN SWITCH(TRUE(),
    D <= 1.5, "#16F755",
    D <= 2.5, "#FFA500",
    "#FF0000"
)

Severe Delay Status Color =
VAR S = [Severe Delay Count]
VAR RatioSevere = DIVIDE(S, [Total Records])
RETURN SWITCH(TRUE(),
    RatioSevere <= 0.02, "#16F755",
    RatioSevere <= 0.04, "#FFA500",
    "#FF0000"
)
```
**Why the color thresholds don't match Delay Severity's bands:** Delay Severity classifies a single trip; Avg Delay Status Color classifies a station-level average, and averages compress toward the middle — a station with plenty of severely late trains can still average under 2 minutes. Reusing the severity bands here would leave almost every station green, so I calibrated against my actual data instead: 1.5 min sits near my median station, 2.5 min near my 75th percentile — splitting stations into roughly even green/orange/red groups based on real performance, not arbitrary numbers.

`Delay Severity` is **not a DAX measure** — it's a calculated column I added in Power Query (M), on the `liveboard_records` query:

```
if [delay_seconds] < 120 then "On Time (<2min)"
else if [delay_seconds] < 300 then "Minor (2-5min)"
else if [delay_seconds] <= 900 then "Moderate (5-15min)"
else if [delay_seconds] > 900 then "Severe (>15min)"
else null
```

## 4. Nice-to-Have Coverage

| Feature | Status |
|---|---|
| Drill-through navigation | ✅ Done — Micro Report, hidden target. |
| Scheduled cloud refresh | ✅ Done — `routes` and `trips` read from Azure Blob Storage, authenticated via my Entra ID account. Scheduled refresh is live in Power BI Service (daily), tested successfully end to end |
| Cross-Hub Comparison | ✅ "Avg Delay for Major Hubs" — all 5 major hubs, filterable, consistent field usage |

## 5. Tactical Recommendations

1. **Bruxelles-Central is the outlier among my five major hubs** — noticeably worse than Charleroi-Central and Gand-Saint-Pierre on both delay and on-time rate. If I had to prioritize one place to intervene, targeted dispatch or track-allocation fixes here would likely yield the largest system-wide punctuality gain per euro invested, given how much total delay this single station contributes.
2. **Wednesday 17:00 is my single worst recurring bottleneck** — around 48,000 delayed minutes in that one cell alone, and Wednesday as a whole accounts for nearly a third of all delayed minutes I've recorded. Reallocating staffing or buffer time specifically to this slot would address the largest concentrated dip in my entire dataset.

## 6. What I Can Actually Do With This

Now that the pipeline is fully cloud-native — live ingestion into Azure SQL, static reference data on Azure Blob, scheduled refresh with no local dependency — I see a few concrete uses for this dashboard beyond the bootcamp deliverable itself:

- **A live talking point for interviews.** This is a real, working, end-to-end data product I can pull up and walk someone through: ingestion, modeling, DAX, and a decision-support layer on top. It demonstrates the full stack rather than just a notebook.
- **An early-warning tool, once the data window grows.** The mechanism for tracking on-time rate, delay concentration, and category performance is already built and refreshing daily. Given a few more days (weeks) of accumulation, the "Daily Change" trend KPI and day-of-week patterns would start to mean something statistically, rather than just reflecting single-day noise the way they do today.
- **A template I can reuse.** The pattern I landed on here — GTFS-RT ingestion via Azure Functions, Azure SQL as the fact store, static reference data on Blob Storage, Power BI on top with a documented DAX layer — isn't specific to Belgian rail. I could point the same architecture at any other public real-time transit feed.

## 7. Final Conclusion

What matters most to me looking back at this build isn't any individual chart — it's that the pipeline runs as one closed loop from end to end. Real-time GTFS-RT data lands in Azure SQL, static reference data lives on Azure Blob, Power BI refreshes on its own schedule with no local file and no gateway keeping it alive, and every number on every page traces back to that same live source. That was the actual goal, more than any individual insight.

The insights themselves are real, even with only 12 days behind them: Bruxelles-Central underperforms its peer hubs, Wednesday evenings are my worst recurring bottleneck, and Nightjet services run late far more than their small volume would suggest. I trust those patterns enough to act on them operationally, but I'm deliberately not overselling the trend layer — a 12 days window is a starting point, not a verdict, and I'd rather say that plainly than let a "Daily Change" arrow imply more certainty than the data currently supports. If I keep this pipeline running, the same dashboard should only get more useful with time, without me having to change a single measure.
