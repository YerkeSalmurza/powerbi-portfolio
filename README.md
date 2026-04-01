# 📊 KPI Sales Tracker Dashboard

## 📋 Business Objective
Track sales plan execution across regions and months, and forecast the final year-end performance based on actual results and working days passed.

---

## 📸 Dashboard Pages

Navigation menu allows users to easily switch between analytical views:

![Navigation](screenshots/Navigation.png)

### 1. Plan vs Actual
Comparison of planned vs actual KPI values by region (top chart) and by month (bottom chart).
Tooltips enabled. Edit interactions activated — visuals dynamically respond to slicer selections.
Interactive filters: Month, Region.

![Plan vs Actual](screenshots/01_plan-vs-actual.png)

### 2. Rate
Sales ranking by region with color indicators — top 3 regions highlighted in blue, bottom 2 in red.
Tooltips enabled. Responsive to Month and Region filters.

![Rate](screenshots/02_rate.png)

### 3. Forecast
Forecasted year-end KPI values with bottom 3 underperforming regions highlighted.
Forecast logic based on working days passed vs remaining working days.
Tooltips enabled.

![Forecast](screenshots/03_forecast.png)

### 4. Pivot
Detailed KPI pivot table — pure data view with plan, actual, % completion, and remaining by month and by region.
Supports full filtering by Month and Region. Edit interactions enabled.

![Pivot](screenshots/04_pivot.png)

---

## 🗂️ Data Model

### Power Query Structure
All data sources are in Excel format — this is a sample dataset.

![Power Query](screenshots/PowerQuery.png)

**Dimension tables:**
- `holiday_calendar` — excludes public holidays and weekends from working day calculations
- `DimRegion` — links plan and actual data, used in filters

**Fact tables:**
- `Plan` — planned values for 2025 by region and date
- `Sales` — actual sales data for 2025

### Relationships
![Relationships Model View](screenshots/Relation_1.png)
![Manage Relationships](screenshots/Relation_2.png)

| From | Relationship | To |
|------|-------------|-----|
| holiday_calendar (Date) | 1—1 | Calendar (Date) |
| Plan (Date) | *—1 | Calendar (Date) |
| Plan (Region) | *—1 | DimRegion (Region) |
| Sales (DOG_FROMDATE) | *—1 | Calendar (Date) |
| Sales (FILIAL) | *—1 | DimRegion (Region) |

---

## ⚙️ Power Query Transformations

Only the `Plan` table was transformed using **Unpivot Other Columns** to convert monthly columns into rows with a unified "Date" and "Plan" structure.

![Transform](screenshots/Transform.png)

Additionally, column data types were set to appropriate formats:
- **Date** columns → date format
- **Plan** values → numeric format

---

## 📐 DAX: Working with Data in Table View

A **Calendar** table was created, taking minimum and maximum dates from the Sales table:
```dax
Calendar = ADDCOLUMNS(
    CALENDAR(
        DATE(YEAR(MIN(MIN(Sales[DOG_FROMDATE]), MIN(Plan[Date]))), 1, 1),
        DATE(YEAR(MAX(MAX(Sales[DOG_FROMDATE]), MAX(Plan[Date]))), 12, 31)
    ),
    "Year", YEAR([Date]),
    "Month", FORMAT([Date], "mmm", "ru-ru"),
    "MonthNumber", MONTH([Date]),
    "Period", FORMAT([Date], "mmmyy", "ru-ru"),
    "WeekDayNum", WEEKDAY([Date], 2),
    "WeekDay", IF(WEEKDAY([Date],2)=1,"Mon",IF(WEEKDAY([Date],2)=2,"Tue",
               IF(WEEKDAY([Date],2)=3,"Wed",IF(WEEKDAY([Date],2)=4,"Thu",
               IF(WEEKDAY([Date],2)=5,"Fri",IF(WEEKDAY([Date],2)=6,"Sat","Sun")))))),
    "Quarter", "кв. " & FORMAT([Date], "q"),
    "PeriodOrder", FORMAT([Date], "YYYYMM")
)
```

### Calculated Columns

**DateWithData** — returns TRUE if the date ≤ max available date. Used as a page-level filter to exclude future dates:
```dax
DateWithData = 'Calendar'[Date] <= MAX(Sales[DOG_FROMDATE])
```

![Filter](screenshots/Filter.png)

**DayType** — brings the day type (Workday, Weekend, Holiday) from the holiday calendar:
```dax
DayType = RELATED(holiday_calendar[Day Type])
```

### Virtual Table for Ranking

An emoji table was created to color-code ranks by region:
```dax
EmojiTable = DATATABLE(
    "Rank", INTEGER, "Emoji", STRING,
    {
        {1,"😊"},{2,"😊"},{3,"😊"},
        {4,"😐"},{5,"😐"},{6,"😐"},{7,"😐"},{8,"😐"},
        {9,"😐"},{10,"😐"},{11,"😐"},{12,"😐"},
        {13,"😐"},{14,"😐"},{15,"😐"},
        {16,"😨"},{17,"😨"},{18,"😨"}
    }
)
```

---

## 📐 Measures

### 1. Main KPI Measures
```dax
Plan = SUM(Plan[Plan])

Sales = SUM(Sales[Sales])

Plan Execution = [Sales] - [Plan]

Plan Execution, % = DIVIDE([Sales], [Plan], 0)

Sales Forecast, % = IF([Work Days Passed] = 0, BLANK(),
                    [Sales] / [Work Days Passed] * ([Work Days Count] / [Plan]))

Sales Forecast = IF([Sales Forecast, %] = BLANK(), BLANK(),
                 [Plan] * [Sales Forecast, %])
```

### 2. Calendar-Based Measures
```dax
Work Days Count = SUM(holiday_calendar[Work Day Flag])

Work Days Passed =
VAR Yesterday = TODAY() - 1
RETURN CALCULATE(
    [Work Days Count],
    FILTER('Calendar',
        'Calendar'[Date] <= Yesterday &&
        'Calendar'[DateWithData] = TRUE()))

Remaining Work Days = [Work Days Count] - [Work Days Passed]

MaxDate = "Last data for: " & FORMAT(MAX(Sales[DOG_FROMDATE]), "dd mmm")
```

### 3. UX/UI Measures (visuals, rankings, colors)
```dax
-- Bottom 3 regions text card
AntiTop3_KK_Forecast_Text =
VAR CurrentPeriod = SELECTEDVALUE(Calendar[Period], "for the entire period")
VAR MaxRank = MAXX(ALL(DimRegion), [Sales FC Rank])
VAR Bottom3 = TOPN(3,
    FILTER(ALL(DimRegion),
        NOT(ISBLANK([Sales Forecast, %])) && [Sales FC Rank] >= MaxRank - 2),
    [Sales FC Rank], ASC)
VAR Result = CONCATENATEX(Bottom3,
    "• " & DimRegion[REGION] & ": " & FORMAT([Sales Forecast, %], "0.00 %", "en-US"),
    UNICHAR(10))
RETURN "🔻 Bottom 3 Regions by Forecast for " & CurrentPeriod & ":" & UNICHAR(10) & Result

-- Emoji rank indicator
Foreast% Emoji Rank =
IF(HASONEVALUE(DimRegion[REGION]),
    LOOKUPVALUE(EmojiTable[Emoji], EmojiTable[Rank], [Forecast,% Rank]))

-- Forecast rank
Forecast,% Rank = RANKX(ALL(DimRegion[Region]), [Sales Forecast, %], , DESC, Dense)
Sales FC Rank = RANKX(ALL(DimRegion[Region]), [Sales Forecast, %], , DESC, Dense)

-- Color coding for plan execution (absolute)
Plan Execution Colour HEX =
VAR PlEx = [Plan Execution]
RETURN IF(NOT(ISBLANK(PlEx)), IF(PlEx < 0, "#DE6A73", "#35AE78"), "#000000")

-- Color coding for plan execution % (red <50%, yellow 50–80%, green >80%)
Plan Execution Colour, % HEX =
VAR PlEx = [Plan Execution, %]
RETURN IF(NOT(ISBLANK(PlEx)),
    IF(PlEx < 0.5, "#DE6A73",
    IF(PlEx <= 0.8, "#D4AF37", "#27AE60")))

-- Sales forecast color
Sales Forecast Colour HEX =
VAR PlEx = [Sales Forecast]
RETURN IF(NOT(ISBLANK(PlEx)), IF(PlEx < 0, "#DE6A73", "#35AE78"), "#000000")

Sales Forecast, % Colour, % HEX =
VAR PlEx = [Sales Forecast, %]
RETURN IF(NOT(ISBLANK(PlEx)),
    IF(PlEx < 0.5, "#DE6A73",
    IF(PlEx <= 0.8, "#D4AF37", "#27AE60")))

-- Sales region ranking color (top 3 blue, bottom 2 red, rest grey)
Sales Region Colour HEX =
VAR CurrentRank = [Sales Region Rank]
VAR MaxRank = CALCULATE(MAXX(ALLSELECTED(DimRegion), [Sales Region Rank]), ALLSELECTED('Calendar'))
RETURN IF(CurrentRank <= 3, "#335F80",
       IF(CurrentRank = MaxRank || CurrentRank = MaxRank - 1, "#EB082E", "#E6E6E6"))

Sales Region Rank = RANKX(ALL(DimRegion[Region]), [Sales], , DESC, Dense)

-- Dynamic title measures
Selected Month =
IF(HASONEVALUE('Calendar'[Period]),
    "Selected: " & VALUES('Calendar'[Period]),
    "All months selected")

Selected Region =
IF(HASONEVALUE(DimRegion[Region]),
    "Selected: " & VALUES(DimRegion[Region]),
    "All regions selected")
```

---

## 🔒 Row-Level Security (RLS)

RLS configured to restrict data access by role:
- **Almaty** — Supervisor sees data only for Almaty city
- **Aqmola region** — Regional Manager sees data for Astana and Kokshetau

![RLS Setup](screenshots/RLS_Set.png)
![RLS View as Almaty](screenshots/RLS_View.png)

---

## 🛠️ Stack
Power BI | DAX | Power Query | Excel
