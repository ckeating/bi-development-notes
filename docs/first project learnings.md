## Components of source SQL

1. Caboodle DateDim
2. One row "Reporting Context" view
   - indent

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
