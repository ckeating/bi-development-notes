
Perfect — I’ll give you clean, copy-paste-ready SQL patterns you can drop straight into your DateDim build or view, aligned with everything you’ve already decided (DirectQuery, governance, fiscal clarity).

---

## Assumptions
- Fiscal year starts **October 1**
- You already have a base **`[Date]`** column of type `date`

## What this covers
1. Fiscal month number  
2. Fiscal month name / short name  
3. Fiscal year  
4. Fiscal quarter  
5. Recommended sort helpers  

---

## 1️⃣ Fiscal Month Number (Oct = 1 … Sep = 12)

This is the foundation.

**FiscalMonthNumber**
```sql
((MONTH([Date]) - 10 + 12) % 12) + 1