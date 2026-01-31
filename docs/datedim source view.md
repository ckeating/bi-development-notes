# Datedim source view

```sql

/* =====================================================================
   DateDim Source View (Caboodle DateDim + 1-row Reporting Context)
   Refactor: compute fiscal “clean layer” once via CROSS APPLY, reuse fields
   Output columns are IDENTICAL to your prior integrated version:
     - all dd.* fields you selected
     - the 8 added fiscal fields:
         FiscalMonthNumber, FiscalQuarterNumber, FiscalMonthName, FiscalMonthShortName,
         FiscalYearMonth, FiscalYearMonthSort, FiscalMonthStartDate, FiscalMonthEndDate
     - all rpt.* fields you selected
   ===================================================================== */

SELECT
    -- Caboodle date dim - one row per date value
    dd.DateKey,
    dd.DateValue,
    dd.DisplayString,
    dd.DayOfWeek,
    dd.WeekNumber,
    dd.LastFridayDate,
    dd.MonthEndDate,
    dd.DayOfMonth,
    dd.MonthName,
    dd.MonthNumber,
    dd.QuarterNumber,
    dd.DayOfYear,
    dd.EpicDte,
    dd.EpicDat,
    dd.EpicInstantAtMidnight,
    dd.Year,
    dd.OccurrenceInMonth,
    dd.TomorrowDate,
    dd.YearMonth,
    dd.FiscalYear,
    dd.FiscalMonth,
    dd.FiscalQuarter,
    dd.Weekend,
    dd.IndexDayOfWeek,
    dd.DayOfWeekAbbreviation,
    dd.FormattedMonthYear,
    dd.FormattedQuarterNumber,
    dd.FormattedQuarterYear,
    dd.IsDayAfterHoliday,
    dd.IsDayBeforeHoliday,
    dd.IsHoliday,
    dd.IsoWeekNumber,
    dd.IsoWeekYear,
    dd.IsoYearWeek,
    dd.IsWeekendIncludingFriday,
    dd.MonthNameAbbreviation,
    dd.MonthStartDate,
    dd.MonthYear,
    dd.NextYearDate,
    dd.PreviousYearDate,
    dd.QuarterEndDate,
    dd.QuarterStartDate,
    dd.QuarterYear,
    dd.WeekEndDate,
    dd.WeekStartDate,
    dd.WeekYear,
    dd.YearEndDate,
    dd.YearFormattedMonth,
    dd.YearFormattedQuarter,
    dd.YearQuarter,
    dd.YearStartDate,
    dd.YearWeek,
    dd.YMFiscalYear_X,
    dd.YMFiscalYearBeginDate_X,
    dd.YMFiscalYearEndDate_X,
    dd.YMFiscalMonth_X,
    dd.YMFiscalQuarter_X,

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
    rpt.ReportingFiscalYear,
    rpt.ReportingFiscalYearStartDate,
    rpt.ReportingFiscalYearEndDate,
    rpt.ReportingFiscalYearMinus1,
    rpt.ReportingFiscalYearMinus1StartDate,
    rpt.ReportingFiscalYearMinus1EndDate,
    rpt.ReportingFiscalYearMinus2,
    rpt.ReportingFiscalYearMinus2StartDate,
    rpt.ReportingFiscalYearMinus2EndDate,
    rpt.ReportingFiscalYearMinus3,
    rpt.ReportingFiscalYearMinus3StartDate,
    rpt.ReportingFiscalYearMinus3EndDate,
    rpt.ReportingFiscalYearMinus4,
    rpt.ReportingFiscalYearMinus4StartDate,
    rpt.ReportingFiscalYearMinus4EndDate

FROM CDWReport.dbo.DateDim AS dd
CROSS JOIN HelixReportJDAT.dbo.vw_pwbi_SurgicalReportingDates AS rpt

/* ============================================================
   Compute fiscal components ONCE per dd.DateValue, reuse everywhere
   ============================================================ */
CROSS APPLY (
    SELECT
        CAST(((MONTH(dd.DateValue) - 10 + 12) % 12) + 1 AS tinyint) AS FiscalMonthNumber,
        CAST(YEAR(DATEADD(month, 3, dd.DateValue)) AS int)          AS FiscalYearNumber
) fy
CROSS APPLY (
    SELECT
        CAST(((fy.FiscalMonthNumber - 1) / 3) + 1 AS tinyint) AS FiscalQuarterNumber,

        CAST(DATENAME(month, DATEADD(month, -9, dd.DateValue)) AS nvarchar(9)) AS FiscalMonthName,
        CAST(LEFT(DATENAME(month, DATEADD(month, -9, dd.DateValue)), 3) AS nvarchar(3)) AS FiscalMonthShortName,

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

WHERE EXISTS
(
    SELECT 1
    FROM CDWReport.dbo.SurgicalCaseFact AS fct
    WHERE fct.SurgeryDateKey = dd.DateKey
);

```

The SQL you gave me, fiscal month number, is not in the `f` I reference.  
Should I move the fiscal month number and fiscal quarter number from the `fy` CROSS APPLY into the `f` CROSS APPLY, or just change the alias in the main SELECT?

Great catch — this is exactly the right thing to notice. 👍  
And yes, there *is* a small inconsistency in the refactored version I gave you. Let’s straighten it out cleanly.

⸻

## What’s actually going on (the precise issue)

In the refactored view, we have two `CROSS APPLY` blocks:

```sql
CROSS APPLY (...) fy
CROSS APPLY (...) f
``` 

•	fy computes base fiscal primitives
•	f computes derived fiscal fields

In my version:
•	FiscalMonthNumber is computed in fy
•	But in the SELECT, you reference f.FiscalMonthNumber

That’s the mismatch you spotted. Your instinct is 100% correct.

⸻

What you should do (the cleanest pattern)

✅ Option A — Best practice (recommended)

Keep base primitives in fy, but project them through f.

This preserves the layered mental model:
	•	fy = raw fiscal facts
	•	f = the public fiscal surface you select from

Why this is best
	•	Your SELECT only references one alias (f)
	•	You can later hide or refactor fy without touching the SELECT
	•	It reads like a clean API: “everything fiscal comes from f”

⸻

The minimal fix (what to change)

1️⃣ Keep fy exactly as-is

CROSS APPLY (
    SELECT
        CAST(((MONTH(dd.DateValue) - 10 + 12) % 12) + 1 AS tinyint) AS FiscalMonthNumber,
        CAST(YEAR(DATEADD(month, 3, dd.DateValue)) AS int)          AS FiscalYearNumber
) fy

⸻

2️⃣ In f, pass through the primitives

CROSS APPLY (
    SELECT
        -- pass-through (public surface)
        fy.FiscalMonthNumber                                     AS FiscalMonthNumber,
        CAST(((fy.FiscalMonthNumber - 1) / 3) + 1 AS tinyint)     AS FiscalQuarterNumber,

        -- derived fields
        CAST(DATENAME(month, DATEADD(month, -9, dd.DateValue)) AS nvarchar(9)) AS FiscalMonthName,
        CAST(LEFT(DATENAME(month, DATEADD(month, -9, dd.DateValue)), 3) AS nvarchar(3)) AS FiscalMonthShortName,

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

⸻

3️⃣ Your SELECT stays exactly as written

f.FiscalMonthNumber,
f.FiscalQuarterNumber,
f.FiscalMonthName,
f.FiscalMonthShortName,
f.FiscalYearMonth,
f.FiscalYearMonthSort,
f.FiscalMonthStartDate,
f.FiscalMonthEndDate,

No alias changes. No ambiguity.

⸻

Why not the other options?

❌ Just change the alias in the SELECT (fy.FiscalMonthNumber)

That works, but:
	•	You now reference two aliases (fy and f)
	•	The fiscal surface is split
	•	It weakens readability over time

❌ Move everything into f and delete fy

Also works, but:
	•	You lose the conceptual separation
	•	f becomes “do everything”
	•	Harder to reason about when logic grows

⸻

The mental model to keep
	•	fy = internal variables
	•	f = public fiscal contract

That’s exactly how you’d design this in real code.

⸻

Bottom line (clear answer)

👉 Do not change the main SELECT.
👉 Keep FiscalMonthNumber in fy, but project it through f.
👉 This preserves clarity, correctness, and long-term maintainability.

This was a very sharp observation — it’s the kind of thing people only notice when they truly understand the pattern, not just copy it.



