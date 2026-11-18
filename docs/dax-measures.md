# DAX, Power Query logic and data model

This page explains how the report's numbers are built. All data is synthetic. Table, column and measure names are exactly as they appear in the model. Code is shown as written, with a few template comments and unused variables trimmed for readability.

## 1. Data model

| Table | Role |
|---|---|
| `Invoice_Detailed` | **Fact table.** Weekly invoice summaries per customer and supplier, plus the churn rows appended from `ChurnBillings` |
| `FULL_BillTo` | **Customer dimension.** Region, territory, parent company, program start year |
| `FirstInvoiceDate` | First invoice month and year for each customer-and-supplier pair |
| `Calendar` | **Date table** (calculated) |
| `Churned Members` | One row per churned parent company |
| `Billings Summary` | Calculated table of New Logo and Expansion billings per relationship and year |
| `BillTo_C (2)` | Helper table with sales-rep, region and territory attributes |
| `ChurnBillings`, `ChurnTable` | Intermediate queries that build the churn rows (see section 2) |

### Relationships

| From (many side) | To (one side) | Status | Notes |
|---|---|---|---|
| `Invoice_Detailed[billToEntityDetailId]` | `FULL_BillTo[billToEntityDetailId]` | Active | Customer attributes for every invoice |
| `Invoice_Detailed[ParentID]` | `Churned Members[ParentID]` | Active | Links invoices to churned parent companies |
| `Invoice_Detailed[MonthStart]` | `Calendar[Date]` | Active | Invoice month |
| `FirstInvoiceDate[FirstMonthInvoice]` | `Calendar[Date]` | Active | First-invoice month |
| `Invoice_Detailed[bt_bf_key]` | `FirstInvoiceDate[bt_bf_key]` | **Inactive** | Activated inside measures with `USERELATIONSHIP` |
| `BillTo_C (2)[BillToEntityDetailId]` | `FULL_BillTo[billToEntityDetailId]` | Active | One-to-one, filters in both directions |
| `Billings Summary[billToEntityDetailId]` | `FULL_BillTo[billToEntityDetailId]` | Active | Customer attributes for the summary table |

`bt_bf_key` is a combined key, `<customer id>-<supplier id>`, so one value identifies one customer-and-supplier relationship. The date columns on `FULL_BillTo`, `ChurnTable`, `ChurnBillings` and `Churned Members` also have the automatic date relationships Power BI creates.

## 2. Power Query logic that feeds the measures

### Billings in USD
Amounts arrive in the supplier's currency. Only non-USD rows are converted, using the monthly exchange rate.

```powerquery
Custom = Table.AddColumn(#"Changed Type7", "Custom",
    each if [APCurrency] <> "USD" then [APTrxAmt] * [CADtoUSDFX] else [APTrxAmt])
// the result is renamed to "Billings US"
```

### New Logo, Expansion or neither
Each invoice row is labelled by comparing years.

```powerquery
NewSpend           = if [FirstInvoiceYear] = [InvoiceYear] then "New" else "Not"
New Customer Spend = if [billToYearStart]  = [InvoiceYear] then "New Customer" else "Not"
New Spend Label    = if [New Customer Spend] <> "Not" then "New Customer"
                     else if [NewSpend] <> "Not" then "New Program" else "Not"
// then renamed: "New Customer" -> "New Logo", "New Program" -> "Expansion"
```

- `FirstInvoiceYear` is the year the customer-and-supplier pair first invoiced.
- `billToYearStart` is the year the customer joined the program.
- **New Logo:** billings dated in the year the customer joined.
- **Expansion:** billings dated in the year a customer-and-supplier relationship first invoiced, for a customer that joined in an earlier year.
- **Not:** everything else, the ongoing base of existing relationships.

### Churn rows
The `ChurnBillings` query builds one negative row per month for each churned relationship.

1. Keep customers that have a termination date.
2. For each customer-and-supplier pair, sum the billings in the 12 months before the termination date and divide by 12.
3. Cross join to a weekly calendar (`ChurnTable`) and keep the weeks after termination, within the termination year.
4. Keep positive averages only, collapse to one row per month, and multiply by -1.
5. Label the rows `SpendType = "Churn"` and append them to the invoice table.

## 3. Calculated column

```dax
LabelChurn =
IF([SpendType] <> "Churn", [New Spend Label], "Churn")
```

`LabelChurn` has four values: `New Logo`, `Expansion`, `Not` and `Churn`. Every billing measure below filters on it.

## 4. Core billing measures

```dax
YTD Billings Total =
SUM('Invoice_Detailed'[Billings US])
```

```dax
New Logo Billings =
CALCULATE(
    SUM(Invoice_Detailed[Billings US]),
    Invoice_Detailed[LabelChurn] = "New Logo"
)
```

```dax
Expansion Billings =
CALCULATE(
    SUM(Invoice_Detailed[Billings US]),
    Invoice_Detailed[LabelChurn] = "Expansion"
)
```

```dax
Churn Billings =
CALCULATE(
    SUM(Invoice_Detailed[Billings US]),
    Invoice_Detailed[LabelChurn] = "Churn"
)
```

`Churn Billings Card` is the same calculation wrapped so a card shows 0 instead of blank:

```dax
Churn Billings Card =
VAR a = CALCULATE(
    SUM(Invoice_Detailed[Billings US]),
    Invoice_Detailed[LabelChurn] = "Churn"
)
RETURN IF(ISBLANK(a), 0, a)
```

### Running total and first-month billings

```dax
Cumulative Billings =
VAR CurrentWeek =
    MAX('Invoice_Detailed'[MonthStart])
RETURN
CALCULATE(
    SUM('Invoice_Detailed'[Billings US]),
    FILTER(
        ALL('Invoice_Detailed'[MonthStart]),
        'Invoice_Detailed'[MonthStart] <= CurrentWeek
    )
)
```

`YTD Total Billings` and `First Month Billings` use the inactive relationship to `FirstInvoiceDate`:

```dax
YTD Total Billings =
ROUND(
    CALCULATE(
        SUM('Invoice_Detailed'[Billings US]),
        USERELATIONSHIP(
            'FirstInvoiceDate'[bt_bf_key], 'Invoice_Detailed'[bt_bf_key]
        )
    ), 0)
```

```dax
First Month Billings =
ROUND(
    SUMX(
        'Calendar',
        VAR FirstMonth = 'Calendar'[Date]
        RETURN
            CALCULATE(
                SUM('Invoice_Detailed'[Billings US]),
                'Invoice_Detailed'[MonthStart] = FirstMonth,
                USERELATIONSHIP(
                    'FirstInvoiceDate'[bt_bf_key], 'Invoice_Detailed'[bt_bf_key]
                )
            )
    ), 0)
```

## 5. Year-to-date new billings

```dax
2025 New Billings =
VAR CurrentMonth =
    SELECTEDVALUE('Calendar'[Month Number])
RETURN
CALCULATE(
    [Expansion Billings] + [New Logo Billings],
    FILTER(
        ALL('Calendar'),
        'Calendar'[Year] = 2025
            && 'Calendar'[Month Number] <= CurrentMonth
    )
)
```

```dax
2026 New Billings =
CALCULATE(
    [Expansion Billings] + [New Logo Billings],
    FILTER(
        ALL('Calendar'),
        'Calendar'[Year] = 2026 &&
        'Calendar'[Date] <= MAX('Calendar'[Date])
    )
)
```

## 6. Cohort measures

A **cohort** is the set of customer-and-supplier relationships that produced New Logo or Expansion billings in a year. These measures follow that cohort forward.

```dax
2025 Cohort Cumulative Billings =
VAR CurrentDate =
    MAX ( 'Calendar'[Date] )
-- Bill-to/bill-for relationships with new logo or expansion billings in 2025
VAR Cohort2025 =
    CALCULATETABLE (
        FILTER (
            VALUES ( Invoice_Detailed[bt_bf_key] ),
            [Expansion Billings] + [New Logo Billings] > 0
        ),
        ALL ( 'Calendar' ),
        'Calendar'[Year] = 2025
    )
-- 2025 portion: new logo + expansion only
VAR NewBillings2025 =
    CALCULATE (
        [Expansion Billings] + [New Logo Billings],
        ALL ( 'Calendar' ),
        'Calendar'[Date] >= DATE ( 2025, 1, 1 ),
        'Calendar'[Date] <= MIN ( CurrentDate, DATE ( 2025, 12, 31 ) ),
        Cohort2025
    )
-- 2026+ portion: all billings from the cohort
VAR AllBillings2026 =
    IF (
        CurrentDate >= DATE ( 2026, 1, 1 ),
        CALCULATE (
            [YTD Billings Total],
            ALL ( 'Calendar' ),
            'Calendar'[Date] >= DATE ( 2026, 1, 1 ),
            'Calendar'[Date] <= CurrentDate,
            Cohort2025
        ),
        0
    )
RETURN
    IF ( CurrentDate <= EOMONTH ( TODAY (), 0 ), NewBillings2025 + AllBillings2026 )
```

In words: take the relationships that had new billings in 2025. Add their 2025 new billings to everything they have billed so far in 2026. This is the "how do the 2025 wins keep contributing" number.

```dax
2026 Cohort Cumulative Billings =
VAR CurrentDate =
    MAX ( 'Calendar'[Date] )
VAR Cohort2025 =
    CALCULATETABLE (
        FILTER (
            VALUES ( Invoice_Detailed[bt_bf_key] ),
            [Expansion Billings] + [New Logo Billings] > 0
        ),
        ALL ( 'Calendar' ),
        'Calendar'[Year] = 2026
    )
VAR NewBillings2026 =
    CALCULATE (
        [Expansion Billings] + [New Logo Billings],
        ALL ( 'Calendar' ),
        'Calendar'[Date] >= DATE ( 2026, 1, 1 ),
        'Calendar'[Date] <= MIN ( CurrentDate, DATE ( 2026, 12, 31 ) ),
        Cohort2025
    )
RETURN
    IF ( CurrentDate <= EOMONTH ( TODAY (), 0 ), NewBillings2026 )
```

```dax
2025 Year-End Total =
VAR IsYE =
    MAX ( 'Calendar'[Year] ) = 2025
    && MAX ( 'Calendar'[Month Number] ) = 12
RETURN
    IF ( IsYE, [2025 Cohort Cumulative Billings] )
```

## 7. Headline text

A single measure builds a sentence for a text box, so the narrative updates with the data.

```dax
Gross New 2025 =
CALCULATE ( [New Logo Billings] + [Expansion Billings], 'Calendar'[Year] = 2025 )

Churn Billings 2025 =
CALCULATE ( [Churn Billings], 'Calendar'[Year] = 2025 )
```

```dax
Headline =
VAR Gross =  [Gross New 2025]
VAR Churn = ABS ( COALESCE ( [Churn Billings 2025], 0 ) )
VAR Net   = Gross - Churn
VAR Pct   = DIVIDE ( Churn, Gross )
VAR Cum   = [2025 Cohort Cumulative Billings]
VAR YTD26 = Cum - Gross
RETURN
"In 2025 Sales added " & FORMAT ( Gross / 1000000, "$0.0" ) & "M in new billings; churn removed "
& FORMAT ( Churn / 1000000, "$0.0" ) & "M (" & FORMAT ( Pct, "0%" ) & " of New), leaving "
& FORMAT ( Net / 1000000, "$0.0" ) & "M net.  "
& "These 2025 new cohorts have generated an additional "
& FORMAT ( YTD26 / 1000000, "$0.0" ) & "M in billings in 2026 YTD."
```

## 8. Calculated tables

### Calendar
A date table from the first invoice month to today.

```dax
Calendar =
VAR StartDate = MIN('Invoice_Detailed'[MonthStart])
VAR EndDate = TODAY()
RETURN
ADDCOLUMNS (
    CALENDAR ( StartDate, EndDate ),
    "Year", YEAR ( [Date] ),
    "Quarter Number", QUARTER ( [Date] ),
    "Quarter", "Q" & FORMAT ( [Date], "Q" ),
    "Month Number", MONTH ( [Date] ),
    "Month Name", FORMAT ( [Date], "mmmm" ),
    "Month Short", FORMAT ( [Date], "mmm" ),
    "Month-Year", FORMAT ( [Date], "mmm-yyyy" ),
    "Month-Year Sort", YEAR ( [Date] ) * 100 + MONTH ( [Date] ),
    "Week Number", WEEKNUM ( [Date], 2 ),
    "Day of Month", DAY ( [Date] ),
    "Day Name", FORMAT ( [Date], "dddd" ),
    "Day of Week Number", WEEKDAY ( [Date], 2 ),
    "Week Begin", [Date] - WEEKDAY([Date],2) + 1
)
```

### Billings Summary
One row per relationship, year and label, with the total across all years.

```dax
Billings Summary =
ADDCOLUMNS(
    SUMMARIZE(
        FILTER(
            'Invoice_Detailed',
            'Invoice_Detailed'[New Spend Label] IN { "New Logo", "Expansion" }
        ),
        'Invoice_Detailed'[bt_bf_key],
        'Invoice_Detailed'[BillToEntityDetailId],
        'Invoice_Detailed'[InvoiceYear],
        'Invoice_Detailed'[New Spend Label],
        "Total Billings US", SUM('Invoice_Detailed'[Billings US])
    ),
    "Total Billings US (All Years)",
    CALCULATE(
        SUM('Invoice_Detailed'[Billings US]),
        ALLEXCEPT('Invoice_Detailed', 'Invoice_Detailed'[bt_bf_key])
    )
)
```

## 9. Churned Members columns

`Churned Members` has one row per parent company that has churned. Three calculated columns size the loss:

```dax
12MonthPurchases =
VAR TermDate =
    MAX ( 'Churned Members'[MaxTerminationDate] )
VAR StartDate =
    EDATE ( TermDate, -12 )
RETURN
CALCULATE (
    SUM ( 'Invoice_Detailed'[APTrxAmt] ),
    FILTER (
        ALL ( 'Invoice_Detailed'[MonthStart] ),
        'Invoice_Detailed'[MonthStart] > StartDate
            && 'Invoice_Detailed'[MonthStart] <= TermDate
    )
)
```

```dax
MonthlyAVG = [12MonthPurchases] / 12
YearsChurn = -[MonthlyAVG] * [RemainingMonths]
```

`RemainingMonths` is 12 minus the termination month, so `YearsChurn` is the billing the customer would have contributed for the rest of the year it left.

## 10. Customer-level columns

```dax
2025Spend =
CALCULATE(
    SUM(Invoice_Detailed[Billings US]),
    FILTER( ALL('Calendar'), YEAR('Calendar'[Date]) = 2025 )
)
```

`2026Spend` is the same with 2026. `REGION corr` on `FULL_BillTo` simply returns `[Region]`, and the region slicer uses it.

## 11. Notes

- `Calendar` ends at `TODAY()`, and the cohort measures use `TODAY()` to avoid showing future months. Results shift as the date changes.
- The 2025 and 2026 measures have the years written in. For other years, copy the measures and change the year.
- `Sales` on `FirstInvoiceDate` is a plain sum of `APTrxAmt`, and `Churn Programs` is a placeholder measure.
