# Manufacturing Analytics Dashboard — Power BI Build Guide

This guide walks you from the provided assets to a finished `.pbix`. It assumes Power BI Desktop
(free) is installed. Total build time following this guide: roughly 3–5 hours for an experienced
Power BI user, longer if you're building visuals for the first time.

**What's provided:**
- `Manufacturing_Sample_Dataset.xlsx` — 11-sheet star schema sample data
- `DAX_Measures_Library.md` — every measure referenced in this guide
- `Manufacturing_Theme.json` — importable custom theme (blue-gray industrial)
- This guide

---

## Step 1 — Import the data

1. Power BI Desktop → **Get Data → Excel Workbook** → select `Manufacturing_Sample_Dataset.xlsx`.
2. In Navigator, check every sheet **except README**, click **Transform Data** (not Load
   directly — you want to fix types first).
3. In Power Query, for each table confirm column data types: `DateKey` as Whole Number,
   `Date` as Date, `Cost`/`UnitCost` as Fixed Decimal, all ID columns as Text.
4. **Close & Apply.**

When you replace this with real data later, swap the source step in Power Query for your
SQL Server / PostgreSQL / REST API connector — the table and column names should stay the same
so the model and DAX below don't need rework. For SQL sources, push filtering and aggregation
down where possible (**Query Folding**) rather than transforming after load.

## Step 2 — Build the star schema relationships

Go to **Model view** and create these relationships (all Many-to-One, single direction unless noted):

| From (fact) | Column | To (dim) | Column |
|---|---|---|---|
| FactProduction | DateKey | DimDate | DateKey |
| FactProduction | MachineID | DimMachine | MachineID |
| FactProduction | LineID | DimLine | LineID |
| FactProduction | ShiftID | DimShift | ShiftID |
| FactProduction | ProductID | DimProduct | ProductID |
| FactDowntime | DateKey | DimDate | DateKey |
| FactDowntime | MachineID | DimMachine | MachineID |
| FactDowntime | ReasonID | DimDowntimeReason | ReasonID |
| FactMaintenance | DateKey | DimDate | DateKey |
| FactMaintenance | MachineID | DimMachine | MachineID |
| FactMaintenance | MaintTypeID | DimMaintenanceType | MaintTypeID |
| FactFailureEvents | DateKey | DimDate | DateKey |
| FactFailureEvents | MachineID | DimMachine | MachineID |
| DimMachine | LineID | DimLine | LineID |

Also relate `DimMachine[LineID] → DimLine[LineID]` so line-level slicers filter machine-level
visuals. Keep every fact-to-dim relationship **single direction** (dims filter facts) to avoid
ambiguous filter paths — this matters once you have four fact tables.

**Mark DimDate as the date table:** select DimDate → **Table tools → Mark as Date Table →**
column `Date`. This enables the time-intelligence measures (YoY, MoM, running totals).

## Step 3 — Add the measures

Create a blank table named `_Measures` (Modeling → New Table → `_Measures = ROW("x", 1)`, then
delete the placeholder column). Paste in every measure from `DAX_Measures_Library.md`. Organize
them into display folders (right-click a measure → **Display Folder**) matching the library's
numbered sections: `1. Production`, `2. Time`, `3. OEE`, `4. Downtime`, `5. Reliability`,
`6. Quality`, `7. Maintenance`, `8. Time Intelligence`, `9. KPI Status`.

Apply number formats as listed in the library's final table (percent, currency, hours).

## Step 4 — Apply the theme

**View → Themes → Browse for themes** → select `Manufacturing_Theme.json`. This sets the
blue-gray palette, card typography, rounded card borders, and light-gray page background used
across every page below.

## Step 5 — Build the pages

Create 7 pages (right-click page tab → Rename). Suggested canvas size: 16:9. Add a consistent
**header band** on every page (rectangle, `#1F3864` fill, page title as text box, small factory
icon top-left) — copy it page to page for visual consistency, or better, build it once and use
Power BI's **page template** feature.

### Page 1 — Executive Overview
- **Top row, 6 KPI cards**: Total Production, OEE %, Machine Utilization %, Downtime %,
  Production Efficiency %, Scrap Rate % — use the `New Card` visual (supports trend sparkline +
  target) bound to the matching measure, with `OEE Status Color` etc. driving conditional font
  color.
- **Middle row**: OEE trend line chart (`OEE %` by `DimDate[Date]`, with `OEE Trailing 7-Day Avg`
  as a second line); Production by Line clustered bar.
- **Bottom row**: Downtime Pareto (see Page 5); a **multi-row card / matrix "Executive Summary"**
  listing top 3 issues — worst-OEE machine, highest scrap-rate product, largest single downtime
  event this period, computed with `TOPN` measures or a small summary table visual sorted
  descending.
- Add a **date range slicer** and **Line slicer** pinned top-right, synced to all pages
  (Format → Sync slicers).

### Page 2 — Production Analytics
- Daily/Weekly/Monthly toggle: build one line chart bound to `Total Units Produced`, add a
  **field parameter** (Modeling → New Parameter → Fields) letting the user swap the date
  hierarchy grain, or simpler — three small multiple charts side by side.
- Production by Line, by Machine, by Shift — three bar charts, each with drill-down enabled
  (right-click axis → add drill hierarchy Line → Machine).
- Throughput Analysis: scatter chart, X = `Run Time (min)`, Y = `Total Units Produced`, size =
  machine, to visually flag under/over performers relative to the ideal-rate line.

### Page 3 — Machine Performance
- Machine status matrix: `DimMachine[MachineName]` rows, columns = Runtime / Idle / Downtime /
  Utilization %, with data bars (conditional formatting → data bars) on Utilization %.
- Top 5 / Bottom 5 performing machines: two horizontal bar charts sorted by `OEE %`, using a
  **Top N filter** (Filters pane → Filter type: Top N).
- **Gauge chart** for fleet-average Utilization % against an 85% target line.
- Drillthrough: right-click a machine bar → set up a **drillthrough page** ("Machine Detail")
  showing that single machine's full history — required field DimMachine[MachineID] added to
  the drillthrough filters well on the target page.

### Page 4 — Maintenance Dashboard
- KPI cards: Open Work Orders, Completed Work Orders, Maintenance Cost (period), PM Compliance %.
- Donut chart: work order count by `DimMaintenanceType[MaintTypeName]` (PM/CM/PdM split).
- Maintenance Cost trend area chart by month, stacked by maintenance type.
- Maintenance History: matrix/table of `FactMaintenance` sorted by date descending — WorkOrderID,
  Machine, Type, Status, Technician, Duration, Cost.
- Upcoming Maintenance Schedule: table filtered to `Status = "Scheduled"`, sorted by
  `ScheduledDateKey`.

### Page 5 — Downtime Analysis
- Planned vs Unplanned Downtime: stacked bar by `DimDate[MonthShort]`.
- **Pareto chart** (see DAX library §10): combo chart, bars = downtime minutes by reason
  descending, line = cumulative %, secondary axis 0–100%, with a reference line at 80%
  (Format → Analytics → Constant line) to visually mark the classic "vital few" cutoff.
- Downtime by Machine / by Line — bar charts.
- Downtime Trend — line chart over time, split by Category (Planned/Unplanned) using legend.
- **Root Cause matrix**: rows = `DimDowntimeReason[ReasonDescription]`, columns = `DimLine`, values
  = downtime minutes, as a heat-map-styled matrix (Format → Cell elements → background color
  scale).

### Page 6 — Quality Dashboard
- KPI cards: Scrap %, Defect Rate (per 1000), First Pass Yield %.
- Quality by Product: bar chart of `Scrap %` by `DimProduct[ProductName]`.
- Quality Trend: line chart of `First Pass Yield %` over time.
- Scrap Analysis: treemap, size = Scrap Units, grouped by Line then Machine.

### Page 7 — Predictive Analytics
- KPI cards: MTBF (hrs), MTTR (hrs), Predictive Health Score (fleet average).
- Failure Prediction: table of machines ranked by `Predictive Health Score` ascending (most at
  risk first), with conditional-formatting icons (red/amber/green) driven by the score.
- MTBF/MTTR trend line chart over time by machine (use a machine slicer to focus one at a time).
- Scatter chart: `VibrationRMS_mm_s` (X) vs `TemperatureC` (Y) from `FactFailureEvents`, colored
  by `FailureMode`, to visually surface sensor-signature clusters ahead of failures.
- **Note on "prediction":** the sample `Predictive Health Score` measure is a transparent
  heuristic (recent failure count + vibration), not a trained model. For genuine failure
  prediction, train a model outside Power BI (Azure ML, Python/R) on your real sensor and
  failure history, then bring the model's *output* (a risk score per machine per day) back into
  Power BI as a table via a scheduled pipeline — Power BI itself does not train predictive
  models, only the R/Python-visual and AI-visual integrations described below.

## Step 6 — Interactivity features

- **Slicers:** Date range, Line, Machine, Shift, Product on Executive Overview; sync to all
  pages you want them to persist across (View → Sync Slicers pane).
- **Drill-down:** enabled by default when a bar/column chart's axis has a hierarchy
  (Line → Machine, or Date → Year → Quarter → Month → Day from DimDate). Turn on the
  double-down-arrow "Drill Down" icon in the visual header.
- **Drillthrough:** build the "Machine Detail" page described in Page 3; repeat the pattern for
  a "Product Detail" drillthrough off Page 6 if useful.
- **Tooltips:** create small "Tooltip pages" (Page settings → Page type: Tooltip, size
  300x200px) with a mini KPI card + sparkline, then assign under a visual's Format →
  Tooltip → Type: Report page.
- **Bookmarks:** capture 2–3 states (e.g. "This Month" vs "This Quarter" filter states, or a
  "Focus: Unplanned Downtime" view with the Category slicer pre-set) via the Bookmarks pane, and
  expose them as buttons in the header band.
- **Dynamic titles:** bind a text box's Format → Title (or a card) to a measure such as
  `"OEE — " & SELECTEDVALUE(DimLine[LineName], "All Lines")` so the title updates with slicer
  selection.
- **Conditional formatting:** already covered via the `*Status Color` measures in the DAX
  library — apply them to card font color and matrix/table cell background.

## Step 7 — Row-Level Security (RLS)

1. **Modeling → Manage roles → Create role**, e.g. `Line Manager`.
2. On `DimLine`, add a DAX filter: `[LineName] = USERPRINCIPALNAME()` if you maintain a
   mapping table (`DimUserLineAccess[Email]`/`[LineName]`) — more typically:
   ```DAX
   DimLine[LineID] IN
   CALCULATETABLE(
       VALUES(DimUserLineAccess[LineID]),
       DimUserLineAccess[Email] = USERPRINCIPALNAME()
   )
   ```
   (Add a `DimUserLineAccess` table with columns `Email`, `LineID` to the model for this.)
3. Test via **Modeling → View as roles**.
4. After publishing to the Service, assign workspace members to roles under
   **Dataset → Security**.

## Step 8 — Mobile layout

**View → Mobile Layout**, drag the KPI cards and one or two priority charts per page onto the
phone canvas in a single column, largest/most important first. Do this for at least the
Executive Overview and Machine Performance pages — those are what a supervisor checks on the
floor.

## Step 9 — Scheduled refresh (after publishing)

1. Publish (**Home → Publish**) to a Power BI Service workspace.
2. If your source is on-prem SQL Server/PostgreSQL, install and configure an **On-premises Data
   Gateway** (Power BI Service → Settings → Manage Gateways).
3. Dataset → **Settings → Scheduled refresh** → set credentials, frequency (e.g. every 4 hours
   during production shifts), and failure-notification email.
4. For a REST API source, confirm the connector uses a refreshable credential (API key/OAuth) —
   avoid hardcoded tokens in the M query; use a parameter and set it as a **Dataset Parameter**
   in the Service.

## Step 10 — Performance optimization checklist

- Prefer **Import mode** for this data volume; use **DirectQuery** or a **hybrid/composite
  model** only if source data is too large or must be near-real-time.
- Remove unused columns in Power Query before load (each column costs memory).
- Avoid calculated columns where a measure will do — measures compute on demand, calculated
  columns bloat the model on every refresh.
- Keep visuals per page under ~10; split dense pages rather than cramming.
- Set default Top N filters on any matrix/table that could otherwise return thousands of rows.
- Use `VAR` inside DAX measures (as done throughout the library) to avoid repeated evaluation of
  the same expression.
- Check **Performance Analyzer** (View tab) after building each page; if a visual takes
  >1–2 seconds, check for unnecessary bidirectional relationships or unfiltered large tables.

## Deployment notes for real manufacturing data

- Replace the `Manufacturing_Sample_Dataset.xlsx` source step with your real connector: for SQL
  Server/PostgreSQL, connect directly to the production/MES database's fact/dimension views (ask
  your DBA to expose them as views matching this schema, or map columns in Power Query); for a
  REST API (e.g. an MES or SCADA historian's API), use **Get Data → Web** or **Blank Query →
  Web.Contents** with pagination handled in M.
- If your real machine-level data is high-frequency (seconds/minutes), pre-aggregate to shift or
  hourly grain before it reaches Power BI — a star schema at sensor-tick grain will not perform
  well in Import mode.
- Everything else (relationships, measures, pages, theme) transfers unchanged as long as table
  and column names match this guide, or you update the measures accordingly.
