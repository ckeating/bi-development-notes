# bi-development-notes
```sql
CROSS APPLY (
    SELECT
        -- pass-through primitives
        fy.FiscalMonthNumber AS FiscalMonthNumber,
        CAST(((fy.FiscalMonthNumber - 1) / 3) + 1 AS tinyint) AS FiscalQuarterNumber,

        -- display labels (calendar names, fiscal ordering)
        CAST(DATENAME(month, dd.DateValue) AS nvarchar(9)) AS FiscalMonthName,
        CAST(LEFT(DATENAME(month, dd.DateValue), 3) AS nvarchar(3)) AS FiscalMonthShortName,

        -- composite identifiers
        CAST(
            CONCAT(
                'FY',
                CAST(fy.FiscalYearNumber AS char(4)),
                '-',
                RIGHT('0' + CAST(fy.FiscalMonthNumber AS varchar(2)), 2)
            ) AS nvarchar(9)
        ) AS FiscalYearMonth,

        CAST((fy.FiscalYearNumber * 100) + fy.FiscalMonthNumber AS int) AS FiscalYearMonthSort,

        -- boundaries
        CAST(
            DATEADD(
                month,
                DATEDIFF(month, 0, DATEADD(month, -9, dd.DateValue)),
                0
            ) AS date
        ) AS FiscalMonthStartDate,

        CAST(EOMONTH(DATEADD(month, -9, dd.DateValue)) AS date) AS FiscalMonthEndDate
) f

```
## MIN(Date) vs SELECTEDVALUE(Date) on Axes

**Short answer:** yes — `MIN(Date)` (or `MAX(Date)`) is *much* more flexible than `SELECTEDVALUE(Date)` for axis-driven visuals.  
Long answer below, because this is one of those “once you see it, you never unsee it” Fabric / Power BI concepts.

---

## Why `SELECTEDVALUE()` bites you on axes

`SELECTEDVALUE()` only works when **exactly one value** exists in the filter context.

That’s a fragile assumption on visuals.

### Works when:
- Axis = `DateDim[Date]` (daily grain)
- Card / KPI
- Slicer with single-select **ON**

### Breaks when:
- Axis = Month, Quarter, Year
- Axis = text column (Month Name)
- Tooltips
- Multi-select slicers
- Any visual that groups rows

When Power BI evaluates a **month bucket**, the filter context contains **many dates**.

**Result:**

