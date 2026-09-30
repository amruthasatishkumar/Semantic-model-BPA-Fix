# Apply Best Practice Analyzer fixes with Semantic Link Labs

A reusable Microsoft Fabric notebook that runs the **Best Practice Analyzer (BPA)** on a semantic
model, turns the findings into an **editable plan**, lets you remove anything you do not want
changed, and then **applies the approved fixes** back to the model, no Tabular Editor required.

This is built to hand to customers: they run it against their own model, review the proposed
changes, trim the list, and apply.

## What is in this folder

| File | Purpose |
|------|---------|
| [Apply_BPA_Fixes.ipynb](./Apply_BPA_Fixes.ipynb) | The notebook. Import it into a Fabric workspace and run it top to bottom. |
| [README.md](./README.md) | This guide. |

## Prerequisites

- A **Microsoft Fabric** workspace and the ability to create/import a notebook.
- **Write / edit** permission on the target semantic model (workspace Member or Admin, or Write on the model).
- [Semantic Link Labs](https://github.com/microsoft/semantic-link-labs) (`sempy_labs`). It is usually
  pre-installed in Fabric; if not, the first code cell installs it with `%pip install semantic-link-labs`.
- Optional: a **Lakehouse attached** to the notebook, only if you want the Excel/CSV review round-trip.

## How to use it

1. **Import the notebook** into your Fabric workspace (Workspace > New > Import notebook) and open it.
2. **Set two values** in the first code cell:
   - `dataset` = your semantic model name (or ID), e.g. `"Sudsberry Property Assessment"`.
   - `workspace` = workspace name/ID, or leave `None` to use the notebook's own workspace.
3. **Run Step 1** to see the BPA scorecard and the findings as a table.
4. **Run Step 2** to build the fix plan. Each finding gets:
   - `Auto-Fixable` - whether this notebook knows how to fix it automatically.
   - `Proposed Action` - exactly what the fix would do.
   - `Apply` - defaults to `True` for auto-fixable findings.
5. **Review and trim (Step 3).** Turn off anything you want to keep:
   - *Option A (in-notebook):* set `rows_to_skip = [0, 4, 7]`, or filter by rule/category/object.
   - *Option B (Excel):* export to `Files/bpa_fix_plan.csv`, edit the `Apply` column, reload.
6. **Apply (Step 4).** The cell runs as a **dry run** first and prints what it would change.
   When you are happy, set `DRY_RUN = False` and run it again to write the changes.
7. **Verify (Step 5).** Re-run the BPA. Fixed findings should be gone.

## Rules that are auto-fixed out of the box

| BPA rule | Automated action |
|----------|------------------|
| Whole numbers should be formatted with thousands separators and no decimals | Set format string `#,0` |
| Percentages should be formatted with thousands separators and decimals | Set format string `#,0.00%` |
| Provide format string for 'Date' columns | Set format string `mm/dd/yyyy` |
| Relationship columns should be of integer data type | Change both relationship columns to `Int64` |
| Hide foreign keys | Hide the column |
| Set IsAvailableInMdx to false on non-attribute columns | Set `IsAvailableInMdx = False` |
| Mark primary keys | Mark the column as a key column |

Every other BPA finding is labelled **"Manual review"** and is never touched by the notebook.

## Findings deliberately left for manual review

Some rules are intentionally **not** auto-fixed. Seeing them still flagged after an apply is expected,
it is not a failure. There are two reasons a rule is left manual:

**1. It needs human judgment** (could be automated, but shouldn't be without a decision):

| Rule | Why it's manual | How to fix it yourself |
|------|-----------------|------------------------|
| Model should have a date table | A person must pick the correct date table and date column | Mark it, e.g. `tom.mark_as_date_table(table_name="Valuation Calendar", column_name="Date")`, or use Model view > mark as date table |

**2. It's not a semantic-model change at all** (nothing to edit via the model):

| Rule | Why it's manual | How to fix it yourself |
|------|-----------------|------------------------|
| Ensure Direct Lake source tables are V-Ordered | V-Order is a physical property of the **Delta tables in your Lakehouse**, not a model setting, so there is no model property to change | Re-write / `OPTIMIZE` the source tables with V-Order enabled in the Lakehouse (e.g. Spark `OPTIMIZE <table> VORDER`, or `sempy_labs` lakehouse optimize with V-Order) |

If you ever decide a judgment rule is safe to automate for *your* models, add it to `REMEDIATIONS`
(see below). The V-Order type of rule cannot be added here because it is a data-engineering
operation, not a model edit.

## Extending it to more rules

The mapping lives in the `REMEDIATIONS` dictionary in Step 2. Each entry is a rule name plus a
`fix(tom, row)` function that uses the Semantic Link Labs TOM helpers. Example:

```python
def _fix_summarize_none(tom, row):
    t, c = _parse_column(row["Object Name"])
    tom.set_summarize_by(table_name=t, column_name=c, value="None")
    return "SummarizeBy set to None"

REMEDIATIONS["Set SummarizeBy property to 'none' for non-key numeric columns"] = {
    "action": "Set SummarizeBy = None",
    "fix": _fix_summarize_none,
}
```

Useful TOM helpers for writing new fixes: `tom.update_measure`, `tom.update_column`,
`tom.set_data_type`, `tom.set_summarize_by`, `tom.set_sort_by_column`,
`tom.set_is_available_in_mdx`, `tom.mark_as_date_table`, `tom.mark_primary_keys`.

## Safety notes

- **Dry run by default.** Nothing is written until you set `DRY_RUN = False`.
- **Only registered rules are changed.** Findings without a fix are left alone.
- **Int64 changes are metadata only.** For Import models the next refresh fails if the source values
  are not whole numbers; for Direct Lake the source column must already be integer. Such cases are
  reported per-row as `Failed` and do not block the other fixes.
- **Format strings are cosmetic.** Adjust the strings in `REMEDIATIONS` for your locale if needed.
- Test on a **copy of the model** first if you are cautious, then run against production.

## Reference

- Semantic Link Labs: https://github.com/microsoft/semantic-link-labs
- BPA rules: https://github.com/microsoft/Analysis-Services/tree/master/BestPracticeRules
