# Power BI matrix to DAX query: net hires by RD / ED and month

Purpose: reproduce a Power BI matrix (rows: Regional Director > Executive Director; columns: completed months of the current year; value: net hires; with subtotals and totals) as a single DAX query for the Power Automate action **Run a query against a dataset** (Power BI connector).

Status: DRAFT. Table, column and measure names are placeholders (marked `<<...>>`) until Ralf confirms the real model.

## 1. Assumptions to confirm

1. "Net hires" = hires minus separations. If the model has no separations, replace the measure with a plain hires count.
2. Rows: `Org[RD]` > `Org[ED]` (two levels of the same hierarchy). Subtotal per RD, plus a grand total.
3. A date table `dimDates` marked as a date table, with `Year` and `MonthNumber` columns, related to the fact table.
4. "Current month" is defined by `TODAY()`. In the Power BI service this uses UTC, so on the 1st of a month (local time) the result can lag by a few hours.
5. In January there are no completed months, so every month column and the total are blank.

## 2. Query (recommended: fixed 12 month columns, no pivot needed)

Each month is its own measure column. Months that are not yet completed return BLANK, so the result has a fixed schema (`[M01]` to `[M12]`) and rolls forward by itself. `[Total]` adds up only the completed months.

```dax
DEFINE
    
    MEASURE 'Employee Hires & Terms'[M01] = IF ( 1  < MONTH ( TODAY () ), CALCULATE ( [Net Hires Count], 'dimDates'[YearNumber] = YEAR ( TODAY () ), MONTH('dimDates'[DateValue]) = 1 ) )
    MEASURE 'Employee Hires & Terms'[M02] = IF ( 2  < MONTH ( TODAY () ), CALCULATE ( [Net Hires Count], 'dimDates'[YearNumber] = YEAR ( TODAY () ), MONTH('dimDates'[DateValue]) = 2 ) )
    MEASURE 'Employee Hires & Terms'[M03] = IF ( 3  < MONTH ( TODAY () ), CALCULATE ( [Net Hires Count], 'dimDates'[YearNumber] = YEAR ( TODAY () ), MONTH('dimDates'[DateValue]) = 3 ) )
    MEASURE 'Employee Hires & Terms'[M04] = IF ( 4  < MONTH ( TODAY () ), CALCULATE ( [Net Hires Count], 'dimDates'[YearNumber] = YEAR ( TODAY () ), MONTH('dimDates'[DateValue]) = 4 ) )
    MEASURE 'Employee Hires & Terms'[M05] = IF ( 5  < MONTH ( TODAY () ), CALCULATE ( [Net Hires Count], 'dimDates'[YearNumber] = YEAR ( TODAY () ), MONTH('dimDates'[DateValue]) = 5 ) )
    MEASURE 'Employee Hires & Terms'[M06] = IF ( 6  < MONTH ( TODAY () ), CALCULATE ( [Net Hires Count], 'dimDates'[YearNumber] = YEAR ( TODAY () ), MONTH('dimDates'[DateValue]) = 6 ) )
    MEASURE 'Employee Hires & Terms'[M07] = IF ( 7  < MONTH ( TODAY () ), CALCULATE ( [Net Hires Count], 'dimDates'[YearNumber] = YEAR ( TODAY () ), MONTH('dimDates'[DateValue]) = 7 ) )
    MEASURE 'Employee Hires & Terms'[M08] = IF ( 8  < MONTH ( TODAY () ), CALCULATE ( [Net Hires Count], 'dimDates'[YearNumber] = YEAR ( TODAY () ), MONTH('dimDates'[DateValue]) = 8 ) )
    MEASURE 'Employee Hires & Terms'[M09] = IF ( 9  < MONTH ( TODAY () ), CALCULATE ( [Net Hires Count], 'dimDates'[YearNumber] = YEAR ( TODAY () ), MONTH('dimDates'[DateValue]) = 9 ) )
    MEASURE 'Employee Hires & Terms'[M10] = IF ( 10 < MONTH ( TODAY () ), CALCULATE ( [Net Hires Count], 'dimDates'[YearNumber] = YEAR ( TODAY () ), MONTH('dimDates'[DateValue]) = 10 ) )
    MEASURE 'Employee Hires & Terms'[M11] = IF ( 11 < MONTH ( TODAY () ), CALCULATE ( [Net Hires Count], 'dimDates'[YearNumber] = YEAR ( TODAY () ), MONTH('dimDates'[DateValue]) = 11 ) )
    MEASURE 'Employee Hires & Terms'[M12] = IF ( 12 < MONTH ( TODAY () ), CALCULATE ( [Net Hires Count], 'dimDates'[YearNumber] = YEAR ( TODAY () ), MONTH('dimDates'[DateValue]) = 12 ) )

    MEASURE 'Employee Hires & Terms'[Total] =
        CALCULATE (
            [Net Hires Count],
            'dimDates'[YearNumber] = YEAR ( TODAY () ),
            MONTH('dimDates'[DateValue]) < MONTH ( TODAY () )
        )

EVALUATE
VAR _Rolled =
    SUMMARIZECOLUMNS (
        ROLLUPADDISSUBTOTAL ( 'dimManagementLocationHierarchy'[Regional Director Combined], "IsGrandTotal" ),
        ROLLUPADDISSUBTOTAL ( 'dimManagementLocationHierarchy'[Executive Director Combined], "IsRDSubtotal" ),
        "M01", [M01], "M02", [M02], "M03", [M03], "M04", [M04],
        "M05", [M05], "M06", [M06], "M07", [M07], "M08", [M08],
        "M09", [M09], "M10", [M10], "M11", [M11], "M12", [M12],
        "Total", [Total]
    )
RETURN
    // Drop the "ED across all RDs" rows that the two independent rollups also generate.
    FILTER ( _Rolled, NOT ( [IsGrandTotal] && NOT [IsRDSubtotal] ) )
ORDER BY
    [IsGrandTotal] ASC,
    'Org'[RD] ASC,
    [IsRDSubtotal] ASC,
    'Org'[ED] ASC
```

How to read the rows

| `IsGrandTotal` | `IsRDSubtotal` | Meaning | RD / ED values |
|---|---|---|---|
| false | false | Detail row (one ED) | both filled |
| false | true | Subtotal of an RD | ED is null |
| true | true | Grand total | both null |

Why `ROLLUPADDISSUBTOTAL` on each column: it adds the subtotal rows and an `IsSubtotal`-style flag column per rolled column ([ROLLUPADDISSUBTOTAL](https://learn.microsoft.com/dax/rollupaddissubtotal-function-dax), [SUMMARIZECOLUMNS](https://learn.microsoft.com/dax/summarizecolumns-function-dax)). Use `ROLLUPGROUP` if you ever need to roll up several columns as one unit ([ROLLUPGROUP](https://learn.microsoft.com/dax/rollupgroup-function-dax)).

Alternative: a "long" format (one row per RD/ED/month, with `ROLLUPADDISSUBTOTAL` also on the month column) keeps a dynamic number of months, but then the flow must pivot the rows. Only use it if the number of month columns must change in the email itself.

## 3. Consuming the result in Power Automate

1. Action: **Power BI > Run a query against a dataset** (workspace, dataset, query text above). The connector wraps the REST API [Execute Queries](https://learn.microsoft.com/rest/api/power-bi/datasets/execute-queries).
2. The response holds `results[0].tables[0].rows`. Column keys keep DAX naming, for example `Org[RD]`, `Org[ED]`, `[IsGrandTotal]`, `[IsRDSubtotal]`, `[M01]`, `[Total]`.
3. Use **Parse JSON** (or reference `body('Run_a_query_against_a_dataset')?['firstTableRows']`) and then **Select** to map each row to the output columns, for example `coalesce(item()?['[M01]'], 0)`. Blank measures come back as null or the key can be missing, so always use `?[...]` and `coalesce`.
4. Build the table with **Create HTML table** (keep inline styles, since Outlook's Word engine ignores most CSS; see the Outlook guides in this folder). To bold subtotal and grand-total rows, add a `Select` that wraps the text in `<b>` when `item()?['[IsRDSubtotal]']` is true, then fix the escaped tags with `replace()` before sending.
5. Future months: the fixed schema returns `[M07]` to `[M12]` as null in months not yet completed. Either show a dash, or build the Select with a Switch on `month(utcNow())` so the email only lists completed months.
6. Time zone: `TODAY()` runs in UTC on the service. Schedule the flow at least a few hours after local midnight on the 1st.

## 4. Limits to keep in mind

- Execute Queries (JSON) hard limit: 100,000 rows and 1,000,000 values per query; this query returns about (number of EDs + number of RDs + 1) rows by 14 values, so it is far below that ([best practices](https://learn.microsoft.com/power-bi/developer/execute-dax-queries-arrow/best-practices), [Execute Queries vs Execute DAX Queries](https://learn.microsoft.com/power-bi/developer/execute-dax-queries-arrow/overview#considerations-and-limitations)).
- One query per request in this API. The Arrow-based Execute DAX Queries API needs Premium/Fabric and cannot be consumed by the standard connector.
- Other limits (15 MB, 120 requests/minute per user, throttling) are in [power-automate-power-bi-reports.md](power-automate-power-bi-reports.md).
- `SUMMARIZECOLUMNS` drops rows where every measure is blank, so an ED with no activity in any month will not appear. If every ED must appear with zeros, add `+ 0` in the measures (this can change the row count and subtotals) or tell me and I will switch to a base-table approach.
- RLS: if the dataset uses row-level security, the connection user's role applies; a service principal does not get RLS. Use the same account that owns the flow.

## 5. Open questions for Ralf

1. Real table and column names for the hierarchy (RD, ED), the fact table and the date table (and its Year / MonthNumber columns).
2. Definition of "net hires" (existing model measure name, or hires minus separations on which columns).
3. Should EDs without activity still appear (with zeros)?
4. Should the email list only completed months (so I add the month switch), or is a fixed 12-column table with empty future months fine?
