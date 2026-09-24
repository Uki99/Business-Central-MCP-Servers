---
name: al-rdl-layout
description: Use whenever inspecting, editing, or validating a Business Central RDLC/RDL report layout (.rdl / .rdlc) — BC RDL layouts, their datasets, report parameters, and tablix columns — through the rdl-mcp MCP tools, instead of hand-editing the RDL XML blind. Covers the inspect → route → edit → diff → verify workflow, the exact tool surface and parameters, the verified behavior of the published 0.1.0 build on BC-generated layouts (including the row-detection traps that silently discard your input), what it does and does not support, and the result envelope every tool returns.
---

# al-rdl-layout

The `rdl-mcp` MCP server gives a narrow set of read and column/dataset/parameter edit tools for RDL
files, so you stop hand-editing 2000+ lines of namespaced RDL XML for the handful of changes it
already covers.

There is no create, no preview, no render, no atomic write, no rollback, and `validate_rdl` checks
almost nothing. Treat the server as a *scalpel for simple column work on a layout you have already
read*, not as an authoring surface.

This skill is the **mechanics**: the workflow, tool surface, routing gate that decides where an edit
lands, and result/write behavior. Its sibling skill
`al-rdl-layout-design` is the **design**: what a BC RDL report should look like, which RDL construct
implements each block, and which of those constructs this server cannot build at all. When shaping a
layout, read that skill first and come back here for how to make the calls.

**Version scope:** this skill is verified against the published PyPI distribution `rdl-mcp 0.1.0`.
The server's MCP initialization value `1.0.0` is not the distribution version, and GitHub `main`
contains unreleased behavior. Pin the running MCP launcher to exactly `0.1.0`; do not use a floating
`latest`. A separate `uv run --with rdl-mcp ...` command checks a new temporary environment, not the
server process already connected to the client. Confirm the running server through its in-band tool
schemas: 15 tools; `get_rdl_datasets.field_limit` defaults to `0`; `add_column` requires
`column_index`, `header_text`, and `field_binding`; and `remove_column` has no
`auto_adjust_page_width`. If that surface differs, re-check the implementation before mutating.

**Evidence scope:** implementation claims below were verified against the published `0.1.0`
source and executable probes. Compiler behavior was checked with AL 18.0.41. Corpus observations
were checked on 2026-09-21 against all 379 W1 BaseApp layouts (378 parseable) at Microsoft BCApps
commit `dc691ba9c2b7a61ee4dcc7d3b7a99c1a1140dadd`, BaseApp tree
`c3b7bdf0c960a15ab34a8f5a70ddecde9d680bab`, and 20 layouts in an audited BC extension repository.
Frequencies calibrate risk; they are not requirements for every layout.

## 1. Intended workflow

The inner loop is thin and **not self-checking**. Nothing here previews, renders, or proves a
binding resolves.

```
inspect → route (is this layout in scope at all?) → ONE edit → read the XML diff → validate_rdl
                                                                       │
                                                                       ▼
        sandbox verify (OUTER — the only real sign-off):
        AL build → publish → select extension layout if applicable → run report → compare EVERY page
```

| Stage | Tools | Purpose |
|---|---|---|
| **Preserve** | *(outside this server)* use a disposable copy or confirm the original is recoverable from source control | No tool backs up, gates, or rolls back a write. This is the rollback mechanism. |
| **Inspect** | `describe_rdl_report`, `get_rdl_datasets`, `get_rdl_parameters`, `get_rdl_columns` | Orient: how many datasets/tablixes/columns, which fields and caption parameters exist. |
| **Route** | *(read the raw XML)* | Apply the §2 routing gate before any indexed column mutation. For `update_column_header`, inspect every exact match instead. |
| **Edit** | `update_column_header`, `update_column_width`, `update_column_format`, `add_column`, `remove_column`; `add_dataset_field`, `remove_dataset_field`, `add_parameter`, `update_parameter`, `update_stored_procedure` | One deterministic change at a time. |
| **Diff** | *(outside this server)* compare the edited copy with the original | Your only post-edit evidence inside the loop. Every write reserializes the whole file and can create formatting churn (§5), so use a whitespace-insensitive or XML-aware comparison and read the structure, not the line count. |
| **Validate** | `validate_rdl` | A shallow smoke test only — see §2. Passing proves the file still parses and still has a dataset and a tablix. Nothing more. |
| **Visual check** | *(outside this server)* Report Builder / RDLC Report Designer | Use the design surface for structural inspection. Preview is data-backed only when a runnable sample data source and query have been configured; a normal BC-generated dataset has an empty command. |
| **Sandbox verify (OUTER)** | *(outside this server)* AL build → publish → select layout → run the report | The **only** Business Central sign-off. For a layout added by `reportextension`, open the target company, select it on **Report Layouts**, and choose **Set Default** before running. Compare every output page: wrapping, widths, totals, group breaks, page breaks, overflow. |

### Where the layout, the dataset, and the captions come from

`rdl-mcp` cannot create any of them. In Business Central:

- **The AL report object owns the dataset.** Declare the layout on the report — the recommended
  syntax is `DefaultRenderingLayout = MyLayout;` plus a `rendering` section containing
  `layout(MyLayout) { Type = RDLC; LayoutFile = 'X.rdl'; Caption = …; Summary = …; }`. A `report`
  that uses `rendering` without `DefaultRenderingLayout` fails with `AL0721`. A `reportextension`
  may add layouts in its own `rendering` section but cannot set `DefaultRenderingLayout`; the
  property applies only to `report`. Publishing a report extension makes its layout available but
  does not activate it: in the target company, select the layout on **Report Layouts** and choose
  **Set Default** before sandbox verification. The legacy alternative is
  `DefaultLayout = RDLC; RDLCLayout = 'X.rdl';`. Then **build the AL project**. The compiler creates
  the referenced `.rdl` when it does not exist; it does not overwrite a layout you have already
  edited. If nothing is generated, set
  `"al.compilationOptions": { "generateReportLayout": true }` in settings.
- **The dataset surfaces as one RDL dataset**, conventionally named `DataSet_Result`, with one
  `Field` per AL report column. Bind the exact generated field name returned by
  `get_rdl_datasets`; do not infer a `<Column>_<DataItem>` name.
- **Current compiler-generated report labels and `IncludeCaption` captions are RDL report
  parameters**, normally referenced as `=Parameters!<Name>.Value`. Existing Microsoft and custom
  layouts also commonly expose captions as dataset fields, using expressions such as
  `=Fields!<Name>Caption.Value` or `=First(Fields!<Name>Caption.Value, "DataSet_Result")`.
  Inspect both `get_rdl_parameters` and `get_rdl_datasets` and preserve the vocabulary the actual
  layout uses. Expression-bound captions of either kind defeat this server's static-header
  heuristics — see §2.
- **Structural authoring** (textboxes, rectangles, images, groups, page header/footer, page breaks,
  matrices, charts) belongs to SQL Server Report Builder or the Visual Studio RDLC Report Designer.
  There is no tool here for any of it.
- **To validate the data independently of the layout**, run the report and use **Send to →
  Microsoft Excel Document (data only)**: it returns every dataset column with no layout applied.

A newly generated RDL is only a scaffold; inspect the actual output for your target version. A
typical scaffold has `DataSet_Result`, caption parameters,
`<Language>=User!Language</Language>`, `Body/Height = 2in`, `ReportSection/Width = 6.5in`, and a
`Page` whose only child is `Style`. It can omit `PageWidth`, `PageHeight`, margins, `PageHeader`,
`PageFooter`, tablix, and visible report items. The `6.5in` section width therefore does not define
a paper size; author the required page dimensions and margins explicitly. This is valid compiler
output even though this server's `validate_rdl` returns `valid: false` with
`No Tablix (table) found`. Decimal columns also receive a companion `<Column>Format` dataset field;
preserve bindings to those fields because they carry Business Central formatting.
Compiler-generated fields normally have no `rd:TypeName`, so `get_rdl_datasets` returning
`type: "Unknown"` is expected.

### Host-specific paths on Windows

Use absolute paths. Some Windows MCP hosts may pass a non-ASCII path mojibaked (`Telefónica` as
`TelefÃ³nica`), though this has not been reproduced consistently and is not established
`rdl-mcp 0.1.0` behavior. If an actual returned path or error shows that corruption, copy the layout
to an ASCII-only temporary path, operate on the copy, inspect the diff, and replace the original
through normal filesystem tooling. Do not retry the same damaged path.

## 2. Tool reference

All fifteen tools take an absolute `filepath`. Names below are the protocol names; a host may expose
them with a prefix.

### Routing gate for column tools

Read this before `get_rdl_columns`, `update_column_width`, `update_column_format`, `add_column`, or
`remove_column`. They all begin from the same target:

```python
tablix = root.find('.//{ns}Tablix')   # FIRST Tablix in document order
```

There is **no tablix name, index, or selector argument on any of them.** A BC document layout
routinely contains several body tablixes (header block, lines, totals, per-entity sub-tables). If
the one you mean is not the first in document order, `get_rdl_columns` describes the wrong tablix
and each mutator in that list targets the wrong one; a mutator can still report `success: true`.

#### Three different row heuristics

| Used by | Rule | On a BC layout |
|---|---|---|
| `_detect_row_type` — drives **`add_column`** | static text ≥ 50 % of cells → `header`; elif nonaggregate `=`-expressions ≥ 50 % → `data`; elif ≥ 1 aggregate → `footer`; else `empty`. "Aggregate" is a substring test for `Sum(`, `Count(`, `Avg(`, `Min(`, `Max(`, and `First(`. | A bare `=Parameters!…` or `=Fields!…Caption` caption row becomes **`data`**; a `=First(Fields!…Caption…)` row becomes **`footer`**. Neither is a header, so `header_text` is not written there. |
| `get_rdl_columns` | Across **all descendant rows** of the first tablix, header row = the row with the most static-text cells, requiring ≥ 50 % static; **fallback** = the first row whose first cell is non-`=` text. Data row = the first row where ≥ 50 % of cells start with `=` (including caption expressions). | An expression-bound caption row can become the **data** row, including one inside a nested tablix. In the audited W1 corpus, 370 of 378 parseable layouts returned `columns: []`; a nonempty result is the exception, not proof of a correct mapping. |
| `update_column_format` | first direct body row where ≥ 50 % of cells contain `Fields!` expressions | An expression-bound caption row commonly qualifies before the real detail row. On a spanned row, placeholder cells can prevent any row reaching 50 %. Never assume this selected the detail row. |

**Consequences on BC tablixes:**

- `get_rdl_columns` normally returns no columns on a Microsoft-style BC layout. When it does return
  columns, it can combine one descendant row's text and textbox names with another row's expressions;
  a reported `field_binding` may therefore be a caption parameter rather than a data field.
- `add_column(filepath="/absolute/path/Layout.rdl", column_index=-1,
  header_text="Credit Limit", field_binding="=Fields!X.Value",
  footer_expression="=Sum(Fields!X.Value)")`
  writes `header_text` only to rows classified as `header`, and `footer_expression` only to rows
  classified as `footer`. No direct first-tablix row in the audited W1 snapshot met the header rule;
  only 2 of 107 did in the audited extension repository. Treat both arguments as conditional and
  verify the actual row classifications. `format_string` is emitted only on generated `data` and
  `footer` cells, never on `header` or `empty` cells. Other rows receive the field binding or an
  empty value.
- `update_column_format` can return `success: true` after adding a numeric format to a caption
  textbox while leaving the amount textbox's dynamic `=Fields!<Column>Format.Value` unchanged. On
  another valid BC tablix, the same call can return `No data row found` because spans keep the
  field-expression count below 50 %.

`update_column_header` is the exception: it does not select a tablix. It blindly replaces every
exactly matching `TextRun/Value` in the report, including expression text.

#### Different descendant and direct search spaces

- `get_rdl_columns` and `update_column_width` resolve columns with `tablix.findall('.//TablixColumn')`
  — a **descendant** search, so columns of any **nested** tablix are folded into the same index
  space.
- `get_rdl_columns` also searches **descendant rows**, so its chosen header and data rows can come
  from different nested tablixes.
- `add_column`, `remove_column`, and `update_column_format` use direct body columns, rows, or cells.

On a layout with a nested tablix, `column_index` therefore means two different things depending on
which tool you call.

#### Gate

Use these tools only when you have confirmed, by reading the raw XML around the target tablix, that:

1. The tablix you want is the **first** `<Tablix>` in the file, **or** you are only calling
   read tools.
2. It has **no nested tablix** inside it.
3. You know the actual **row order** and the actual **cell order**, rather than trusting
   `get_rdl_columns`' mapping.
4. No direct body cell has `TablixCell/CellContents/ColSpan` or `RowSpan` — the indexed tools do not
  understand spans, and `add_column` / `remove_column` insert or remove the *n*-th `TablixCell`
  positionally.
5. `TablixColumnHierarchy` has no descendant `Group`. A matrix is serialized as a `Tablix` with a
  column group, so these tools can reach it but do not maintain its dynamic column hierarchy.
6. For `add_column`, at least one direct body row satisfies its ≥ 50 % static-text `header` rule;
  when supplying `footer_expression`, at least one direct row must also classify as `footer`. If a
  matching row type is absent, that argument is not written even though the call succeeds. When
  supplying `format_string`, expect it only on rows classified as `data` or `footer`.
7. For `update_column_format`, you have identified the exact row selected by its 50 % rule and the
  target's existing `Format`. Do not replace `=Fields!<Column>Format.Value` unless fixed formatting
  is explicitly required.

If any of those fail, route the change to Report Builder / RDLC Report Designer, or to a reviewed
hand edit of the XML, and say so. Read tools remain useful either way.

### Inspect (read-only)

| Tool | Key params (defaults) | Returns | Reach for it when… |
|---|---|---|---|
| `describe_rdl_report` | `filepath` | `report_summary` (`datasets`, `parameters`, `table_columns`), `datasets[]` (`name`, `command_type`, `command`, `field_count`), `filepath` | First call on any layout. `table_columns` counts `TablixColumn` **descendants of the first tablix only** — on a multi-tablix layout it is not the report's column count. On BC layouts `command_type` is usually `Unknown` and `command` is null/empty because BC supplies the data. |
| `get_rdl_datasets` | `filepath`, `field_limit`=`0`, `field_pattern`=`null` | Per dataset: `name`, `datasource`, `command_type`, `command_text`, `query_parameters[]`, `field_count`, and (when `field_limit ≠ 0`) `fields[]` (`name`, `data_field`, `type`) + `fields_truncated` | Finding the exact `Fields!` name to bind. **`field_limit` defaults to `0` = counts only** — pass `-1` for all, or a positive N. `field_count` remains the unfiltered dataset count even when `field_pattern` narrows `fields[]`. BC-generated fields normally return `type: "Unknown"`. The pattern is a case-insensitive regex; a malformed regex is swallowed and the full list is returned, so sanity-check the names. |
| `get_rdl_parameters` | `filepath` | `parameters[]`: `name`, `data_type`, `prompt`, `default_value`, `valid_values[]` (static values, or a `dataset_reference` entry) | Listing the caption parameters an AL build generated from report labels / `IncludeCaption`. On a BC layout this is your **caption inventory** — the RDL equivalent of asking which labels exist. |
| `get_rdl_columns` | `filepath` | `columns[]`: `index`, `header`, `width`, `textbox_name`, and optionally `field_binding`, `field_name`, `format`. `{error: "No Tablix (table) found in report"}` when there is no tablix | A hint about a simple first tablix. On expression-caption or nested BC layouts its row mapping is incomplete or wrong (routing gate above), so never use it alone as the source of truth for `column_index`. An expression header is reported as an extracted field name or `(Dynamic Header)`; an empty one as `(Empty)`. |

### Validate

| Tool | Key params | Returns | What it actually checks |
|---|---|---|---|
| `validate_rdl` | `filepath` | `{valid: true, message}` or `{valid: false, issues: [...]}` | Four shallow checks only: (1) XML parses; (2) at least one `DataSet` exists; (3) each dataset has a `Query` or at least one `Field`; (4) at least one `Tablix` exists. A nonexistent field binding, duplicate parameter, removed final column, and still-referenced removed field all return `valid: true`. Conversely, a pristine compiler-generated layout returns `valid: false` only because it has no tablix. It catches every exception, including file-not-found, and returns it in `issues`; unlike other tools, it does not surface that exception as JSON-RPC `-32000`. It does not check the RDL schema, expressions, references, names, scope, geometry, or rendering. |

### Column edits (mutating — pass the routing gate first)

| Tool | Key params (defaults) | Returns | Reach for it when… |
|---|---|---|---|
| `update_column_header` | `filepath`, `old_header`, `new_header` | `{success, message}` | Blind exact-match replacement over every `TextRun/Value` in the body, page chrome, and every tablix; it reports no match count. It can replace expression text and destroy a binding. It cannot change the value supplied by a bound AL caption: change that source in AL instead. Count exact occurrences and inspect each one before calling. |
| `update_column_width` | `filepath`, `column_index`, `new_width` (e.g. `"2.5in"`, `"3cm"`) | `{success, message}` with old and new width | Resizes one **descendant** column of the first tablix. The value is written verbatim and nothing else is recomputed: `Tablix/Width`, report-section width, page width, and sibling positions stay unchanged. **Pass a nonnegative index**: any Python-valid negative index targets from the end; a more-negative out-of-range index raises an unhandled exception. Reconcile all geometry afterwards. |
| `update_column_format` | `filepath`, `column_index`, `format_string` (e.g. `"#,##0.00"`, `"dd/MM/yyyy"`, `"C2"`) | `{success, message, details: {column_index, old_format, new_format}}` | Writes `TextRun/Style/Format` on the first descendant `TextRun` in the indexed cell of the first direct row meeting the 50 % `Fields!` rule; nested content can therefore be selected. It can target a caption row, fail to find a spanned detail row, or overwrite a dynamic BC format expression. **Pass a nonnegative index**: Python-valid negatives target from the end; a more-negative out-of-range value raises an unhandled exception. Totals rows are not updated. |
| `add_column` | `filepath`, `column_index` (0-based; **`-1` = append**), `header_text`, `field_binding` (e.g. `"=Fields!Amount.Value"`), `width`=`"1in"`, `format_string`=`null`, `footer_expression`=`null` | `{success, message, details}` | Adds a column only to a flat first tablix whose row classification has been verified. It does **not** add the field to the dataset. It inserts one cell per direct row, an empty column member, and a direct column. `header_text` is written only to a classifier-recognized header row and `footer_expression` only to a recognized footer row; either is silently omitted when that row type is absent. `format_string` is emitted only on generated data/footer cells. The response merely echoes `header_text`. Added text uses Arial Narrow; only a recognized header gets 11 pt bold, while other rows get no explicit font size or alignment. Data/footer/empty rows get `#333333`; every row gets a `LightGrey` border, solid bottom border, and 2 pt padding. The tool does not inherit neighboring style. |
| `remove_column` | `filepath`, `column_index` | `{success, message, details: {column_index, total_columns}}` | Removes the direct column, physical *n*-th cell from every direct row, and *n*-th direct column member. Spans are not understood. It rejects every negative index, leaves dataset fields and references elsewhere untouched, allows removal of the final column, and rewrites `Tablix/Width` in rounded inches. There is no `auto_adjust_page_width`; page and report-section width stay unchanged. |

### Dataset and parameter edits (mutating — rarely correct for BC)

In Business Central the AL report object is the single source of truth for the dataset and for
caption parameters. Editing them in the `.rdl` changes what the layout *expects*, never what BC
*supplies*. Use these tools only to make the layout match a dataset the AL side already produces, or
on a non-BC SSRS report.

| Tool | Key params | Returns | Notes |
|---|---|---|---|
| `add_dataset_field` | `filepath`, `dataset_name`, `field_name`, `data_field`, `type_name` (e.g. `"System.String"`, `"System.Int32"`, `"System.Decimal"`, `"System.DateTime"`) | `{success, message}` | Appends `<Field Name=…><DataField/><rd:TypeName/></Field>` and refuses a duplicate field name. For BC, only mirror a column the AL report already returns; declaring it here does not create data. Prefer generating a fresh scaffold from AL and reconciling the layout against that authoritative schema. |
| `remove_dataset_field` | `filepath`, `dataset_name`, `field_name` | `{success, message}` | Removes the `Field`. **Any textbox still bound to it is left behind** and `validate_rdl` will not notice. Search for `Fields!<name>` before removing. |
| `add_parameter` | `filepath`, `name`, `data_type`, `prompt` | `{success, message}` | Appends a `ReportParameter` and performs no duplicate-name or data-type validation; adding the same name twice succeeds twice. For BC this is almost always wrong: caption parameters come from AL `labels` and `IncludeCaption`. Add the label in AL and regenerate the scaffold instead. |
| `update_parameter` | `filepath`, `name`, `prompt`?, `default_value`? | `{success, message}` listing the changes, or `success: false` when neither optional argument is given | Updates `Prompt` only if that element already exists; it does not create a missing prompt. It creates the default-value structure when needed. The AL side remains authoritative for BC captions. |
| `update_stored_procedure` | `filepath`, `dataset_name`, `new_sproc` | `{success, message}` | Sets an existing `DataSet/Query/CommandText`. A BC-generated layout carries an empty command, so this can return `success: true`, but Business Central supplies the dataset from AL rather than executing this command. Do not use it on BC layouts. |

## 3. Supported matrix (0.1.0)

**Available after the applicable §2 routing checks:**
- Reading the report summary, datasets and fields, report parameters, and a first-tablix column sketch.
- Replacing a unique, inspected literal string (`update_column_header`). Microsoft-style BC layouts
  usually have no literal `TextRun` value to match; never pass an expression as the old value.
- Changing one column's width value (`update_column_width` — then fix `Tablix/Width` by hand).
- Changing a format only after the exact row and cell selected by `update_column_format` have been
  confirmed and fixed formatting is intentional.
- Adding or removing a column (`add_column` / `remove_column`) only after their extra gate conditions
  pass — accepting non-inherited styling on anything added.
- Declaring or removing a dataset `Field`, and adding or updating a `ReportParameter` — on a non-BC
  report, or to mirror something AL already supplies.
- A four-check structural smoke test (`validate_rdl`).

**Not supported — refuse and route:**
- Selecting a tablix by name, or directing an indexed column operation at anything other than the
  **first** `<Tablix>` in the file.
- Indexed or positional column operations when the **first target tablix** contains a nested
  tablix, direct body cells with `CellContents/ColSpan` or `RowSpan`, or a column `Group` that makes
  it matrix-shaped. A later sibling tablix does not disqualify a verified flat first target, but
  these tools cannot select that sibling.
- Row operations of any kind: no add/remove row, no group header or footer row, no group
  configuration, no sorting, no column reordering.
- Any non-tablix report item: textbox, rectangle, line, image, subreport, chart, gauge, or map.
- `PageHeader` / `PageFooter` structure, page numbers, page breaks, visibility expressions,
  `KeepTogether` / `KeepWithGroup` / `RepeatOnNewPage`.
- Styling beyond a format string: fonts, sizes, weights, alignment, borders, padding, colours,
  background fills.
- Creating a layout, or an expression-builder helper of any kind.

**Absent from this server — use another tool:**
- Preview, render, or page images. Use Report Builder / RDLC Report Designer for structural
  inspection. Use designer preview only with separately configured runnable sample data; otherwise
  Business Central is the first data-backed render and the final output check.
- Schema validation, expression validation, reference validation, or a merge dry run.
- Atomic writes, a structural safety gate, corruption rollback, or backups.
- Any BC-connected upload of a layout to a tenant.

## 4. Anti-patterns

- **Don't edit without a preserved original.** There is no rollback, no backup, and no atomic write.
  A disposable copy is the safest input to every mutator.
- **Don't call a `*_column` tool before passing the §2 routing gate.** Confirm the first tablix,
  nesting, spans, row/cell order, and the edit-specific row heuristic. A wrong target returns
  `success: true`.
- **Don't trust `get_rdl_columns`' index/header/binding mapping on a BC layout.** It is demonstrably
  built from three different rows. Read the XML for the mapping; use the tool for orientation.
- **Assume `add_column` can discard `header_text` or `footer_expression` until raw XML proves the
  required row type exists.** No audited W1 first-tablix row met its header rule, while 2 of 107
  direct rows in the audited extension repository did. The response's `details.header_text` only
  echoes the input; success does not prove either value exists in the file. Route the change when
  condition 6 of the gate fails.
- **Don't use `update_column_format` until the selected row is known.** It can format a caption
  instead of the detail cell, and it overwrites an existing dynamic format expression. Preserve
  `=Fields!<Column>Format.Value` unless a fixed format is explicitly required.
- **Don't read `valid: true` as "the change is correct".** 0.1.0's `validate_rdl` cannot see a
  broken field reference, a duplicate textbox name, a bad expression, or a layout wider than its
  page (§2).
- **Don't change a width and walk away.** `update_column_width` desyncs `Tablix/Width` from its
  columns; `add_column`/`remove_column` rewrite it in rounded inches. Reconcile the tablix total,
  `ReportSection/Width`, and `PageWidth − margins` by hand (§5).
- **Don't pass a binding expression to `update_column_header`.** It will rewrite that expression and
  can silently break the report. For a bound BC caption, change the AL label/caption and its XLF;
  reserve this tool for inspected literal text that occurs only where intended.
- **Don't use `update_stored_procedure`, `add_parameter`, or `add_dataset_field` to "add data" to a
  BC report.** The AL report object owns the dataset and the caption parameters. Change AL, rebuild,
  then bind in the layout.
- **Don't infer feature support from the GitHub README or `main` branch.** This skill follows the
  published `0.1.0` distribution described in the version scope above.
- **Don't treat a successful edit as visual proof.** This server has no preview. Report Builder or
  RDLC Designer can provide structural feedback, and a data-backed preview only when separately
  configured with runnable sample data. Only a BC sandbox render of representative data, compared
  page by page in the intended output format, closes the loop.
- **Don't reach for these tools at all when the change is structural.** Report Builder, the RDLC
  Report Designer, or a reviewed hand edit of the XML followed by `validate_rdl` and a sandbox render
  is the correct route for rows, groups, page chrome, styling, and anything outside the first tablix.

## 5. Error and write behavior

### Result envelope

Mutating tools return `{success: bool, message: str}`, sometimes with a `details` object. Read tools
return a bare data object (`{columns: [...]}`, `{datasets: [...]}`, `{parameters: [...]}`), or
`{error: "..."}` for `get_rdl_columns` with no tablix. `validate_rdl` returns
`{valid, message}` / `{valid, issues[]}`.

There is **no error code and no hint field**. A `success: false` is a plain "not found" or "index out
of range" and leaves the file untouched. Except in `validate_rdl`, an unhandled exception surfaces as
a JSON-RPC error `-32000` carrying the Python message. `validate_rdl` catches every exception and
returns `{valid: false, issues: [...]}` instead.

**`success: true` proves only that XML was written to disk.** It does not mean the RDL is schema-valid,
that the binding resolves, that the change landed where intended, or that the report still renders.

### Whole-file rewrites

Each mutation re-parses, re-indents, and rewrites the entire file. Consequences:

- The writer strips a UTF-8 BOM, rewrites the XML declaration with single quotes, re-indents with
  two spaces, and uses the host line-ending convention (CRLF on Windows). When the source already
  matches those conventions, the structural diff can stay small: a local probe changed only 2 of
  804 line positions. Differently formatted input can still produce whole-file churn, so use a
  whitespace-insensitive or XML-aware diff and inspect the structure rather than the line count.
- There is no backup, atomic replacement, or rollback. Preserve the original before every mutation.
- A double UTF-8 BOM is not normalized on read: XML parsing fails before any tool can inspect or edit
  the layout. `validate_rdl` reports the parse issue; other calls surface JSON-RPC `-32000`. This
  occurred in the broader W1 corpus but not in the 20 local layouts, so parse failure is not proof
  the file is non-BC.

### Geometry bookkeeping

- `add_column` and `remove_column` recompute an existing `Tablix/Width` from direct columns, emit
  inches rounded to one decimal, and understand only `in` and `cm`; `mm`, `pt`, and unitless widths
  contribute zero. If the tablix has no `Width` element, the tools do not create one. Because the
  existing width is replaced by the rounded sum, it can even shrink after an add when the prior
  width differs from the direct-column sum; a local probe changed `5in` to `4.8in` after adding a
  `2cm` column. Both audited corpora commonly use `cm`; missing `Width` occurred only in the broader
  corpus.
- `update_column_width` recomputes nothing.
- No tool updates page margins, `ReportSection/Width`, row heights, or sibling positions. After a
  width change, reconcile the column sum, tablix width, report-section width, and printable page
  width. Overflow commonly creates blank interleaved pages.
