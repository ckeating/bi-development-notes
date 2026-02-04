# Main learnings so far from this project (Direct Query semantic model)

1. Pushing date math (and most likely most logic) into the source SQL
2. Correct DAX implementation:
   - "Getter measures"
   - Scalar values (such as putting a date or month name on a visual) 
   - Proper technique to get an "in window' measure to reference the date axis properly



## Pushing date math into source sql for date dimension

1. Main source: Caboodle DateDim
2. One row "Reporting Context" view
3. Addition logic in main date view to calculate fiscal components

### One row Reporting Context view


**Reporting "offsets" (relative to current rundate)**:
```sql
CAST(DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()) - 1, 0) AS date) AS LastClosedMonthStartDate,
	cast(eomonth(getdate(),-1) as date) as LastClosedMonthEndDate,
	CAST(DATEADD(YEAR, -1, DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()) - 1, 0)) AS date)
    AS LastClosedMonthStartDateMinus1,
	cast(dateadd(year,-1,eomonth(getdate(),-1)) as date) as LastClosedMonthEndDateMinus1,
```
through 4 years prior:
```sql
	cast(dateadd(year,-3,eomonth(getdate(),-1)) as date) as LastClosedMonthEndDateMinus3,
	CAST(DATEADD(YEAR, -4, DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()) - 1, 0)) AS date)
    AS LastClosedMonthStartDateMinus4,	
	cast(dateadd(year,-4,eomonth(getdate(),-1)) as date) as LastClosedMonthEndDateMinus4,
```


**Next, the relative date boundaries related to the last closed month. Starts with "anchor date" the last full month:**

```sql
FROM
(
    /* Last closed month end date */
    SELECT EOMONTH(DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()) - 1, 0)) AS ReportingAsOfDate
) a
```

this "a" table logic then gets reused via CROSS APPLY logic:

Note the use of **a.ReportingDate** in the following:

```sql
CROSS APPLY
(
    SELECT
        cast(getdate() as date) as CurrentDate,
		year(dateadd(month,3,getdate())) as CurrentFiscalYear,
		DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, getdate())) - 1, 10, 1) AS CurrentFiscalYearStartDate,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, getdate())),      9, 30) AS CurrentFiscalYearEndDate,

		
		a.ReportingAsOfDate,
		cast(datename(month,a.ReportingAsOfDate) as varchar(9)) as ReportingMonthName,
		cast(LEFT(DATENAME(month, a.ReportingAsOfDate), 3) AS nvarchar(3)) AS ReportingMonthShortName,

```

From this the reporting windows are calculated:

```sql
/* FY number for an Oct-start fiscal year */
        YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) AS ReportingFiscalYear,

        /* FY0 boundaries (Reporting FY) */
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 1, 10, 1) AS ReportingFiscalYearStartDate,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)),      9, 30) AS ReportingFiscalYearEndDate,
```
through


 ```sql
        /* FY-4 */
        YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 4 AS ReportingFiscalYearMinus4,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 5, 10, 1) AS ReportingFiscalYearMinus4StartDate,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 4,  9, 30) AS ReportingFiscalYearMinus4EndDate   
```

Then this is brought into the main select against the Caboodle dim:

```sql
alter view dbo.vw_SurgeryDates
as 
select
    -- Caboodle date dim - one row per date value
    dd.DateKey,
    dd.DateValue,
    dd.DisplayString,
    dd.DayOfWeek,
    dd.WeekNumber,
    dd.LastFridayDate,
    dd.MonthEndDate,
    dd.DayOfMonth,
    ..
    ..
    /* ============================================================
       Add (recommended): clean Fiscal* layer (per-date, from dd.DateValue)
       FY starts Oct 1 (FiscalYear = YEAR(DateValue + 3 months))
       ============================================================ */    
	f.FiscalMonthNumber,
    f.FiscalQuarterNumber,
    f.FiscalMonthName,
    f.FiscalMonthShortName,
    f.FiscalYearMonth,
    f.FiscalYearMonthSort,
    f.FiscalMonthStartDate,
    f.FiscalMonthEndDate,

    -- one row "reporting context":
    rpt.CurrentDate,
    rpt.LastClosedMonthStartDate,
    rpt.LastClosedMonthEndDate,
    rpt.LastClosedMonthStartDateMinus1,
    rpt.LastClosedMonthEndDateMinus1,
    rpt.LastClosedMonthStartDateMinus2,
    rpt.LastClosedMonthEndDateMinus2,
    rpt.LastClosedMonthStartDateMinus3,
    rpt.LastClosedMonthEndDateMinus3,
    rpt.LastClosedMonthStartDateMinus4,
    rpt.LastClosedMonthEndDateMinus4,
    rpt.CurrentFiscalYear,
    rpt.CurrentFiscalYearStartDate,
    rpt.CurrentFiscalYearEndDate,
    rpt.ReportingAsOfDate,
	rpt.ReportingMonthName,
	
FROM CDWReport.dbo.DateDim AS dd
CROSS JOIN HelixReportJDAT.dbo.vw_pwbi_SurgicalReportingDates AS rpt

```
then use the CROSS APPLY technique asaing as a way of propagating values for reuse purposes:

```sql
/* ============================================================
   Compute fiscal components ONCE per dd.DateValue, reuse everywhere
   ============================================================ */
CROSS APPLY (
	--pass-through primitives
    SELECT
        CAST(((MONTH(dd.DateValue) - 10 + 12) % 12) + 1 AS tinyint) AS FiscalMonthNumber,
        CAST(YEAR(DATEADD(month, 3, dd.DateValue)) AS int)          AS FiscalYearNumber
) fy
CROSS APPLY (
    select
		--pass-through primitives
		fy.FiscalMonthNumber as FiscalMonthNumber,
        CAST(((fy.FiscalMonthNumber - 1) / 3) + 1 AS tinyint) AS FiscalQuarterNumber,

		--display labels (calendar names, fiscal ordering)
        CAST(DATENAME(month,dd.DateValue) as nvarchar(9)) as FiscalMonthName,
        CAST(LEFT(DATENAME(month, dd.DateValue), 3) AS nvarchar(3)) AS FiscalMonthShortName,

        CAST(
            CONCAT(
                'FY',
                CAST(fy.FiscalYearNumber AS char(4)),
                '-',
                RIGHT('0' + CAST(fy.FiscalMonthNumber AS varchar(2)), 2)
            ) AS nvarchar(9)
        ) AS FiscalYearMonth,

        CAST((fy.FiscalYearNumber * 100) + fy.FiscalMonthNumber AS int) AS FiscalYearMonthSort,

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

Full reporting context sql:

```sql
CREATE view dbo.vw_pwbi_SurgicalReportingDates
as


/* ============================================================
   Reporting Context (last closed month -> ReportingAsOfDate)
   Fiscal Year = Oct 1 .. Sep 30  (FY = YEAR(AsOfDate + 3 months))
   Extends to FY-1 .. FY-4 with explicit start/end boundaries
   ============================================================ */

SELECT
	ctx.CurrentDate,
	
	--Offsets
	CAST(DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()) - 1, 0) AS date) AS LastClosedMonthStartDate,
	cast(eomonth(getdate(),-1) as date) as LastClosedMonthEndDate,
	CAST(DATEADD(YEAR, -1, DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()) - 1, 0)) AS date)
    AS LastClosedMonthStartDateMinus1,
	cast(dateadd(year,-1,eomonth(getdate(),-1)) as date) as LastClosedMonthEndDateMinus1,
	CAST(DATEADD(YEAR, -2, DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()) - 1, 0)) AS date)
    AS LastClosedMonthStartDateMinus2,
	cast(dateadd(year,-2,eomonth(getdate(),-1)) as date) as LastClosedMonthEndDateMinus2,
	CAST(DATEADD(YEAR, -3, DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()) - 1, 0)) AS date)
    AS LastClosedMonthStartDateMinus3,
	
	cast(dateadd(year,-3,eomonth(getdate(),-1)) as date) as LastClosedMonthEndDateMinus3,
	CAST(DATEADD(YEAR, -4, DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()) - 1, 0)) AS date)
    AS LastClosedMonthStartDateMinus4,	
	cast(dateadd(year,-4,eomonth(getdate(),-1)) as date) as LastClosedMonthEndDateMinus4,
    
	ctx.CurrentFiscalYear,
	ctx.CurrentFiscalYearStartDate,
	ctx.CurrentFiscalYearEndDate,

    ctx.ReportingAsOfDate,
	ctx.ReportingMonthName,
	ctx.ReportingMonthShortName,
	
    ctx.ReportingFiscalYear,
    ctx.ReportingFiscalYearStartDate,
    ctx.ReportingFiscalYearEndDate,

    ctx.ReportingFiscalYearMinus1,
    ctx.ReportingFiscalYearMinus1StartDate,
    ctx.ReportingFiscalYearMinus1EndDate,

    ctx.ReportingFiscalYearMinus2,
    ctx.ReportingFiscalYearMinus2StartDate,
    ctx.ReportingFiscalYearMinus2EndDate,

    ctx.ReportingFiscalYearMinus3,
    ctx.ReportingFiscalYearMinus3StartDate,
    ctx.ReportingFiscalYearMinus3EndDate,

    ctx.ReportingFiscalYearMinus4,
    ctx.ReportingFiscalYearMinus4StartDate,
    ctx.ReportingFiscalYearMinus4EndDate
FROM
(
    /* Last closed month end date */
    SELECT EOMONTH(DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()) - 1, 0)) AS ReportingAsOfDate
) a
CROSS APPLY
(
    SELECT
        cast(getdate() as date) as CurrentDate,
		year(dateadd(month,3,getdate())) as CurrentFiscalYear,
		DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, getdate())) - 1, 10, 1) AS CurrentFiscalYearStartDate,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, getdate())),      9, 30) AS CurrentFiscalYearEndDate,

		
		a.ReportingAsOfDate,
		cast(datename(month,a.ReportingAsOfDate) as varchar(9)) as ReportingMonthName,
		cast(LEFT(DATENAME(month, a.ReportingAsOfDate), 3) AS nvarchar(3)) AS ReportingMonthShortName,


        /* FY number for an Oct-start fiscal year */
        YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) AS ReportingFiscalYear,

        /* FY0 boundaries (Reporting FY) */
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 1, 10, 1) AS ReportingFiscalYearStartDate,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)),      9, 30) AS ReportingFiscalYearEndDate,

        /* FY-1 */
        YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 1 AS ReportingFiscalYearMinus1,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 2, 10, 1) AS ReportingFiscalYearMinus1StartDate,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 1,  9, 30) AS ReportingFiscalYearMinus1EndDate,

        /* FY-2 */
        YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 2 AS ReportingFiscalYearMinus2,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 3, 10, 1) AS ReportingFiscalYearMinus2StartDate,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 2, 9, 30) AS ReportingFiscalYearMinus2EndDate,

        /* FY-3 */
        YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 3 AS ReportingFiscalYearMinus3,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 4, 10, 1) AS ReportingFiscalYearMinus3StartDate,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 3,  9, 30) AS ReportingFiscalYearMinus3EndDate,

        /* FY-4 */
        YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 4 AS ReportingFiscalYearMinus4,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 5, 10, 1) AS ReportingFiscalYearMinus4StartDate,
        DATEFROMPARTS(YEAR(DATEADD(MONTH, 3, a.ReportingAsOfDate)) - 4,  9, 30) AS ReportingFiscalYearMinus4EndDate
) ctx;

```
## DAX implementation

### "Getter measures"

> Measures whose sole purpose it to retrieve and expose a single, authoritative scalar value from the model - not to calculate or aggregate business data
>
They:
* "get" a value that already exists
* make it usable in visuals, filters, or other measures.
* contain little or no business logic

They are **accessors** , not computations

**Examples from model**

The month name of January comes from 

![ex](../img/getter%20month%20name.png)

This starts with the source SQL for DateDim:

* First the last closed month is derived at runtime based on the current date:

```sql
FROM
(
    /* Last closed month end date */
    SELECT EOMONTH(DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()) - 1, 0)) AS ReportingAsOfDate
) a
```

* then the month name is derived from `ReportingAsOfDate`: The about report was run in February 2026. 
* Last full month is **January 2026**
* Month name is **January**

```sql

CROSS APPLY
(
    SELECT
		cast(datename(month,a.ReportingAsOfDate) as varchar(9)) as ReportingMonthName,
		cast(LEFT(DATENAME(month, a.ReportingAsOfDate), 3) AS nvarchar(3)) AS ReportingMonthShortName,
```



