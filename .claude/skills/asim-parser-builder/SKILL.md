---
name: asim-parser-builder
description: Creates Microsoft Sentinel ASIM parsers (KQL) from sample source log data, entirely offline. Use when asked to create, build, or write an ASIM parser, a vim/vimParser, or a normalization parser for a log source, without connecting to Azure or a Log Analytics workspace.
---

# ASIM Parser Builder (offline authoring only)

Adapted from Microsoft's `ASIMParserCreation-Agentic` tool
(https://github.com/Azure/Azure-Sentinel/tree/master/ASIM/tools/ASIMParserCreation-Agentic),
with every Azure-dependent step removed. This skill only produces local `.kql`
files from a sample of raw log data and the public ASIM schema documentation.
It never runs `az login`, never queries a Log Analytics workspace, never
deploys anything, and never opens a PR against Azure-Sentinel.

## Context to maintain throughout the workflow

- Event vendor and event product (used for file naming)
- Target ASIM schema name
- Parameter-less parser file path — `ASim<Schema><Vendor><Product>.kql`
- Parameterized parser file path — `vim<Schema><Vendor><Product>.kql`

## Step 1 — Gather requirements from the user

Ask for:

1. A short description of the log source (vendor/product), and a doc link if the user has one.
2. A representative **sample of raw log lines/events** — pasted text or a path to a file. Ask for a few different examples if possible, covering different event types/values, so field mapping and enum coverage are accurate.
3. The target ASIM schema, if the user already knows it (e.g. `NetworkSession`, `ProcessEvent`, `Authentication`, `DNS`, `WebSession`, `FileEvent`, `RegistryEvent`, `Dhcp`, `UserManagement`...). If unknown, infer it from the sample data and confirm with the user — fetch the schema list from https://learn.microsoft.com/en-us/azure/sentinel/normalization-about-schemas if needed.

Do **not** ask for a Log Analytics workspace ID, a Sentinel table name, or Azure credentials — none of that is needed for offline authoring, and this skill must never attempt Azure CLI or workspace access.

## Step 2 — Learn the target schema

Fetch these references (via WebFetch) before generating anything:

- Field requirement levels (Mandatory / Recommended / Optional) per schema:
  `https://raw.githubusercontent.com/Azure/Azure-Sentinel/refs/heads/master/ASIM/dev/ASimTester/ASimTester.csv`
- The schema-specific Microsoft Learn page for field meanings, types, and allowed enum values (linked from the schema list above).
- General parser development guidance (naming, parameter filtering pattern):
  `https://learn.microsoft.com/en-us/azure/sentinel/normalization-develop-parsers`

## Step 3 — Build the parameter-less parser

Parsers are KQL functions following the flow **Filter → Parse → Map**:

- Filter early on native source columns before parsing, for performance.
- Prefer high-performance parsing operators in this order: `split` > `parse-kv` > `parse_csv` > `parse`/`parse-where` > `extract_all` > `extract` > `parse_json` > `parse_xml`. Avoid regex-based parsing when a cheaper operator works.
- Normalize values with `iff`/`case`/lookup tables instead of copying raw source values.
- Use `project-rename` for direct renames, `extend` for calculated/normalized fields.
- Map as many fields as reasonably possible, including optional ones — this future-proofs the parser.
- Use `project` (not `project-away`) to drop unmapped source columns, so the parser doesn't silently expose new source columns as unnormalized extras.
- Include a `Type` column in the output (the source table/log type name).
- Include a `disabled: bool = false` parameter in the function signature, and filter with `| where not(disabled)` right after reading the source.
- If the parser uses `AdditionalFields`, also include a `pack: bool = false` parameter to let callers skip populating it for performance.
- Do **not** copy an existing upstream parser as a template — build the mapping from the actual sample data and the schema doc.

Save as `ASim<Schema><Vendor><Product>.kql` (strict naming convention, e.g. `ASimNetworkSessionCiscoASA.kql`).

## Step 4 — Build the parameterized parser

1. Copy the body of the parameter-less parser into a new file.
2. Add standard ASIM filtering parameters for the schema (check the schema doc / `normalization-develop-parsers#filtering-based-on-parser-parameters` for the conventional parameter names, e.g. `starttime`, `endtime`, `srcipaddr`, `eventtype_in`, etc.) as both function arguments and early `where` filters, so callers can narrow data before the parse step.

Save as `vim<Schema><Vendor><Product>.kql` (e.g. `vimNetworkSessionCiscoASA.kql`).

## Step 5 — Offline validation (replaces ASimSchemaTester / ASimDataTester)

There's no workspace to run the real Microsoft testers against, so review manually instead, using the CSV and schema doc from Step 2:

- **Error** — any Mandatory field for the schema is missing from the output, or has the wrong type/alias.
- **Warning** — a Recommended field is missing (only flag if the sample data actually contains that information and it wasn't mapped).
- **Info** — an output column isn't part of the schema (an "unnormalized extra" — reconsider whether it should be dropped via `project` or renamed into `AdditionalFields`).
- Check enumerated fields only take values from the schema's allowed set.
- Check IP, time, and GUID fields are cast to the correct type.
- Re-read the generated KQL for obvious syntax mistakes (unbalanced pipes, unclosed strings) since there's no live engine to catch them.

Iterate up to a few times against these checks before presenting the result.

## Step 6 — Report

Summarize for the user:

| Section | Details |
|---|---|
| Source column → ASIM field mappings | Table of source columns and the ASIM fields they were mapped to |
| Schema | Target ASIM schema name |
| Vendor / Product | Event vendor and event product |
| Files produced | Paths of the `ASim...kql` and `vim...kql` files |
| Open issues | Any Mandatory fields that couldn't be mapped from the sample data, and Warnings accepted |

## Explicitly out of scope

This skill never runs Azure CLI commands, never queries or deploys to a Log Analytics workspace, never generates ARM templates, and never opens a pull request against Azure-Sentinel. If the user wants any of that, tell them it's outside this skill and point them at Microsoft's original `ASIMParserCreation-Agentic` tool, which requires `az login` and a real workspace.
