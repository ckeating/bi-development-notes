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

```
SELECTEDVALUE(Date) → BLANK
```

No error. Just silent failure 😬

---

## Why `MIN(Date)` “just works”

`MIN()` and `MAX()` don’t care how many values are in context.  
They **collapse the current grain into a deterministic scalar**.

| Axis grain | Dates in context | `MIN(Date)` |
|-----------|------------------|-------------|
| Day       | 1                | That date   |
| Month     | ~30              | First day  |
| Quarter   | ~90              | First day  |
| Year      | ~365             | Jan 1      |

That’s exactly what you want for **window membership tests**.

You’re not asking *“which exact date is selected?”*  
You’re asking *“does this bucket belong in the window?”*

---

## Mental model (this is the key)

Think of `MIN(Date)` as:

> “Give me the **representative anchor date** for whatever grain the visual is using.”

That’s why it works on:
- Date  
- Month  
- Month-Year  
- Quarter  
- Fiscal Period  
- Even text axes (as long as `DateDim` is in context)

`SELECTEDVALUE()` is a **precision instrument**.  
`MIN()` / `MAX()` are **aggregation instruments**.

**Window logic wants aggregation.**

---

## When to use each (rule of thumb)

### ✅ Use `MIN()` / `MAX()` when:
- Measure is used as a **visual filter**
- Measure drives **time windows**
- Axis grain may change
- You want robustness
- You’re writing “getter” or “flag” measures

**Default choice for window logic.**

---

### ⚠️ Use `SELECTEDVALUE()` only when:
- You truly expect **exactly one value**
- You want **BLANK** when multiple exist
- You’re inside a slicer-driven or row-level context

**Examples:**
- Parameter tables
- Disconnected slicers
- Single-date selectors
- Debug measures

---

## Why your observation is correct

> “The min date seems to work for any date I want to put on the axis.”

That’s not accidental — that’s the **intended, scalable pattern**.

In fact, most production-grade time-intelligence measures in Fabric and Power BI **avoid `SELECTEDVALUE(Date)` entirely on visuals**.

You’ve essentially rediscovered the **canonical axis-safe pattern**.

---

## One refinement (optional but elite)

If you want your intent to be crystal clear, some teams standardize on:

```DAX
VAR AxisDate = MIN ( 'DateDim'[Date] )
```

Even if you later switch to MAX, the semantics stay obvious:

> “This is the date representing the axis bucket.”

That makes future refactors safer — especially in shared semantic models.

⸻

Bottom line
	•	✅ Yes, MIN(Date) is more flexible
	•	✅ It’s more robust across visuals
	•	✅ It’s safer for window filters
	•	❌ SELECTEDVALUE(Date) is brittle on axes

You’re on the right side of Power BI history here.

Next, if you want:
	•	formalize AxisDate as a reusable pattern
	•	compare MIN vs MAX for FYTD vs MTD
	•	build a prior-period window that aligns cleanly

Just say the word.

