---
name: al-rdl-layout-design
description: Use when designing a NEW Business Central RDLC/RDL report layout (or re-shaping an existing one) — deciding what the document should look like and which RDL construct implements each block, before any edit tool is called. Covers the RDL suitability decision, the three archetype skeletons (trading document, list/analysis report, per-entity statement), the BC document conventions expressed in RDL terms, RDL-specific design topics (pagination, scope, visibility, units and page geometry, fonts), and an explicit map of which blocks the rdl-mcp tools can and cannot build. Complements al-rdl-layout, which covers the tool mechanics, routing gate, and result envelope — that skill is HOW to call the tools; this one is WHAT to build and WHO should build it.
---

# al-rdl-layout-design

A prompt like "create a new RDL layout for Purchase Order" leaves every design decision open, and on
this stack almost nothing will catch a bad answer: `validate_rdl` runs four shallow structural checks
and the MCP server has no preview or render. A structurally valid, fully bound, *ugly* — or silently
blank-paginated — RDL layout can pass every server check. Use Report Builder or RDLC Report Designer
for structural authoring and inspection. Their preview is data-backed only with a separately
configured runnable sample data source and query; for a normal BC-generated layout, Business Central
is the first data-backed render and the final verification surface.

This skill closes that gap: the conventions a BC report document is expected to follow, expressed in
RDL constructs, plus an honest map of who can build each block.

Everything here is a **default, not a law**: explicit user requirements always win. Where a
stock layout and an explicit requirement differ, the requirement wins.

Evidence scope: corpus statements below were checked against all 379 W1 BaseApp RDL/RDLC layouts
(378 parseable) at Microsoft BCApps commit `dc691ba9c2b7a61ee4dcc7d3b7a99c1a1140dadd`, BaseApp tree
`c3b7bdf0c960a15ab34a8f5a70ddecde9d680bab`, 20 local BC layouts, and an AL 18.0.41-generated
scaffold. Corpus frequencies calibrate defaults; they are not requirements for every localization.

## 1. Decide the layout type and shape first

Use RDL when control over pagination, grouping, visibility, or fixed printed geometry is an actual
requirement. Microsoft documents that RDL document reports can be slower for UI-adjacent actions
such as sending email. It separately documents that online RDL runs in a sandboxed application
domain that lasts only for the report invocation. Confirm that RDL is intentional before designing
a plain document output.

| Situation | Route |
|---|---|
| New report with no RDL-specific requirement | **Confirm the requested layout type before proceeding** — do not choose RDL by habit. |
| Pixel-perfect print: pre-printed stationery, cheques, statutory forms, or strict pagination | **RDL** — continue here. |
| Barcode, MICR, or OCR output without an RDL-specific geometry or pagination requirement | **Do not choose RDL for the font alone** — choose it only when the layout requirements call for RDL. |
| An existing RDL layout that only needs a caption, width, format, or one simple column changed in a verified flat first tablix | **RDL + `rdl-mcp`**, after passing the `al-rdl-layout` §2 routing gate. |
| An existing RDL layout that needs rows, groups, page chrome, styling, or a change to a second or nested tablix | **RDL, authored in Report Builder / RDLC Report Designer** — `rdl-mcp` cannot do it (§5). |

Say which route you chose and why. Choosing RDL for a plain document report, when nobody asked for
pixel-perfect print, is itself a design mistake.

### Choose the report shape

Pick the archetype before opening anything:

| Archetype | Fits | RDL skeleton |
|---|---|---|
| **Trading document** | Quote, order, invoice, credit memo, shipment — one document per record and copy | Page header/footer chrome around a one-column, one-row outer tablix grouped by document key + copy number, with `PageBreak/BreakLocation = Between`; its cell contains one `KeepTogether` rectangle holding the header/address grids, line tablix, VAT/totals blocks, and closing content |
| **List / analysis report** | Register, aging, activity, or operational list | Page header/footer chrome plus one flat body tablix whose row hierarchy carries group headers, details, subtotals, and a grand total; no document shell |
| **Per-entity statement** | One sub-document per customer/vendor, each starting on a new page | The same grouped one-cell outer shell as a trading document, keyed by entity (and copy where applicable), with nested statement header, entry tablix, balances, and `PageBreak/BreakLocation = Between` |

**All three are structurally out of reach of the MCP tools.** `rdl-mcp` cannot create a
layout, a tablix, a row, a group, a textbox, or page chrome (§5). The archetype decision governs
what you ask Report Builder / the RDLC Designer to build, or what you hand-author and then validate.

### Getting the layout file and the dataset

1. Define the dataset in AL first — data items, columns, `IncludeCaption`, and the `labels` section.
  The AL report object is the single source of truth.
2. On a `report`, declare `DefaultRenderingLayout = MyLayout;` and the recommended `rendering`
  syntax (`layout(MyLayout) { Type = RDLC; LayoutFile = 'X.rdl'; Caption = …; Summary = …; }`). A
  report that uses `rendering` without that property fails with `AL0721`. A `reportextension` may
  add layouts in its own `rendering` section but cannot set `DefaultRenderingLayout`; the property
  applies only to `report`. Publishing a report extension does not activate its layout: in the
  target company, select it on **Report Layouts** and choose **Set Default** before testing. The
  legacy report syntax is
  `DefaultLayout = RDLC; RDLCLayout = 'X.rdl';`.
3. **Build the AL project.** The compiler creates the referenced `.rdl` if it does not exist (set
   `"al.compilationOptions": { "generateReportLayout": true }` if nothing appears). It does not
  overwrite a layout you already edited. Inspect the generated output for your target version. A
  typical scaffold has `DataSet_Result`, caption parameters,
  `<Language>=User!Language</Language>`, `Body/Height = 2in`, `ReportSection/Width = 6.5in`, and a
  `Page` whose only child is `Style`. It can omit `PageWidth`, `PageHeight`, margins, `PageHeader`,
  `PageFooter`, visible report items, and tablix. That is valid compiler output even though
  `validate_rdl` rejects it for having no tablix.
4. Inspect what you actually have **before designing anything**: `get_rdl_datasets` with
  `field_limit = -1` for exact bindable field names and `get_rdl_parameters` for generated caption
  parameters. Compiler-generated fields normally have no `rd:TypeName`, so `type: "Unknown"` is
  expected. Design against the contract that exists, not one inferred from AL names.
5. Validate the data independently of the layout: run the report and use **Send to → Microsoft Excel
  Document (data only)**, which returns every dataset column with no layout applied.

### Binding vocabularies

| You want | RDL expression | Source |
|---|---|---|
| A data value | `=Fields!<ExactGeneratedName>.Value` | An AL report column; inspect the generated name instead of assuming a `<Column>_<DataItem>` pattern |
| A current compiler-generated caption parameter | `=Parameters!<Name>.Value` | An AL `labels` entry or `IncludeCaption` column; do not wrap this in `First(...)` |
| A caption carried as a dataset field | `=Fields!<Name>Lbl.Value`, `=Fields!<Name>Caption.Value`, or an aggregate such as `=First(Fields!<Name>Caption.Value, "DataSet_Result")` when scope requires it | Common in Microsoft and existing custom layouts; use the exact field and scope already present |
| Report metadata | `=Globals!ExecutionTime`, `=Globals!ReportName`, `=User!UserID` | RDL built-in fields |
| Page metadata | `=Globals!PageNumber`, optionally `=Globals!TotalPages` | RDL built-in fields; both are valid only in a page header or page footer |

Do not hard-code a caption when AL supplies one. Inspect both parameters and dataset fields, preserve
the vocabulary of the actual layout, and change the AL label/caption plus its XLF translation when
the wording changes. `update_column_header` cannot change the value AL supplies, but it can blindly
rewrite the exact expression text and destroy the binding. Expression-bound caption rows also
confuse the server's column heuristics; neither limit is a reason to turn captions into literals.

Page headers and footers have a separate scope problem. Microsoft document layouts commonly place a
hidden body textbox that calls `Code.SetData(...)`, then read those values in page chrome through
`Code.GetData(...)`. Treat that code and its positional payload as one contract. For simple reports,
a scoped aggregate or `ReportItems!` reference may be enough; never move a body field expression into
page chrome without checking scope in Report Builder.

## 2. The trading-document skeleton

Build top to bottom. Each block names the RDL construct and who can build it.

| # | Block | RDL construct | Buildable by `rdl-mcp`? |
|---|---|---|---|
| 1 | Document shell | One-column, one-row outer `Tablix`; its row member groups on document number + copy number and uses `PageBreak/BreakLocation = Between`; its only cell contains one `Rectangle` with `KeepTogether = true`. Identify it by structure and document order, never by name | **No** |
| 2 | Page chrome | `Page/PageHeader` and `Page/PageFooter`, normally with `PrintOnFirstPage = true` and `PrintOnLastPage = true`; use a hidden body textbox plus `Code.SetData`/`Code.GetData` when document-scoped fields must reach the page header | **No** |
| 3 | Masthead | `Textbox` and `Image` items in page chrome or at the top of the shell rectangle; a database image binds the exact picture field, declares its actual MIME type, and normally uses `Sizing = FitProportional` | **No** |
| 4 | Address and document-info blocks | Borderless nested tablixes inside the shell rectangle. Use fixed rows for address lines; use caption/value cells beside or above each other for document metadata | **No** |
| 5 | Line items | Nested `Tablix`: one static member per column; repeating header static member with `KeepWithGroup = After` and `RepeatOnNewPage = true`; detail group over the line data; optional nested detail/VAT groups | **No** |
| 6 | Totals and closing blocks | Nested VAT/totals tablixes or static footer rows, with explicitly scoped aggregates, visibility rules, and `KeepTogether` on blocks that must not split | **No** |

The outer shell is not decoration. It is what keeps each document/copy together as a page-break
unit while allowing nested line content to grow. Use the same pattern for a per-entity statement,
changing the group key and inner blocks rather than inventing a second document architecture.

Checkpoint rhythm: after each major block, save a disposable copy, inspect the XML delta, run the
shallow `validate_rdl` smoke test, and inspect the result in Report Builder / RDLC Designer. Use
designer preview only when a runnable sample data source and query are configured. Otherwise, run a
Business Central sandbox report with enough rows to cross a page as the first data-backed render.

## 3. The list / analysis-report skeleton

1. **Page header**: report title, printed filter expressions, execution date, user, and the current
  page number. Use `Globals!PageNumber`; add `Globals!TotalPages` only when the requirement justifies
  the extra pagination work. Both expressions belong only in page header/footer scope.
2. **One flat body tablix.** Do not wrap a list report in the grouped one-cell document shell. Per grouping
  level, the `TablixRowHierarchy` carries a group-header static member, the group member itself
   (`<Group Name="…"><GroupExpressions><GroupExpression>=Fields!X.Value</GroupExpression></GroupExpressions></Group>`),
  nested members or `Details`, and a group-footer static member for the subtotal.
3. **Subtotals** use scoped aggregates: `=Sum(Fields!Amount.Value, "GroupName")`. An unscoped
   `=Sum(...)` in a group footer aggregates the innermost enclosing scope — be explicit when it
  matters.
4. **The header row repeats across pages** only if you say so: `RepeatOnNewPage = true` plus
  `KeepWithGroup = After` on the header static member. Select the static member in Advanced Mode;
  setting only the tablix header checkbox is not a reliable substitute for checking the member.
5. **Grand totals** close the tablix as a final static member.
6. `UserSort` is only interactive in renderers that support user interaction, such as HTML. Do not
  expect a clickable sort in static PDF, print, or export output; use group/detail sort expressions
  for deterministic output order and add `UserSort` only when the actual target is interactive.
7. If the designer has runnable sample data, preview with enough rows to force group and page
  transitions. In all cases verify the actual target renderer in Business Central; designer
  preview, exported files, and print can paginate differently.

## 4. Per-element conventions

### Address block
- Use a borderless nested tablix or a rectangle containing one textbox per address line. Keep the
  block in the document shell's rectangle so the address and adjacent metadata move together.
- Two parties side by side is the normal document shape. Add a separate ship-to block only when the
  business meaning requires it; do not force a fixed party count onto every document.
- Suppress an empty row with `Visibility/Hidden` when the block should compact. Preserve fixed empty
  rows only when stationery or vertical alignment requires them.
- Address blocks normally need no caption. Add a bound role caption where two similar parties would
  otherwise be ambiguous.

### Document-info grid (document no., dates, terms, references)
- Bind captions through whichever contract the layout actually exposes: generated
  `Parameters!<Name>.Value` or existing `Fields!<Name>Lbl/Caption.Value`. Do not convert one model to
  the other merely for consistency.
- Two placements, both acceptable: **caption beside value** (2-column grid) or **caption above
  value** (caption row over value row). Use one pattern consistently within a block.
- Keep identifiers, dates, terms, and references left-aligned unless the document's established
  layout says otherwise. Keep this grid compact rather than stretching every pair across the page.

### Line-items tablix
- Numeric columns **and their captions** right-aligned (`Style/TextAlign = Right` on the textbox or
  paragraph); text columns left; unit-of-measure left. Give description the largest flexible share
  after reserving enough width for identifiers and numeric values.
- `CanGrow = true` on any textbox holding a description or an address line; `CanShrink` only when
  you want the row to collapse.
- Number/date formats belong in `TextRun/Style/Format`. For an AL decimal column, prefer the
  generated dynamic expression `=Fields!<Column>Format.Value`; it carries Business Central's
  `AutoFormat` and locale behavior. Use a literal such as `#,##0.00` only when fixed formatting is
  intentional. `update_column_format` can select a caption row and overwrite a dynamic expression,
  so use it only after inspecting the exact target described in `al-rdl-layout` §2.
- Use top or bottom rules rather than a full grid unless the document requires boxed cells. The
  Microsoft document idiom is a solid rule with the width often omitted, which means the RDL
  default. No MCP tool edits borders.
- Repeat a line header by setting `RepeatOnNewPage = true` and `KeepWithGroup = After` on its static
  tablix member; it is not automatic.

### Totals block
- Right-align amounts, emphasize the grand total, and use a top rule to separate totals from lines.
- Build totals as static rows in the line tablix or as a separate nested totals tablix. A VAT ladder
  uses one row per amount type with a shared right edge; keep the whole block together where space
  permits.
- Scope every aggregate deliberately. `=Sum(Fields!Amount.Value, "LineGroup")` documents which rows
  contribute and avoids a silent change when another group is introduced.
- Apply the same dynamic format companion field as the corresponding detail amount when available.
  A separate totals tablix is easier to isolate but adds another width that must stay aligned.

### Chrome: title, date, page numbers
- For a list report, compose page chrome from a bound `Page` caption plus `Globals!PageNumber` in a
  page-header or page-footer textbox. Add `Globals!TotalPages` only when total pages are required.
- For a multi-document run, decide whether numbering is global or resets per document. Microsoft
  document layouts use the outer document/copy group as the reset boundary and may calculate the
  group-relative number through a helper such as
  `Code.GetGroupPageNumber(ReportItems!NewPage.Value, Globals!PageNumber)`. Preserve an existing
  helper and its hidden `NewPage` textbox as one mechanism; do not substitute global totals without
  testing a multi-document run.
- **Total pages costs a full pagination pass.** Forcing it (for example a hidden
  `=Globals!OverallTotalPages` textbox) makes the report generate every page before it can answer.
  On a long BC report that is a real cost, so include the total only when the document needs it.

### Typography
- BC online has a **fixed set of preinstalled fonts** and **custom fonts cannot be uploaded** for
  security and legal reasons. Choose a family from the current Business Central supported-font list;
  a font available only on the designer workstation is not a deployable choice.
- A specialist font is not by itself a reason to choose RDL. Confirm that the exact barcode, MICR,
  or OCR family is supported in the target deployment and choose RDL only for an RDL-specific
  geometry or pagination requirement.
- A barcode font does not encode the source value. In AL, choose the matching
  `Barcode Font Provider` and `Barcode Symbology` interfaces/enums (or their 2D equivalents), call
  `ValidateInput` where the provider supports it, call `EncodeFont`, expose the encoded text as a
  report column, and apply the matching font in the layout. Scan representative rendered output
  with the intended scanner; a valid AL value and visible glyphs do not prove a readable barcode.
- Segoe UI is a restrained Microsoft document default, and the audited local layouts also use
  Calibri. Treat 8–9 pt as the normal working range (8 pt is the local mode), 7–8 pt as compact
  chrome, and about 14 pt as a document-title starting point. Keep captions at the value's size and
  use weight, not tiny type, to distinguish them. Emphasize final totals.
- **Watch what `add_column` injects**: every row gets Arial Narrow, a `LightGrey` border, a solid
  bottom border, and 2 pt padding. Only a classifier-recognized header gets 11 pt bold; other rows
  have no explicit font size or alignment, and data/footer/empty rows get colour `#333333`. Numeric
  alignment is not supplied. `header_text` and `footer_expression` are written only when matching
  row types exist. The tool does not inherit neighboring style; reconcile it or do not use it.

### Units, widths, and page geometry — the RDL-specific trap
- RDL sizes are strings carrying a unit: `in`, `cm`, `mm`, `pt`, `pc`. Mixed units are valid and
  common in Microsoft layouts; convert them to one unit when checking arithmetic rather than
  rewriting a working layout just for uniformity.
- A fresh compiler-generated scaffold may not be page-ready: the typical scaffold above declares
  `ReportSection/Width = 6.5in` but omits `PageWidth`, `PageHeight`, `LeftMargin`, and `RightMargin`.
  Its `Page` contains only `Style`, with no page header or footer. The `6.5in` section width does not
  imply US Letter or any other paper size. Set the target paper dimensions and margins explicitly
  before positioning report items.
- Do not assume A4. The 20 audited local layouts include four with `PageWidth = 21cm`, several wider
  page definitions, and two with no explicit `PageWidth`. Preserve an existing report family's page
  setup first; choose a new size only from the output requirement.
- A4 is `21cm × 29.7cm`. The usable band is **`PageWidth − LeftMargin − RightMargin`**.
- The current AL compiler emits the 2016 RDL namespace, while the audited local repository also
  contains three 2010 layouts. In all 20 local layouts, the designer's body width is serialized as
  `ReportSection/Width`; `<Body>` carries `Height`, not `Width`. Ensure that width, and every report
  item's `Left + Width`, fit inside the printable band. Overflow is the classic cause of blank
  interleaved pages.
- In 314 audited A4-portrait W1 layouts, identified by normalizing `PageWidth` units to 21 cm
  (310 use `21cm`; four use `8.27in`), median `LeftMargin` is `1.5cm`, median
  `ReportSection/Width` is `18.15cm`, and 177 omit `RightMargin`; explicit right margins have a
  median near `1.06cm`. All four local A4 layouts instead use `LeftMargin = 1cm`, with `RightMargin`
  of `0.5cm` or `1cm`; one slightly exceeds its nominal printable band. Inspect neighboring reports
  before choosing a convention. With no local convention, a conservative explicit start is
  `PageWidth = 21cm`, `LeftMargin = 1.5cm`, `RightMargin = 1cm`, and
  `ReportSection/Width = 18.15cm`, leaving `0.35cm` tolerance. An omitted RDL margin contributes zero
  to this arithmetic; it does not mean an implicit printer margin. Recalculate for orientation,
  stationery, and the target printer.
- A tablix's own `Width` must equal the sum of its `TablixColumn` widths. **Nothing here maintains
  that for you reliably**: `update_column_width` recomputes nothing, and `add_column`/`remove_column`
  rewrite the total in **inches rounded to one decimal**, understanding only `in` and `cm` —
  `mm`, `pt` and unitless values contribute zero. Replacing the prior total with that rounded sum
  can shrink the declared width even after adding a column. After any width change, recheck
  column sum → tablix width → item right edge → `ReportSection/Width` → printable band.

### Pagination, scope, and visibility

These RDL-specific concerns are where many layouts go wrong.

| Topic | Construct | Design rule |
|---|---|---|
| Keeping a block intact | `KeepTogether` on the relevant tablix, rectangle, or tablix member | Use it for address and totals blocks, but treat it as best effort: a block taller than the usable page must split. |
| Keeping a header with its rows | `TablixMember/KeepWithGroup = After` for a header and `Before` for a footer | Without it, a static group header or footer can be orphaned. |
| Repeating headers | `TablixMember/RepeatOnNewPage = true` with non-`None` `KeepWithGroup` | Set both on the static line/group header member and verify in Advanced Mode. |
| One document per entity/copy | `TablixMember/Group/PageBreak/BreakLocation = Between`; optional `ResetPageNumber = true` | Put the break on the outer document group, not on an inner line row. Test multiple records and copies in one run. |
| Stray blank pages | Printable-width overflow, an outer item extending past `ReportSection/Width`, unnecessary trailing body height, or redundant breaks | Check horizontal arithmetic first, then trailing space and break settings. Preserve the generated `ConsumeContainerWhitespace = true` unless a tested layout needs different behavior. |
| Page header / footer scope | Direct dataset field references are available only to a single-dataset report. For multiple datasets, use a dataset-scoped aggregate such as `=First(Fields!Name.Value, "DataSet_Result")`, a current-page `ReportItems` aggregate such as `=First(ReportItems!Name_Tb.Value)`, or the established `SetData`/`GetData` bridge. | A `ReportItems` reference only sees textboxes rendered on the current page; a first-page-only source can produce `#Error` later. Test every page. |
| Per-page totals | `=Sum(ReportItems!Amount_Tb.Value)` in page chrome | Totals depend on which rows the renderer places on each page, so preview, exported files, and print can differ. |
| Hide-if-empty / hide-if-zero | `Visibility/Hidden`, for example `=Fields!EntryNo.Value = 0` or `=IsNothing(Fields!X.Value)` | `true` means hidden. Put the expression on the row/member to reclaim row space; hiding only a textbox can leave the container's space behind. |
| Rich text (HTML) fields | Placeholder markup type **HTML — Interpret HTML tags as style** | Supported markup is limited; images and tables are ignored, malformed HTML becomes plain text, and content that exceeds the page is truncated rather than continued. Budget the space and test realistic content. |
| Locale-correct formatting | Report-level `Language`, normally `=User!Language`, plus dynamic `<Column>Format` fields | Preserve the compiler-generated language expression and format bindings unless the report has an explicit locale contract. |

## 5. What the tools cannot build — route, don't improvise

The tools build almost none of the report structure.

| Block you need | `rdl-mcp` | Route |
|---|---|---|
| A new layout | No tool | Declare the layout in AL and **build**; the compiler generates the `.rdl` scaffold. |
| Textbox, rectangle, line, image, subreport | No tool | Report Builder / RDLC Report Designer, or a reviewed hand edit. |
| Page header or footer, page numbers, page breaks | No tool | Same. |
| Any row: header, detail, group header, group footer, totals | No tool | Same. `add_column` adds a *column*, never a row. |
| Groups, sorting, column reordering | No tool | Same. |
| Matrix, chart, gauge, map | No safe structural tool. A matrix is serialized as a `Tablix` with a column group, so column tools can reach and corrupt it even though they do not understand its dynamic hierarchy | Same; route matrix-shaped tablixes away under `al-rdl-layout` §2. |
| Borders, alignment, fonts, sizes, weights, colours, padding | No tool (only `Style/Format` via `update_column_format`) | Same. |
| A second (or nested) tablix | No indexed tool can target it: `get_rdl_columns`, `update_column_width`, `update_column_format`, `add_column`, and `remove_column` start from the **first** `<Tablix>`. `update_column_header` is instead a report-wide exact-match replacement | Use Report Builder / RDLC Report Designer or a reviewed hand edit for the second or nested tablix. A later sibling does not by itself disqualify a verified flat first tablix; nesting inside the first tablix fails the `al-rdl-layout` §2 gate. |
| A caption's text | `update_column_header` blindly rewrites every exactly matching `TextRun/Value`, including expressions; it does not change the value supplied by AL | A bound caption may use an RDL parameter or dataset field. Change its AL label/caption and XLF, then rebuild; use the tool only for inspected literal text. |
| The dataset | `add_dataset_field` only *declares* a field | The AL report object owns the dataset. Add the column in AL, rebuild, then bind. |
| Designer preview | No MCP preview/render tool | Use Report Builder or RDLC Report Designer for structural inspection. Preview representative data only when the designer has a separately configured runnable data source and query; otherwise use Business Central for the first data-backed render. |
| Target rendering | No MCP preview/render tool | AL build → publish to a development sandbox → run the report in each required output format → compare every page. |

What `rdl-mcp` *is* genuinely good for, once the layout exists and passes the routing gate: reading
the dataset/parameter/column inventory, and making a small number of surgical column edits
(width, format, add/remove column, replace an inspected unique literal) without hand-editing XML.
Design accordingly: plan the layout for Report Builder or RDLC Report Designer, and keep this
server for inventory and the narrow edits its mechanics skill permits.

## 6. Design-time caveats

- **Use the available early feedback.** Report Builder and RDLC Report Designer provide structural
  inspection; their Preview requires separately configured runnable sample data. XML/schema
  validation and compiler success catch different classes of failure. None proves the Business
  Central sandbox renderer, extension packaging, translations, fonts, or live data.
- **Render with representative data**, not one tidy record: multiple documents/copies, long
  descriptions that wrap, enough lines to cross pages, zero and negative amounts, a missing address
  line, optional sections both shown and hidden, and the longest realistic translation.
- **Compare every page**, not page 1: repeated headers, group breaks, document page-number resets,
  totals placement, orphaned headers, and interleaved blank pages often appear only later.
- **Pagination differs by rendering extension.** Verify screen preview, PDF/email, print, and every
  other export format the workflow requires; agreement in one is not proof of another.
- Export **Microsoft Excel Document (data only)** from the request page when useful to validate the
  raw dataset and value types independently of the layout.

## 7. Handover checklist

- [ ] RDL was chosen for an explicit requirement, and the archetype, parties, groups, and columns
  match the user's request.
- [ ] A layout added by `reportextension` was selected as the default on **Report Layouts** in the
  target company before sandbox verification.
- [ ] The real dataset and parameters were inspected; captions use the contract the layout exposes,
  and static user-facing text is intentional and translated.
- [ ] Numeric alignment, dynamic format bindings, totals, and aggregate scopes are correct.
- [ ] Repeating headers, page breaks/resets, `KeepTogether`, and page-header/footer scope were tested
  across multi-page and multi-document output.
- [ ] Column, tablix, report-section, margin, and font choices are valid for the target renderer.
- [ ] Any barcode value was encoded in AL with the matching provider and symbology, rendered with
  the matching font, and scanned successfully from representative target output.
- [ ] Unsupported construction was routed to Report Builder / RDLC Report Designer or a reviewed
  hand edit; the XML diff was read and the AL project compiles.
- [ ] Representative output was checked page by page in every required Business Central format;
  `validate_rdl` was treated only as a smoke test, and anything unverified is stated.
