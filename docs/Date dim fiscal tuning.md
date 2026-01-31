## fiscal tuning

You’re in a good place: your current fiscal columns already exist (`FiscalYear`, `FiscalMonth`, `FiscalQuarter`) and you also have a separate **as-of / reporting window** system (`ReportingFiscalYear…`, `LastClosedMonth…`).  
So the goal is **consistency without breaking measures**.

## My recommendation in one line

**Don’t rename existing columns that are already referenced by measures/reports.**  
Instead, **add a clean, consistent “Fiscal” layer** (new columns), and optionally rename later with a controlled deprecation plan.

---

## What I would change (minimal-break, maximum-clarity)

### ✅ Keep as-is (do not rename)

These are common, likely referenced, and already readable:

- `FiscalYear`
- `FiscalMonth`
- `FiscalQuarter`
- `ReportingFiscalYear` and all `ReportingFiscalYearMinusN*`
- `LastClosedMonthStartDate*` / `LastClosedMonthEndDate*`
- `CurrentFiscalYear*`

**Why keep them:** renaming will break DAX measures, report visuals, calculated columns, and any downstream SQL/views.

---

## What I would add (clean fiscal naming aligned to the new pattern)

You currently have:
- `FiscalMonth` (number)
- `FiscalQuarter` (number)

But you don’t have the **explicit-intent names and sort helpers** from the new pattern.

### Add these new physical columns

**Fiscal clarity (new)**

- `FiscalMonthNumber` (alias of your current `FiscalMonth`)
- `FiscalQuarterNumber` (alias of your current `FiscalQuarter`)
- `FiscalMonthName`
- `FiscalMonthShortName`
- `FiscalYearMonth`
- `FiscalYearMonthSort`
- `FiscalMonthStartDate`
- `FiscalMonthEndDate`

This gives you a **consistent, Copilot-friendly, thin-report-friendly** set without breaking anything.

---

## Columns I *do* recommend refactoring (because they’re risky / unclear)

### 1) The `YMFiscalYear_*` family (rename or quarantine)

These will confuse everyone (including you in 6 months):

- `YMFiscalYear_X`
- `YMFiscalYearBeginDate_X`
- `YMFiscalYearEndDate_X`
- `YMFiscalMonth_X`
- `YMFiscalQuarter_X`

**Issue:** the `_X` suffix and `YM` prefix don’t convey meaning, and these look like *helper artifacts* that no report author should touch.

**Best practice options:**

- **Option A (safest):** keep names but hide them in the semantic model (mark as *internal/helper*).
- **Option B (cleaner):** create clean equivalents and stop using these:
  - `FiscalYearMonthSort` (replaces most `YM*` needs)
  - `FiscalMonthStartDate`, `FiscalMonthEndDate`
  - `FiscalYearStartDate`, `FiscalYearEndDate`  
    (you already have `Year*` for calendar and `CurrentFiscalYearStartDate` etc.)

If you know these are not referenced anywhere, you can rename them — **but only after impact checking**.

---

## A few naming improvements (optional, but helps long-term)

These aren’t *wrong*, but you’ll thank yourself if you standardize over time.

### Potential duplicates / ambiguous intent

- `Year` vs `IsoWeekYear` vs `IsoWeekYear` (nvarchar)  
  - `IsoWeekYear` being `nvarchar` is odd; if it’s a label, consider renaming later to `IsoWeekYearLabel` (optional, low priority).
- `YearMonth`, `MonthYear`, `YearFormattedMonth`, `FormattedMonthYear`  
  - You have **four+ variants of essentially the same concept**, which increases confusion and report inconsistency.

**Suggested canonical choices:**

- **Calendar**
  - `YearMonth`
  - `YearMonthSort`
- **Fiscal**
  - `FiscalYearMonth`
  - `FiscalYearMonthSort`

Hide or deprecate the others gradually.

---

## If you want to rename existing fiscal columns later  
*(the “eventual cleanup” plan)*

If you prefer the explicit style (`FiscalMonthNumber`, `FiscalQuarterNumber`) long-term:

1. **Add the new columns first** (aliases):
   - `FiscalMonthNumber AS FiscalMonth`
   - `FiscalQuarterNumber AS FiscalQuarter`
2. Update **new** reports/measures to use the new names.
3. After adoption, **rename or drop old ones** in a major version bump.

This avoids the classic *“one rename breaks 17 thin reports”* problem.

---

## Summary: what you should do right now

### Do **not** rename
- `FiscalYear`
- `FiscalMonth`
- `FiscalQuarter`
- `ReportingFiscalYear*`
- `LastClosedMonth*`

### Add (recommended)
- `FiscalMonthNumber`
- `FiscalQuarterNumber`
- `FiscalMonthName`
- `FiscalMonthShortName`
- `FiscalYearMonth`
- `FiscalYearMonthSort`
- `FiscalMonthStartDate`
- `FiscalMonthEndDate`

### Quarantine / hide (recommended)
- `YMFiscalYear_X` and related columns  
  *(or replace with the new canonical fields)*

---

If you want, paste your current **SELECT** (or view definition) and I’ll rewrite **just the fiscal section** as a clean block that:

- keeps your existing columns intact  
- adds the new fiscal fields  
- avoids duplicating the same logic five times  
- uses consistent naming for both calendar and fiscal labels/sorts  
