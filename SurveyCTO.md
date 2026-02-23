# SurveyCTO: Key Concepts for Working with Survey Data

SurveyCTO is a platform for designing and collecting survey data, primarily on mobile devices.
Forms are defined in an XLSForm spreadsheet with (at minimum) two tabs: **survey** and **choices**.

---

## Form definition: the two tabs

### Survey tab
Each row defines one field. Key columns:

| Column | Purpose |
|--------|---------|
| `type` | Field type (e.g., `text`, `integer`, `select_one listname`, `begin repeat`) |
| `name` | Variable name — becomes the column header in exported data |
| `label` | Human-readable question text shown to enumerators |
| `relevant` | Expression controlling whether the field is shown (skip logic) |
| `constraint` | Validation expression; blocks form submission if false |
| `calculate` | Expression for `calculate`-type fields (see below) |
| `repeat_count` | For repeat groups: how many times to repeat (can reference a prior field) |

### Choices tab
Each row defines one choice option for a `select_one` or `select_multiple` field. Key columns:

| Column | Purpose |
|--------|---------|
| `list_name` | Links to the `select_one listname` / `select_multiple listname` type value |
| `name` | The **coded value** stored in exported data (e.g., `1`, `2`, `yes`) |
| `label` | Human-readable text shown to the enumerator |

**Important:** exported data contains the choice `name` (coded value), not the `label`.

---

## Field types

### `select_one listname`
Single-choice field (radio buttons by default). Exported as a single column containing the
chosen option's `name` value.

### `select_multiple listname`
Multi-choice field (checkboxes by default). Exports produce **two things**:
1. A space-separated string of all selected choice `name` values in one column
   (e.g., `"1 3 5"`)
2. A series of 0/1 dummy columns, one per possible choice, named `fieldname_choicename`
   (e.g., `crops_maize`, `crops_beans`, `crops_wheat`)

When working with a `select_multiple` variable, use the dummy columns for analysis; the
space-separated column is mainly useful for filtering/checking.

### `calculate`
Automatically computed field — never shown in the UI. The value is derived from an expression
referencing other fields (e.g., `${field_a} + ${field_b}`). Appears as a regular column in
exported data. Used for derived variables, running totals, formatted strings, etc.

### `note`
Display-only text shown to the enumerator; no data collected. Does not appear in exported data.

### `geopoint`
GPS coordinate capture. Exports as **four columns**: `fieldname-Latitude`,
`fieldname-Longitude`, `fieldname-Altitude`, `fieldname-Accuracy`.

---

## Repeat groups

Repeat groups allow a block of questions to be asked multiple times (e.g., once per household
member, once per business, once per crop). Defined in the survey tab with:

```
begin repeat   group_name   "Group label"
  ... fields ...
end repeat
```

The number of repetitions is either open-ended (enumerator decides) or fixed via `repeat_count`
referencing a prior integer field (e.g., `${num_businesses}`).

### Wide format export (default)
Each instance gets a numeric suffix appended to every field name:
- `age_1`, `age_2`, `age_3` … for a field named `age` in a repeat group
- Columns extend as far as the maximum number of instances observed across all submissions
- Respondents with fewer instances have blank values in the later columns

### Long format export (alternative)
Each repeat group exports as a **separate file/sheet**, with one row per instance. A `KEY`
column uniquely identifies each instance; a `PARENT_KEY` column links back to the parent
submission. Useful for reshaping or when instances are numerous.

---

## Relevance (skip logic)

The `relevant` column holds a boolean expression. A field is only shown when the expression
evaluates to `true`; otherwise it is hidden and skipped.

```
# Examples
${consent} = 1
${num_businesses} > 0 and ${country} = 'kenya'
selected(${activities}, 'farming')    # for select_multiple: checks if 'farming' was chosen
```

**Critical for data work:** when a field is not relevant (i.e., was skipped), it still appears
in the exported dataset — as a **blank/empty value**. Structurally missing data (due to skip
logic) is therefore indistinguishable from item non-response in the raw export. To correctly
interpret missingness, you must trace the relevance condition in the survey tab.

---

## Column naming conventions

When group names are enabled (the default in SurveyCTO Desktop), column names in the export
are prefixed with the names of all enclosing groups, separated by hyphens or underscores. For
example, a field named `profit` inside a group named `business` inside a group named `module2`
might export as `module2-business-profit` or `module2_business_profit`.

Repeat group suffixes (`_1`, `_2`, …) are appended **after** the full prefixed field name.

**Practical implication:** a variable named `group_subgroup_fieldname_3` in the data means:
- enclosing group: `group` (and `subgroup`)
- field name: `fieldname`
- third instance of the repeat group

---

## Missing data summary

| Cause | Appearance in export |
|-------|---------------------|
| Field not relevant (skip logic) | Blank / empty string |
| Item non-response | Blank / empty string |
| Field added in a later form version | Blank for earlier submissions |
| Repeat group instance doesn't exist | Blank in wide-format suffix columns |

All four look identical in the raw CSV. Use the survey tab relevance conditions and form
version history to distinguish them.

---

## Working with survey instruments

When reviewing a survey instrument, the two key files are:
- **survey.csv** — the survey tab: all fields, types, labels, relevance conditions, and
  calculate expressions
- **choices.csv** — the choices tab: all coded values and their human-readable labels

Together these allow you to:
1. Map any exported column name back to its question text and field type
2. Decode coded values (e.g., `1` → `"Yes"`, `2` → `"No"`)
3. Understand why a variable has missing values (trace the `relevant` condition)
4. Understand the structure of repeat groups and what each `_N` suffix represents