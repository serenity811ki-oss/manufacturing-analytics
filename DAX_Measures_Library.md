# Manufacturing Analytics Dashboard — DAX Measures Library

Paste these into a dedicated **_Measures** table in Power BI (Modeling → New Table → name it
`_Measures`, delete the placeholder column, then add each measure to it). Keep all measures in
one home table — it makes them easy to find and keeps the visual fact tables clean.

Table/column names below assume the sample star schema:
`FactProduction`, `FactDowntime`, `FactMaintenance`, `FactFailureEvents`,
`DimDate`, `DimMachine`, `DimLine`, `DimShift`, `DimProduct`, `DimDowntimeReason`, `DimMaintenanceType`.
Adjust names if your real source system differs.

---

## 1. Production Volume

```DAX
Total Units Produced = SUM(FactProduction[UnitsProduced])

Good Units = SUM(FactProduction[GoodUnits])

Scrap Units = SUM(FactProduction[ScrapUnits])

Target Units = SUM(FactProduction[TargetUnits])

Production Efficiency % =
DIVIDE([Total Units Produced], [Target Units], 0)
```

## 2. Time Buckets (basis for OEE)

```DAX
Planned Production Time (min) = SUM(FactProduction[PlannedTimeMin])

Downtime (min) = SUM(FactProduction[DowntimeMin])

Run Time (min) = SUM(FactProduction[RunTimeMin])

Idle Time (min) = MAX(0, [Planned Production Time (min)] - [Run Time (min)] - [Downtime (min)])
```

## 3. OEE — Availability, Performance, Quality

```DAX
Availability % =
DIVIDE([Run Time (min)], [Planned Production Time (min)], 0)

-- Ideal Cycle Time must be brought in via a relationship or LOOKUPVALUE to DimMachine
Ideal Run Rate (units/min) =
DIVIDE(60, AVERAGE(DimMachine[IdealCycleTimeSec]))

Performance % =
DIVIDE(
    [Total Units Produced],
    [Run Time (min)] * [Ideal Run Rate (units/min)],
    0
)

Quality % =
DIVIDE([Good Units], [Total Units Produced], 0)

OEE % =
[Availability %] * [Performance %] * [Quality %]
```

> **Design note:** Performance% and Quality% can occasionally exceed 100% with noisy sample
> data (e.g. a machine running faster than its rated ideal cycle time). In production data this
> is rare; if it happens, wrap the measure in `MIN(1, ...)` to cap it, or investigate the
> underlying cycle-time master data — a >100% reading is usually a data quality signal, not a
> real result.

## 4. Utilization & Downtime

```DAX
Utilization % =
DIVIDE([Run Time (min)], [Planned Production Time (min)], 0)

Total Downtime (min) = SUM(FactDowntime[DurationMin])

Planned Downtime (min) =
CALCULATE([Total Downtime (min)], FactDowntime[Category] = "Planned")

Unplanned Downtime (min) =
CALCULATE([Total Downtime (min)], FactDowntime[Category] = "Unplanned")

Downtime % =
DIVIDE([Total Downtime (min)], [Planned Production Time (min)], 0)

Unplanned Downtime % =
DIVIDE([Unplanned Downtime (min)], [Total Downtime (min)], 0)
```

## 5. Reliability — MTBF / MTTR / RUL

```DAX
Failure Count = COUNTROWS(FactFailureEvents)

Total Repair Time (hrs) = SUM(FactFailureEvents[RepairTimeHrs])

-- Operating hours between the first and last date in the current filter context
Operating Days in Period =
DATEDIFF(MIN(DimDate[Date]), MAX(DimDate[Date]), DAY) + 1

Operating Hours in Period =
[Operating Days in Period] * 24

MTBF (hrs) =
DIVIDE(
    [Operating Hours in Period] - [Total Repair Time (hrs)],
    [Failure Count],
    BLANK()
)

MTTR (hrs) =
DIVIDE([Total Repair Time (hrs)], [Failure Count], BLANK())

-- Simple heuristic RUL score (0-100, higher = healthier); replace with a
-- real ML/Azure ML output when available — see Predictive Analytics page notes.
Predictive Health Score =
VAR RecentFailures =
    CALCULATE(
        [Failure Count],
        DATESINPERIOD(DimDate[Date], MAX(DimDate[Date]), -30, DAY)
    )
VAR AvgVibration = AVERAGE(FactFailureEvents[VibrationRMS_mm_s])
VAR Score = 100 - (RecentFailures * 8) - (AvgVibration * 4)
RETURN
    MAX(0, MIN(100, Score))
```

## 6. Quality

```DAX
Scrap % =
DIVIDE([Scrap Units], [Total Units Produced], 0)

Defect Rate (per 1000) =
DIVIDE([Scrap Units], [Total Units Produced], 0) * 1000

First Pass Yield % =
DIVIDE([Good Units], [Total Units Produced], 0)

-- Requires a rework flag/quantity field if tracked separately from scrap;
-- shown here assuming a FactProduction[ReworkUnits] column if you add one.
Rework % =
DIVIDE(SUM(FactProduction[ReworkUnits]), [Total Units Produced], 0)
```

## 7. Maintenance

```DAX
Maintenance Cost = SUM(FactMaintenance[Cost])

PM Cost =
CALCULATE([Maintenance Cost], FactMaintenance[MaintTypeID] = "PM")

CM Cost =
CALCULATE([Maintenance Cost], FactMaintenance[MaintTypeID] = "CM")

PdM Cost =
CALCULATE([Maintenance Cost], FactMaintenance[MaintTypeID] = "PdM")

Open Work Orders =
CALCULATE(COUNTROWS(FactMaintenance), FactMaintenance[Status] = "Open")

Completed Work Orders =
CALCULATE(COUNTROWS(FactMaintenance), FactMaintenance[Status] = "Completed")

Scheduled Work Orders =
CALCULATE(COUNTROWS(FactMaintenance), FactMaintenance[Status] = "Scheduled")

Avg Maintenance Duration (hrs) =
AVERAGE(FactMaintenance[DurationHrs])

PM Compliance % =
DIVIDE(
    CALCULATE(COUNTROWS(FactMaintenance), FactMaintenance[MaintTypeID]="PM", FactMaintenance[Status]="Completed"),
    CALCULATE(COUNTROWS(FactMaintenance), FactMaintenance[MaintTypeID]="PM"),
    0
)
```

## 8. Time Intelligence

> Requires `DimDate` marked as the model's **Date Table** (Table tools → Mark as Date Table),
> related to fact tables on `DateKey`/`Date`.

```DAX
Units Produced PY =
CALCULATE([Total Units Produced], SAMEPERIODLASTYEAR(DimDate[Date]))

YoY Growth % =
DIVIDE([Total Units Produced] - [Units Produced PY], [Units Produced PY], 0)

Units Produced PM =
CALCULATE([Total Units Produced], DATEADD(DimDate[Date], -1, MONTH))

MoM Growth % =
DIVIDE([Total Units Produced] - [Units Produced PM], [Units Produced PM], 0)

Running Total Units =
CALCULATE(
    [Total Units Produced],
    FILTER(ALLSELECTED(DimDate[Date]), DimDate[Date] <= MAX(DimDate[Date]))
)

OEE Trailing 7-Day Avg =
AVERAGEX(
    DATESINPERIOD(DimDate[Date], MAX(DimDate[Date]), -7, DAY),
    [OEE %]
)
```

## 9. KPI Status / Conditional Formatting Helpers

```DAX
OEE Status =
VAR v = [OEE %]
RETURN
    SWITCH(
        TRUE(),
        v >= 0.85, "On Target",
        v >= 0.65, "Watch",
        "At Risk"
    )

OEE Status Color =
VAR v = [OEE %]
RETURN
    SWITCH(
        TRUE(),
        v >= 0.85, "#2E7D32",   -- green
        v >= 0.65, "#F9A825",   -- amber
        "#C62828"               -- red
    )

Scrap % Status Color =
VAR v = [Scrap %]
RETURN
    SWITCH(
        TRUE(),
        v <= 0.02, "#2E7D32",
        v <= 0.05, "#F9A825",
        "#C62828"
    )
```

Use `OEE Status Color` / `Scrap % Status Color` in a card visual's **Conditional formatting →
Font color → Format by: Field value** to drive red/amber/green KPI cards without extra visuals.

## 10. Pareto Chart Support (Downtime Root Cause)

```DAX
Downtime by Reason = SUM(FactDowntime[DurationMin])

Downtime Cumulative % =
VAR CurrentReasonTotal = [Downtime by Reason]
VAR AllReasonsTable =
    ALLSELECTED(DimDowntimeReason[ReasonDescription])
VAR RankOfCurrent =
    RANKX(AllReasonsTable, CALCULATE([Downtime by Reason]), , DESC)
VAR CumulativeTotal =
    CALCULATE(
        [Downtime by Reason],
        TOPN(RankOfCurrent, AllReasonsTable, CALCULATE([Downtime by Reason]), DESC)
    )
VAR GrandTotal = CALCULATE([Downtime by Reason], ALLSELECTED(DimDowntimeReason))
RETURN
    DIVIDE(CumulativeTotal, GrandTotal, 0)
```

Build the Pareto as a combo chart: clustered column = `Downtime by Reason` (sorted descending),
line = `Downtime Cumulative %` on a secondary axis fixed 0–100%.

---

## Formatting conventions used throughout

| Measure type | Format string |
|---|---|
| Percent (OEE, Availability, Scrap %...) | `0.0%` |
| Units | `#,##0` |
| Currency (Cost) | `$#,##0` |
| Hours (MTBF/MTTR) | `#,##0.0 "hrs"` |

Set these on each measure under **Modeling → Format** so cards and axis labels render correctly
without extra DAX.
