# DAX measures

All measures live in the `_Measures` table. Each entry has a plain-English description (also stored in the measure's Description property in the model) and the formula with comments.

## Contents

- **Base:** Complaints, Complaints Exact
- **Rates:** Timely Response %, Relief %, Avg Days to Company
- **Shares and ranking:** % of Total Complaints, % of All Complaints, Company Rank
- **Per capita:** Complaints per 100K
- **Time intelligence:** Complaints PY, Complaints YoY%, YoY Context, 12M Change %, 12M Context, Complaints Rolling 3M Avg
- **Drill-through:** Selected Company
- **Model objects (not measures):** dim_Date calculated table, Metric field parameter

---

## Base

### Complaints
Number of complaints in the current filters. Counts rows in fct_Complaints. Base measure that most other measures build on.

```
Complaints =
// Counts complaint rows that pass the current filters.
// Base measure: most other measures reuse it.
COUNTROWS ( fct_Complaints )
```

### Complaints Exact
Complaints as text with thousands separators, used on the Overview card so it shows the full number (e.g. 548,444) instead of an abbreviation like "1M".

```
Complaints Exact =
// Text version of Complaints so the card can't auto-abbreviate it.
// Use only for display; use [Complaints] in calculations.
FORMAT ( [Complaints], "#,0" )
```

---

## Rates

### Timely Response %
Share of complaints where the company responded on time, out of all complaints in the current filters.

```
Timely Response % =
// On-time complaints divided by all complaints.
// + 0 shows 0.0% instead of blank when a company has no on-time responses.
DIVIDE (
    CALCULATE ( [Complaints], fct_Complaints[Timely response?] = "Yes" ) + 0,
    [Complaints]
)
```

### Relief %
Share of complaints closed with monetary or non-monetary relief for the consumer.

```
Relief % =
// Complaints closed with monetary or non-monetary relief, divided by all complaints.
// + 0 shows 0.0% instead of blank when a company gave no relief.
DIVIDE (
    CALCULATE (
        [Complaints],
        fct_Complaints[Company response to consumer]
            IN { "Closed with monetary relief", "Closed with non-monetary relief" }
    ) + 0,
    [Complaints]
)
```

### Avg Days to Company
Average number of days between the CFPB receiving a complaint and sending it to the company.

```
Avg Days to Company =
// For each complaint, days from Date received to Date sent to company,
// then the average. AVERAGEX goes row by row through fct_Complaints.
AVERAGEX (
    fct_Complaints,
    DATEDIFF ( fct_Complaints[Date received], fct_Complaints[Date sent to company], DAY )
)
```

---

## Shares and ranking

### % of Total Complaints
This row's complaints as a share of everything the visual shows. Rows add up to 100%. Used on the channel chart.

```
% of Total Complaints =
// This row's complaints divided by the visual's total.
// ALLSELECTED removes the row's own filter but keeps slicers and visual filters.
DIVIDE ( [Complaints], CALCULATE ( [Complaints], ALLSELECTED () ) )
```

### % of All Complaints
A company's complaints as a share of all companies' complaints, ignoring the table's Top 15 filter. Used as "Share" on the Companies table.

```
% of All Complaints =
// This company's complaints divided by complaints for all companies.
// ALL ( dim_Company ) removes the company filter (including Top 15) but keeps the slicers.
DIVIDE ( [Complaints], CALCULATE ( [Complaints], ALL ( dim_Company ) ) )
```

### Company Rank
A company's rank by complaint count, where 1 = most complaints, within the current slicer filters.

```
Company Rank =
// Ranks each company by Complaints, highest = 1.
// ALL ( dim_Company[Company] ) ranks against every company, not just the current row.
RANKX ( ALL ( dim_Company[Company] ), [Complaints] )
```

---

## Per capita

### Complaints per 100K
Complaints per 100,000 residents, using Census July 2025 state population estimates.

```
Complaints per 100K =
// Complaints divided by state population, times 100,000.
// Population comes from dim_State (Census Vintage 2025).
DIVIDE ( [Complaints], SUM ( dim_State[Population] ) ) * 100000
```

---

## Time intelligence

### Complaints PY
Complaints for the same dates one year earlier.

```
Complaints PY =
// Same measure, with the date filter shifted back one year.
CALCULATE ( [Complaints], SAMEPERIODLASTYEAR ( dim_Date[Date] ) )
```

### Complaints YoY%
Percent change versus the same dates one year earlier. Most meaningful for ranges of one year or less; the Overview card uses 12M Change % instead.

```
Complaints YoY% =
// (This period - same period last year) / same period last year.
// Variables store each number once so it is calculated only once.
VAR CurrentYear = [Complaints]
VAR PriorYear = [Complaints PY]
RETURN
    DIVIDE ( CurrentYear - PriorYear, PriorYear )
```

### YoY Context
Text label showing the prior-year count and dates that Complaints YoY% compares against.

```
YoY Context =
// Builds a label like "vs 312,450 in 10/1/24 to 9/30/25".
// Shows a message instead when there is no prior-year data.
VAR PriorYear = [Complaints PY]
VAR StartDate = MIN ( dim_Date[Date] )
VAR EndDate = MAX ( dim_Date[Date] )
RETURN
    IF (
        ISBLANK ( PriorYear ),
        "No prior-year data for these dates",
        "vs " & FORMAT ( PriorYear, "#,0" ) & " in "
            & FORMAT ( EDATE ( StartDate, -12 ), "m/d/yy" ) & " to "
            & FORMAT ( EDATE ( EndDate, -12 ), "m/d/yy" )
    )
```

### 12M Change %
Percent change for the 12 months ending on the date slicer's end date versus the 12 months before. Ignores the slicer's start date so it always compares equal periods. Blank without two full years of data.

```
12M Change % =
// Last 12 months vs. the 12 before, both ending at the slicer's end date.
// The slicer's start date is ignored on purpose, so the periods are always equal.
// Returns blank when there isn't 24 months of data to compare.
VAR EndDate = MAX ( dim_Date[Date] )
VAR DataStart = CALCULATE ( MIN ( dim_Date[Date] ), REMOVEFILTERS ( dim_Date ) )
VAR Last12 = CALCULATE ( [Complaints], DATESINPERIOD ( dim_Date[Date], EndDate, -12, MONTH ) )
VAR Prior12 = CALCULATE ( [Complaints], DATESINPERIOD ( dim_Date[Date], EDATE ( EndDate, -12 ), -12, MONTH ) )
RETURN
    IF ( EDATE ( EndDate, -24 ) + 1 < DataStart, BLANK (), DIVIDE ( Last12 - Prior12, Prior12 ) )
```

### 12M Context
Text label for the 12-month change card showing which dates are compared.

```
12M Context =
// Builds the label "12 months to [end date] vs. the 12 before".
VAR EndDate = MAX ( dim_Date[Date] )
RETURN
    "12 months to " & FORMAT ( EndDate, "m/d/yy" ) & " vs. the 12 before"
```

### Complaints Rolling 3M Avg
Average monthly complaints over the 3 months ending in the current month. Smooths month-to-month noise on the trend line.

```
Complaints Rolling 3M Avg =
// Complaints over the 3 months ending on the current month's last date, divided by 3.
// Always divides by 3, so the first two months of data read low.
DIVIDE (
    CALCULATE (
        [Complaints],
        DATESINPERIOD ( dim_Date[Date], MAX ( dim_Date[Date] ), -3, MONTH )
    ),
    3
)
```

---

## Drill-through

### Selected Company
The company name shown as the title of the Company Detail drill-through page.

```
Selected Company =
// The single company passed in by the drill-through; a fallback label if there are several.
SELECTEDVALUE ( dim_Company[Company], "Multiple companies" )
```

---

## Model objects (not measures)

### dim_Date (calculated table)
Date table from Jan 2023 through the end of the last month in the data. Variable names avoid DAX function names (LastDate and FirstDate are functions).

```
dim_Date =
VAR MaxDate = MAX ( fct_Complaints[Date received] )
RETURN
    ADDCOLUMNS (
        CALENDAR ( DATE ( 2023, 1, 1 ), EOMONTH ( MaxDate, 0 ) ),
        "Year", YEAR ( [Date] ),
        "Month Number", MONTH ( [Date] ),
        "Month", FORMAT ( [Date], "mmm" ),
        "Year Month", FORMAT ( [Date], "yyyy-mm" ),
        "Month Start", DATE ( YEAR ( [Date] ), MONTH ( [Date] ), 1 ),
        "Quarter", "Q" & QUARTER ( [Date] ),
        "Is Complete Month", EOMONTH ( [Date], 0 ) <= EOMONTH ( MaxDate, -1 )
    )
```

### Metric (field parameter)
Created with Modeling > New parameter > Fields. Lets viewers switch the Geography page's state chart between Complaints, Complaints per 100K, Timely Response % and Relief %. Power BI generates its table definition automatically.
