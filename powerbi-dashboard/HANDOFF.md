# ClearChoice Doctor Center-Level Controllables Dashboard — Power BI Build Handoff

## 1. Project overview

Building a Power BI report (`.pbip` format) that replicates an HTML/CSS mockup dashboard for ClearChoice Dental Implant Centers. Purpose: executive dashboard tracking 6 "doctor-controlled" KPIs across ~105 centers, for three audiences (executive leadership, Doctor Support Specialists, doctor-owners).

- Repo: `hienkle511-create/test`, branch `claude/powerbi-dashboard-design-fdi0ep`
- PBIP project location in repo: `powerbi-dashboard/Test08142026.pbip` + `powerbi-dashboard/Test08142026.Report/`
- Report page folder: `Test08142026.Report/definition/pages/7ae1ba30c1349c6800d1/`
- Report-level measures file: `Test08142026.Report/definition/reportExtensions.json`
- Custom theme file: `Test08142026.Report/StaticResources/RegisteredResources/ClearChoiceTheme.json`

## 2. Critical workflow constraint

**The Claude session building this has NO direct access to Power BI Desktop.** No screen control, no live connection, no MCP that actually works for this. All DAX/config must be:
1. Given as text by Claude
2. Manually pasted into Desktop's "New measure" formula bar (or the HTML Content visual's Values field) by the user
3. Tested by the user in Desktop
4. Results reported back to Claude in chat

This loop is slow but is the only path. Any future session must operate this way unless a working PBI connector/screen-control tool becomes available.

## 3. Semantic model (live connection, no local model)

The report is a **live connection** (`byConnection` in `definition.pbir`) to a published semantic model:
- Connection string points to workspace "Sales Metrics", dataset "Performance Management"
- **No edit access to the model itself** — cannot add real model-level measures, only "report-level measures" (a lighter Power BI feature, stored in `reportExtensions.json`, works fine for most DAX but does NOT support composite-model-level table additions)
- User does NOT have permission for "Add a local model" (composite model conversion) — confirmed via testing

### Known tables
- **Center** — dimension. Columns: `Center` (display name), `Region`, `Division`, `id` (used as the short code, e.g. "#7106"), `workday_center_id`, `old_center_name`, `center_active_c`, `center_open_date_c`, `rookie_center`. Division values: "Central-West", "Northern", "Midwest-South".
- **Calendar** — date table. Columns: `actualdate` (Date), `month_start` (Date, first-of-month), `MonthYear`, plus time-intelligence measures (MTD, QTD, YTD, Last 1M/3M/12M, etc.)
- **Calculations** — the measures table (folders: "Operations", "KPI", "Top 20% Tier")

### Key remote (pre-existing) measures
- **`Center Ranking Overall`** — the true name of the overall ranking measure. **IMPORTANT: earlier in this project this was misidentified as `Overall Ranking` — that name is WRONG and caused hours of `Missing_References` debugging.** Always use `Center Ranking Overall`. Lower number = better rank. This measure IS already date-range-dynamic (recalculates per the Calendar filter context) — confirmed by the user, no need to build ranking logic ourselves.
- Existing `Tier 20% <Metric>` measures (pre-built, average of the metric among centers in the top-20%-by-`Center Ranking Overall` group):
  - `Tier 20% Days to Final Median`
  - `Tier 20% Arches Cut per Surgery Day Center` (note the "Center" suffix — inconsistent naming, this one is different from the base metric name)
  - `Tier 20% % of Days Open`
  - `Tier 20% Days Open Baseline Inventory`
  - **No existing Tier 20% measure** for Days to Surgery or SRPC — these two had to be built (see below).
  - **No "Bottom 20%" family exists at all** in the model — all 6 had to be built.

### The 6 KPI metrics — base measure names, direction, format

| Card label | Base measure | Direction | Format |
|---|---|---|---|
| Days to Final | `Days to Final Median` | lower is better | whole number + "days" |
| Days to Surgery | `Prosth Exam to Surgery Days Median` | lower is better | whole number + "days" |
| Arches Cut / Surgery Day | `Arches Cut per Surgery Day` | higher is better | 2 decimals |
| % of Days Open | `% of Days Open` | higher is better | **is a 0-1 fraction** — use `"0%"` format string (auto ×100) — see gotcha below |
| Days Open | `Days Open Baseline Inventory` | higher is better | whole number + "days" — **see SUM gotcha below** |
| SRPC (Sold Revenue per Consult) | `SRPC` | higher is better | currency, whole dollars, `"$" & FORMAT(x,"#,0")` |

### Data-quality gotchas discovered (real model behavior, not DAX bugs)

1. **`% of Days Open` returns different scales depending on filter context.** Per-center (e.g. inside `AVERAGEX`/`FILTER` over `ALL(Center)`) it's a clean 0-1 fraction (e.g. 0.90, 0.95 → correctly use `FORMAT(x,"0%")`). But the plain unfiltered/whole-network aggregate (`[% of Days Open]` called with no Center context) returns a **different, inflated scale** (e.g. raw ~107.4, displaying as 10740% with `"0%"` format) — this is a real inconsistency in the underlying measure, not something we can fix (no model edit access). **Fix used:** never call the raw measure for a "network average" — always wrap: `AVERAGEX(ALL(Center), [% of Days Open])`. This gives a sane per-center average.

2. **`Days Open Baseline Inventory` sums across centers when there's no Center filter**, since it's fundamentally a SUM-type measure (matches the original HTML mockup's `agg:'sum'` design for this metric — days-open was always meant to be summed per center across months, then averaged across centers for a "typical center" figure, never summed across centers). **Fix used:** same pattern — `AVERAGEX(ALL(Center), [Days Open Baseline Inventory])` for any "network average" context.

3. Both gotchas above needed the same 3-place fix per metric: the "Network" HTML card's main value, the "Top20"/"Bottom20" cards' `vNetwork` (used for the delta badge comparison), and the Sparkline's per-month `CALCULATE(...)` value. All fixed and confirmed working.

## 4. Report-level DAX measures already built (all confirmed working)

All defined via Desktop's Fields pane → Calculations table → right-click → New measure (NOT via file-edit — file-edited measures had reference-resolution problems; typed-in-Desktop measures work reliably).

### A. Percentile threshold measures (12 total — 2 per metric)

Pattern: `PERCENTILEX.INC(FILTER(ALL(Center), NOT ISBLANK([BaseMeasure])), [BaseMeasure], k)` where k=0.2 for the "lower is better" direction's Top 20% (and "higher is better"'s Bottom 20%), k=0.8 for the reverse. These represent **"the value threshold a center needs to hit/avoid to be in the top/bottom 20% for THIS metric specifically"** (a deliberate redesign from the original HTML mockup's group-average corner chips — confirmed with user).

```dax
Top 20% Threshold Days to Final Median =
PERCENTILEX.INC ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Days to Final Median] ) ), [Days to Final Median], 0.2 )

Bottom 20% Threshold Days to Final Median =
PERCENTILEX.INC ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Days to Final Median] ) ), [Days to Final Median], 0.8 )

Top 20% Threshold Prosth Exam to Surgery Days Median =
PERCENTILEX.INC ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Prosth Exam to Surgery Days Median] ) ), [Prosth Exam to Surgery Days Median], 0.2 )

Bottom 20% Threshold Prosth Exam to Surgery Days Median =
PERCENTILEX.INC ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Prosth Exam to Surgery Days Median] ) ), [Prosth Exam to Surgery Days Median], 0.8 )

Top 20% Threshold Arches Cut per Surgery Day =
PERCENTILEX.INC ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Arches Cut per Surgery Day] ) ), [Arches Cut per Surgery Day], 0.8 )

Bottom 20% Threshold Arches Cut per Surgery Day =
PERCENTILEX.INC ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Arches Cut per Surgery Day] ) ), [Arches Cut per Surgery Day], 0.2 )

Top 20% Threshold % of Days Open =
PERCENTILEX.INC ( FILTER ( ALL ( Center ), NOT ISBLANK ( [% of Days Open] ) ), [% of Days Open], 0.8 )

Bottom 20% Threshold % of Days Open =
PERCENTILEX.INC ( FILTER ( ALL ( Center ), NOT ISBLANK ( [% of Days Open] ) ), [% of Days Open], 0.2 )

Top 20% Threshold Days Open Baseline Inventory =
PERCENTILEX.INC ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Days Open Baseline Inventory] ) ), [Days Open Baseline Inventory], 0.8 )

Bottom 20% Threshold Days Open Baseline Inventory =
PERCENTILEX.INC ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Days Open Baseline Inventory] ) ), [Days Open Baseline Inventory], 0.2 )

Top 20% Threshold SRPC =
PERCENTILEX.INC ( FILTER ( ALL ( Center ), NOT ISBLANK ( [SRPC] ) ), [SRPC], 0.8 )

Bottom 20% Threshold SRPC =
PERCENTILEX.INC ( FILTER ( ALL ( Center ), NOT ISBLANK ( [SRPC] ) ), [SRPC], 0.2 )
```

### B. Group-average measures (8 total — for mode-switching big values, NOT the same as the thresholds above)

These average the metric across the Overall-Ranking-based top/bottom 21 centers (105 centers × 20% ≈ 21). Used for the Top20%/Bottom20% mode cards' big value (distinct purpose from the per-metric percentile thresholds in section A).

```dax
Bottom 20% Days to Final Median =
VAR TotalCenters = COUNTROWS ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Center Ranking Overall] ) ) )
VAR GroupSize = ROUNDUP ( TotalCenters * 0.2, 0 )
VAR BottomCenters = TOPN ( GroupSize, ALL ( Center ), [Center Ranking Overall], DESC )
RETURN AVERAGEX ( BottomCenters, [Days to Final Median] )

Bottom 20% Prosth Exam to Surgery Days Median =
VAR TotalCenters = COUNTROWS ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Center Ranking Overall] ) ) )
VAR GroupSize = ROUNDUP ( TotalCenters * 0.2, 0 )
VAR BottomCenters = TOPN ( GroupSize, ALL ( Center ), [Center Ranking Overall], DESC )
RETURN AVERAGEX ( BottomCenters, [Prosth Exam to Surgery Days Median] )

Bottom 20% Arches Cut per Surgery Day =
VAR TotalCenters = COUNTROWS ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Center Ranking Overall] ) ) )
VAR GroupSize = ROUNDUP ( TotalCenters * 0.2, 0 )
VAR BottomCenters = TOPN ( GroupSize, ALL ( Center ), [Center Ranking Overall], DESC )
RETURN AVERAGEX ( BottomCenters, [Arches Cut per Surgery Day] )

Bottom 20% % of Days Open =
VAR TotalCenters = COUNTROWS ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Center Ranking Overall] ) ) )
VAR GroupSize = ROUNDUP ( TotalCenters * 0.2, 0 )
VAR BottomCenters = TOPN ( GroupSize, ALL ( Center ), [Center Ranking Overall], DESC )
RETURN AVERAGEX ( BottomCenters, [% of Days Open] )

Bottom 20% Days Open Baseline Inventory =
VAR TotalCenters = COUNTROWS ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Center Ranking Overall] ) ) )
VAR GroupSize = ROUNDUP ( TotalCenters * 0.2, 0 )
VAR BottomCenters = TOPN ( GroupSize, ALL ( Center ), [Center Ranking Overall], DESC )
RETURN AVERAGEX ( BottomCenters, [Days Open Baseline Inventory] )

Bottom 20% SRPC =
VAR TotalCenters = COUNTROWS ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Center Ranking Overall] ) ) )
VAR GroupSize = ROUNDUP ( TotalCenters * 0.2, 0 )
VAR BottomCenters = TOPN ( GroupSize, ALL ( Center ), [Center Ranking Overall], DESC )
RETURN AVERAGEX ( BottomCenters, [SRPC] )

Tier 20% Prosth Exam to Surgery Days Median =
VAR TotalCenters = COUNTROWS ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Center Ranking Overall] ) ) )
VAR GroupSize = ROUNDUP ( TotalCenters * 0.2, 0 )
VAR TopCenters = TOPN ( GroupSize, ALL ( Center ), [Center Ranking Overall], ASC )
RETURN AVERAGEX ( TopCenters, [Prosth Exam to Surgery Days Median] )

Tier 20% SRPC =
VAR TotalCenters = COUNTROWS ( FILTER ( ALL ( Center ), NOT ISBLANK ( [Center Ranking Overall] ) ) )
VAR GroupSize = ROUNDUP ( TotalCenters * 0.2, 0 )
VAR TopCenters = TOPN ( GroupSize, ALL ( Center ), [Center Ranking Overall], ASC )
RETURN AVERAGEX ( TopCenters, [SRPC] )
```

(Naming note: the 2 new "best" measures use the `Tier 20%` prefix to match the existing family; the 6 new "worst" measures use `Bottom 20%` since that's an entirely new family — this was a deliberate, confirmed naming choice, not an inconsistency.)

## 5. HTML Content visual technique (the big pivot)

Discovered partway through the build: a free AppSource custom visual called **"HTML Content" by Daniel Marsh-Patrick** (OKViz) lets you drop **one measure into its "Values" field**, and whatever text that measure returns is rendered directly as live HTML/CSS/SVG. This let us build KPI cards that far more closely match the original mockup's styling than native Power BI visuals ever could (custom fonts, precise spacing, inline SVG sparklines with dashed threshold lines), while still being fully data-bound and reactive to filters/slicers.

**This replaced the original plan of native Card/textbox/lineChart visuals for the KPI row.** The header row (title — kept native in the end, date slicer, buttons, center dropdown) stayed as native Power BI visuals since HTML Content is **display-only** — it cannot filter data or trigger navigation, so anything interactive must stay native.

### Hard-won gotchas for writing DAX-generated HTML

1. **Google Fonts DO load** inside the HTML Content visual via a `<style>@import url(...)</style>` block — no tenant restriction hit in this case. Used Poppins (body/labels) + IBM Plex Mono (numeric/monospace), matching the original mockup's fonts exactly.

2. **CRITICAL quote-escaping bug (cost hours of debugging):** when the outer HTML attribute uses single quotes (`style='...'`), do **NOT** put single-quoted CSS values inside it (e.g. `font-family:'IBM Plex Mono', monospace`). The browser's HTML parser treats the first inner single-quote as the end of the whole `style` attribute, silently truncating everything after it (font-size, font-weight, color — all dropped). **Fix: never quote multi-word font names in inline styles — `font-family:IBM Plex Mono, monospace` (no quotes) works fine, CSS allows unquoted multi-word font names.** This was NOT a DAX escaping issue (DAX doesn't need `''` for single quotes — that instinct was wrong, DAX only doubles `""` for literal double-quotes inside a string).

3. Auto-fit/scaling: font-size values in the generated HTML are true CSS pixels, not auto-scaled by the visual — if something looks "too big/small," check the actual visual's box dimensions on the canvas before assuming a CSS bug.

4. The visual seemingly has no character/complexity limit that caused problems even for the ~150-line sparkline SVG-generating measures.

### Card architecture per metric (established pattern, 4 measures each)

For each of the 6 metrics, four separate report-level measures, each bound to its own HTML Content visual instance:
- **`Card HTML <Metric> Network`** — label, corner chips (Top20/Bottom20 percentile thresholds), big value (plain/network-averaged), "This is the network average" caption
- **`Card HTML <Metric> Top20`** — same layout, big value = Tier20%-group average, plus a colored delta badge (▲/▼/→, green/red/gray) comparing to network average, direction-aware (lower-is-better metrics: green if below average; higher-is-better: green if above)
- **`Card HTML <Metric> Bottom20`** — same as Top20 but using the Bottom20%-group average
- **`Card HTML <Metric> Sparkline`** — inline SVG: monthly trend line + dots + alternating above/below value labels + month-initial x-axis labels + **two dashed threshold reference lines** (green = Top20% threshold, red = Bottom20% threshold) that **vary month-to-month** (computed via `SUMMARIZE(Calendar, Calendar[month_start])` + per-month `CALCULATE(...)` of the threshold measures, not a flat line) — this was an explicit design refinement requested mid-build.

### Full DAX for all 24 Card HTML measures (all confirmed working)

**Days to Final** (lower is better):
```dax
Card HTML Days to Final Network =
VAR vValue = [Days to Final Median]
VAR vTop20 = [Top 20% Threshold Days to Final Median]
VAR vBottom20 = [Bottom 20% Threshold Days to Final Median]
RETURN
"<style>@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500;600;700&display=swap');</style>" &
"<div style='font-family:Poppins, Segoe UI, sans-serif; padding:2px; position:relative;'>" &
  "<div style='display:flex; justify-content:space-between; align-items:flex-start;'>" &
    "<div style='font-size:10.5px; font-weight:600; color:#52697A;'>Days to Final</div>" &
    "<div style='display:flex; flex-direction:column; align-items:flex-end; font-family:IBM Plex Mono, monospace; font-size:7.5px; color:#8398A6; line-height:1.15; white-space:nowrap;'>" &
      "<span>Top20 <b style='color:#52697A;'>" & FORMAT(vTop20, "0") & "</b></span>" &
      "<span>Bot20 <b style='color:#52697A;'>" & FORMAT(vBottom20, "0") & "</b></span>" &
    "</div>" &
  "</div>" &
  "<div style='font-family:IBM Plex Mono, monospace; font-size:19px; font-weight:700; color:#0B2A43; margin-top:1px;'>" &
    FORMAT(vValue, "0") &
    " <span style='font-size:11px; color:#8398A6; font-weight:500;'>days</span>" &
  "</div>" &
  "<div style='font-size:9.5px; color:#8398A6; font-weight:600; margin-top:2px;'>This is the network average</div>" &
"</div>"

Card HTML Days to Final Top20 =
VAR vValue = [Tier 20% Days to Final Median]
VAR vNetwork = [Days to Final Median]
VAR vTop20 = [Top 20% Threshold Days to Final Median]
VAR vBottom20 = [Bottom 20% Threshold Days to Final Median]
VAR vDiff = vValue - vNetwork
VAR vBetter = vDiff < 0
VAR vIsZero = ROUND(vDiff, 0) = 0
VAR vBadgeColor = IF(vIsZero, "#8398A6", IF(vBetter, "#2E8B57", "#C0392B"))
VAR vBadgeBg = IF(vIsZero, "#E7ECEF", IF(vBetter, "#E7F4EC", "#FBEAE7"))
VAR vArrow = IF(vIsZero, "→", IF(vDiff > 0, "▲", "▼"))
VAR vDiffText = FORMAT(ABS(vDiff), "0")
RETURN
"<style>@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500;600;700&display=swap');</style>" &
"<div style='font-family:Poppins, Segoe UI, sans-serif; padding:2px; position:relative;'>" &
  "<div style='display:flex; justify-content:space-between; align-items:flex-start;'>" &
    "<div style='font-size:10.5px; font-weight:600; color:#52697A;'>Days to Final</div>" &
    "<div style='display:flex; flex-direction:column; align-items:flex-end; font-family:IBM Plex Mono, monospace; font-size:7.5px; color:#8398A6; line-height:1.15; white-space:nowrap;'>" &
      "<span>Top20 <b style='color:#52697A;'>" & FORMAT(vTop20, "0") & "</b></span>" &
      "<span>Bot20 <b style='color:#52697A;'>" & FORMAT(vBottom20, "0") & "</b></span>" &
    "</div>" &
  "</div>" &
  "<div style='font-family:IBM Plex Mono, monospace; font-size:19px; font-weight:700; color:#0B2A43; margin-top:1px;'>" &
    FORMAT(vValue, "0") &
    " <span style='font-size:11px; color:#8398A6; font-weight:500;'>days</span>" &
  "</div>" &
  "<div style='display:flex; align-items:center; gap:6px; margin-top:2px;'>" &
    "<span style='font-family:IBM Plex Mono, monospace; font-size:12px; font-weight:700; padding:1.5px 7px; border-radius:6px; background:" & vBadgeBg & "; color:" & vBadgeColor & ";'>" & vArrow & " " & vDiffText & "</span>" &
    "<span style='font-size:9.5px; color:#8398A6; font-weight:600;'>vs Network Avg<br><b style='color:#52697A;'>" & FORMAT(vNetwork, "0") & "</b></span>" &
  "</div>" &
"</div>"

Card HTML Days to Final Bottom20 =
VAR vValue = [Bottom 20% Days to Final Median]
VAR vNetwork = [Days to Final Median]
VAR vTop20 = [Top 20% Threshold Days to Final Median]
VAR vBottom20 = [Bottom 20% Threshold Days to Final Median]
VAR vDiff = vValue - vNetwork
VAR vBetter = vDiff < 0
VAR vIsZero = ROUND(vDiff, 0) = 0
VAR vBadgeColor = IF(vIsZero, "#8398A6", IF(vBetter, "#2E8B57", "#C0392B"))
VAR vBadgeBg = IF(vIsZero, "#E7ECEF", IF(vBetter, "#E7F4EC", "#FBEAE7"))
VAR vArrow = IF(vIsZero, "→", IF(vDiff > 0, "▲", "▼"))
VAR vDiffText = FORMAT(ABS(vDiff), "0")
RETURN
"<style>@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500;600;700&display=swap');</style>" &
"<div style='font-family:Poppins, Segoe UI, sans-serif; padding:2px; position:relative;'>" &
  "<div style='display:flex; justify-content:space-between; align-items:flex-start;'>" &
    "<div style='font-size:10.5px; font-weight:600; color:#52697A;'>Days to Final</div>" &
    "<div style='display:flex; flex-direction:column; align-items:flex-end; font-family:IBM Plex Mono, monospace; font-size:7.5px; color:#8398A6; line-height:1.15; white-space:nowrap;'>" &
      "<span>Top20 <b style='color:#52697A;'>" & FORMAT(vTop20, "0") & "</b></span>" &
      "<span>Bot20 <b style='color:#52697A;'>" & FORMAT(vBottom20, "0") & "</b></span>" &
    "</div>" &
  "</div>" &
  "<div style='font-family:IBM Plex Mono, monospace; font-size:19px; font-weight:700; color:#0B2A43; margin-top:1px;'>" &
    FORMAT(vValue, "0") &
    " <span style='font-size:11px; color:#8398A6; font-weight:500;'>days</span>" &
  "</div>" &
  "<div style='display:flex; align-items:center; gap:6px; margin-top:2px;'>" &
    "<span style='font-family:IBM Plex Mono, monospace; font-size:12px; font-weight:700; padding:1.5px 7px; border-radius:6px; background:" & vBadgeBg & "; color:" & vBadgeColor & ";'>" & vArrow & " " & vDiffText & "</span>" &
    "<span style='font-size:9.5px; color:#8398A6; font-weight:600;'>vs Network Avg<br><b style='color:#52697A;'>" & FORMAT(vNetwork, "0") & "</b></span>" &
  "</div>" &
"</div>"

Card HTML Days to Final Sparkline =
VAR ChartW = 195
VAR ChartH = 60
VAR PadL = 8
VAR PadR = 8
VAR PadT = 10
VAR PadB = 14
VAR InnerW = ChartW - PadL - PadR
VAR InnerH = ChartH - PadT - PadB
VAR MonthlyRaw =
    FILTER(
        ADDCOLUMNS(
            SUMMARIZE(Calendar, Calendar[month_start]),
            "MonthValue", CALCULATE([Days to Final Median]),
            "MonthTop20", CALCULATE([Top 20% Threshold Days to Final Median]),
            "MonthBottom20", CALCULATE([Bottom 20% Threshold Days to Final Median])
        ),
        NOT ISBLANK([MonthValue])
    )
VAR NumMonths = COUNTROWS(MonthlyRaw)
VAR vMinRaw = MINX(MonthlyRaw, MIN([MonthValue], MIN([MonthTop20], [MonthBottom20])))
VAR vMaxRaw = MAXX(MonthlyRaw, MAX([MonthValue], MAX([MonthTop20], [MonthBottom20])))
VAR vRange = vMaxRaw - vMinRaw
VAR vPad = IF(vRange = 0, 1, vRange * 0.15)
VAR vMin = vMinRaw - vPad
VAR vMax = vMaxRaw + vPad
VAR MonthlyIndexed =
    ADDCOLUMNS(MonthlyRaw, "MonthIndex", RANKX(MonthlyRaw, [month_start], , ASC, Dense) - 1)
VAR MonthlyPositioned =
    ADDCOLUMNS(
        MonthlyIndexed,
        "PtX", PadL + DIVIDE([MonthIndex], MAX(NumMonths - 1, 1)) * InnerW,
        "PtY", PadT + InnerH - DIVIDE([MonthValue] - vMin, vMax - vMin) * InnerH,
        "PtTop20Y", PadT + InnerH - DIVIDE([MonthTop20] - vMin, vMax - vMin) * InnerH,
        "PtBottom20Y", PadT + InnerH - DIVIDE([MonthBottom20] - vMin, vMax - vMin) * InnerH,
        "PtLabel", LEFT(FORMAT([month_start], "MMM"), 1)
    )
VAR MonthlyLabeled =
    ADDCOLUMNS(MonthlyPositioned, "LabelYRaw", IF(MOD([MonthIndex], 2) = 0, [PtY] - 5, [PtY] + 11))
VAR MonthlyLabeled2 =
    ADDCOLUMNS(MonthlyLabeled, "LabelY", IF([LabelYRaw] < PadT + 6, [PtY] + 11, IF([LabelYRaw] > ChartH - 4, [PtY] - 5, [LabelYRaw])))
VAR PolylinePoints = CONCATENATEX(MonthlyLabeled2, FORMAT([PtX], "0.0") & "," & FORMAT([PtY], "0.0"), " ", [MonthIndex], ASC)
VAR Top20Points = CONCATENATEX(MonthlyLabeled2, FORMAT([PtX], "0.0") & "," & FORMAT([PtTop20Y], "0.0"), " ", [MonthIndex], ASC)
VAR Bottom20Points = CONCATENATEX(MonthlyLabeled2, FORMAT([PtX], "0.0") & "," & FORMAT([PtBottom20Y], "0.0"), " ", [MonthIndex], ASC)
VAR Dots = CONCATENATEX(MonthlyLabeled2, "<circle cx='" & FORMAT([PtX],"0.0") & "' cy='" & FORMAT([PtY],"0.0") & "' r='2.4' fill='#00A0B0'/>", "", [MonthIndex], ASC)
VAR ValueLabels = CONCATENATEX(MonthlyLabeled2, "<text x='" & FORMAT([PtX],"0.0") & "' y='" & FORMAT([LabelY],"0.0") & "' text-anchor='middle' font-family='IBM Plex Mono, monospace' font-size='7' font-weight='600' fill='#0B2A43'>" & FORMAT([MonthValue],"0") & "</text>", "", [MonthIndex], ASC)
VAR XLabels = CONCATENATEX(MonthlyLabeled2, "<text x='" & FORMAT([PtX],"0.0") & "' y='" & FORMAT(ChartH-2,"0") & "' text-anchor='middle' font-family='IBM Plex Mono, monospace' font-size='9' fill='#8398A6'>" & [PtLabel] & "</text>", "", [MonthIndex], ASC)
RETURN
"<svg width='100%' height='100%' viewBox='0 0 " & ChartW & " " & ChartH & "' preserveAspectRatio='xMidYMid meet' xmlns='http://www.w3.org/2000/svg'>" &
  "<polyline points='" & Top20Points & "' fill='none' stroke='#2E8B57' stroke-width='1' stroke-dasharray='3 2'/>" &
  "<polyline points='" & Bottom20Points & "' fill='none' stroke='#C0392B' stroke-width='1' stroke-dasharray='3 2'/>" &
  "<polyline points='" & PolylinePoints & "' fill='none' stroke='#00A0B0' stroke-width='1.8' stroke-linecap='round' stroke-linejoin='round'/>" &
  Dots & ValueLabels & XLabels &
"</svg>"
```

**Days to Surgery** (lower is better) — identical structure to Days to Final, substitute `[Prosth Exam to Surgery Days Median]` everywhere and label "Days to Surgery". (Full text was given in-chat; regenerate by substitution if not preserved verbatim — pattern is 100% identical to Days to Final above.)

**Arches Cut per Surgery Day** (higher is better) — same structure, `vBetter = vDiff > 0` (flipped), format `"0.00"` (2 decimals), no unit suffix, label "Arches Cut / Surgery Day". Top20 group measure uses `[Tier 20% Arches Cut per Surgery Day Center]` (note "Center" suffix — existing model quirk).

**% of Days Open** (higher is better) — format `"0%"` **only for the threshold/group-average measures** (genuine fractions); for the plain network value use `AVERAGEX(ALL(Center), [% of Days Open])` wrapped, still formatted `"0%"`. `vBetter = vDiff > 0`. Label "% of Days Open".

**Days Open** (higher is better) — plain network value must be `AVERAGEX(ALL(Center), [Days Open Baseline Inventory])` (see gotcha #2 above), format whole number + "days". `vBetter = vDiff > 0`. Label "Days Open". Top20 group uses `[Tier 20% Days Open Baseline Inventory]` (exact match, no suffix quirk).

**SRPC** (higher is better) — currency format `"$" & FORMAT(x,"#,0")` everywhere a value/threshold is shown. `vBetter = vDiff > 0`. Label "SRPC" (kept short, not spelled out, per user's final call for the card label — though SRPC = "Sold Revenue per Consult" if it needs explaining elsewhere).

All 24 measures follow this exact template; if any got lost, they can be regenerated by substituting the metric's base measure, Top20/Bottom20 group measures, threshold measures, label text, and format string per the table in section 3.

## 6. Native visuals already built (header row)

- Custom report theme (`ClearChoiceTheme.json`) — colors matching mockup CSS variables: `--ink:#0B2A43`, `--teal:#00A0B0`, `--good:#2E8B57`, `--warn:#C4772A`, `--alert:#C0392B`, `--muted:#52697A`, `--muted-2:#8398A6`, `--paper:#F5F7F8`, `--line:#D7DEE2`. Default Power BI fonts (Segoe UI/Consolas) for native visuals — no custom font support without tenant-level Google Fonts enablement (this constraint does NOT apply to HTML Content visuals, which load fonts fine via CSS `@import`).
- Title textbox: "Doctor Center-Level Controllables Dashboard" (kept native, not converted to HTML)
- Logo placeholder textbox top-right (still waiting on actual ClearChoice logo file)
- Date range slicer, bound to `Calendar[actualdate]`
- 3 mode buttons (Top 20%, Bottom 20%, "Network Average" — user intentionally renamed from "Network Overview"), 3 division buttons (Central-West, Northern, Midwest-South) — styled as pills (rounded corners ~16-20, white fill, `#D7DEE2` border, `#52697A` text ~11.5px)
- Center search slicer, bound to `Center[Center]` — **gotcha:** categorical slicers need `"active": true` on the query projection, `"showAll": true`, and a `filterConfig.filters` TopN-style block to actually populate selectable values — a plain projection-only binding (which worked fine for the Date slicer) silently shows nothing for a text/category field.
- 5 context-strip textboxes (Network/Top20/Bottom20/Division/Center variants) — built but likely **obsolete under the new multi-page plan** (each becomes per-page static or removed).

### PBIR / visual-authoring gotchas (for any future native-visual work)
- `actionButton` visuals: text only renders if there's a **second properties block scoped to `"selector": {"id": "default"}`**, not just the base unscoped entry — confirmed by reading Desktop's own saved output after the user manually fixed it in the UI.
- Report-level measures added via direct JSON file-edit to `reportExtensions.json` had a `references` metadata block that, if incomplete, caused `Missing_References` errors when the measure was actually used in a visual (even though the measure "existed" and showed no error in the Fields list) — but this ended up being a red herring; the REAL root cause of the persistent errors was the wrong field name (`Overall Ranking` vs `Center Ranking Overall`), not the references metadata. Measures typed directly into Desktop's formula bar do not have this problem — **prefer that path for any new measures going forward.**

## 7. NEW plan: multi-page architecture (agreed, not yet built)

Trigger: user wants each audience to be able to **subscribe** to just their relevant Power BI page (e.g., a division manager subscribes to just their division's page) — this requires splitting what was a single bookmark-driven page into **6 separate pages**.

### Pages
1. **Network Overview** (merged with "Center Detail" — user realized these are the same lens) — Network-variant HTML cards (already built) + Center search slicer (**local to this page only**, not synced globally) + a full table of all ~105 centers with all 6 metrics (sortable). Selecting a center in the slicer should drill the cards down to that one center; clearing it reverts to the network average. **Open TODO:** for the 2 metrics needing the `AVERAGEX(ALL(Center), ...)` fix (% of Days Open, Days Open), that fix makes them IGNORE the Center slicer entirely (since `ALL(Center)` clears any Center-level filter) — need a conditional version for this page only: `IF(ISFILTERED(Center), [BaseMeasure], AVERAGEX(ALL(Center), [BaseMeasure]))` so it correctly shows the single selected center's raw value when filtered, and the safe network average otherwise. **Not yet built — needs new measures for just these 2 metrics' Network-page variants.**
2. **Top 20% Overview** — Top20-variant HTML cards (already built) + a table of the top 21 centers by `Center Ranking Overall` (mirror of the Bottom 20% table, not yet built)
3. **Bottom 20% Overview** — Bottom20-variant HTML cards (already built) + the Bottom 20% table (original task #7 from the single-page plan — not yet built)
4. **Central-West**, **Northern**, **Midwest-South** — each page reuses the **Network-variant** HTML cards (no new measures needed — a division is just a filtered slice), with a **page-level filter** locking `Center[Division]` to that division, plus a table of that division's centers

### Interaction model changes from the original single-page plan
- Mode/division buttons become simple **page-navigation buttons** (Power BI native "Page navigation" action) — **bookmarks may not be needed at all** for mode-switching anymore (this was the whole point of the original task #8, now likely unnecessary or much simpler).
- Center slicer: **local to Network Overview page only** — do not sync it globally, since drilling into one center doesn't make sense on the fixed-group pages (Top20/Bottom20/Division).
- Date range slicer: **should be synced/shared across all 6 pages** (Power BI's native "Sync slicers" pane) so switching pages doesn't reset the selected time window. Not yet configured.
- Header visuals (title, date slicer, buttons, logo) need to be **copy-pasted onto all 6 pages** — Power BI has no persistent/shared header across pages.

## 8. Remaining work (as of handoff)

1. Restructure into 6 pages per the plan above (currently everything is stacked on one page, `7ae1ba30c1349c6800d1`)
2. Build the conditional `IF(ISFILTERED(Center), ..., AVERAGEX(ALL(Center), ...))` measures for % of Days Open and Days Open on the Network page specifically
3. Build/duplicate the header row visuals onto all 6 pages
4. Set up page-level Division filters on the 3 division pages
5. Convert mode/division buttons to page-navigation actions
6. Sync the date range slicer across all pages
7. Build 3 tables: full centers list (Network page), top-21 table (Top20 page), bottom-21 table (Bottom20 page) — plus 3 division-scoped tables
8. Add the ClearChoice logo once the file is available (still a placeholder)
9. Decide whether the old single-page 3-stacked-card-variant approach and the 5 context-strip textboxes should be deleted now that pages replace that mechanism

## 9. Git state

All work through the 6-metric KPI card completion is committed and pushed to `claude/powerbi-dashboard-design-fdi0ep` on `hienkle511-create/test`. However, **most of the "Card HTML" measures and threshold/group measures from sections 4-5 were typed directly into Desktop by the user, not file-edited by Claude** — so the actual live `reportExtensions.json` in the user's local Desktop copy is likely ahead of what's committed to git. **Before continuing work, sync the user's latest Desktop state back into the repo** (same process used throughout this build: user zips their project folder, Claude extracts and diffs against the repo, commits the delta).
