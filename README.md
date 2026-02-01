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
