# Document Generation — Skill Reference

Use this skill when creating, modifying, or troubleshooting Salesforce Document Generation
templates and OmniDataTransform (ODT) metadata for the `maintwo` org (API v67.0).

## Entry Conditions

| Task | Use this skill? |
|------|-----------------|
| Create or modify DOCX templates (token layout, repeating sections) | Yes |
| Author or edit Extract/Transform ODT XML (`.rpt-meta.xml`) | Yes |
| Troubleshoot blank tokens, duplicate rows, or cartesian product issues | Yes |
| Add new tokens to an existing template pipeline | Yes |
| Choose an architecture pattern for a new template | Yes |
| Deploy ODTs or templates to `maintwo` | Yes |

---

## Quick Rules

1. **Token naming**: Flat (`{{Token}}`), Group-level (`{{Group:Token}}`), Line-level (`{{Group:Line:Token}}`), CQL child (`{{Line:CQL:Token}}`), Conditional field (`{{IF_IsChild:Token}}`), Conditional section (`{{#IF_HasGroups}}`)
2. **Extract structure**: Object queries use `outputCreationSequence: 0.0`; field mappings use `1.0`
3. **Transform anti-pattern**: NEVER add individual field mappings alongside a pass-through for the same nested path (causes cartesian product)
4. **Deployment order**: ODTs first -> activate -> template binary (never bundle together)
5. **GlobalKey prefix**: Each ODT uses a unique UUID prefix (see GlobalKey Convention)
6. **Python build scripts** generate the `.docx` binary -- never edit the `.dt` file directly
7. **ODT naming**: Alphanumeric camelCase only (no underscores/spaces)
8. **Colon paths only**: Never use dot notation in `InputFieldName` or `OutputFieldName`
9. **ORDER BY**: Must be a separate `omniDataTransformItem` entry with `FilterOperator="ORDER BY"`
10. **OutputObjectName**: Required on every item -- null causes silent NPE
11. **Currency format**: All currency fields must use `CurrencyRounded` as `outputFieldFormat` (not `Currency`)
12. **FilterGroup**: Always use `0` on nested child sequences (multiple groups cause N*M*G explosion)

---

## DO NOT

| Anti-Pattern | Consequence | Correct Approach |
|--------------|-------------|------------------|
| Add individual field mappings alongside a pass-through for the same nested path | Cartesian product -- duplicate rows multiply (N*M) | Use ONLY pass-through + formulas for nested data |
| Use dot notation (`Parent.Child.Field`) in Extract paths | Breaks SOQL generation silently | Use colon paths: `Parent:Child:Field` |
| Deploy ODTs and templates in the same manifest/batch | Platform GACK error | Deploy ODTs first, activate, then deploy template binary |
| Forget ORDER BY as a separate `omniDataTransformItem` | Results come in random order | Add separate item: `FilterOperator="ORDER BY"`, `FilterValue="FieldName"` |
| Mix CQL and IF_IsChild patterns in the same template | Conflicting nesting structures | Choose one pattern per template |
| Edit `.dt` binary directly | Corrupts template | Edit Python build script, then rebuild |
| Leave `OutputObjectName` null on any item | Item silently dropped at runtime | Always set to `json` (pass-through) or `Formula` (formula items) |
| Use multiple FilterGroups on nested child sequences | N*M*G cartesian explosion | Use single FilterGroup `0` on all nested sequences |
| Set `InputFieldName="Id"` on child-of joins | Wrong join direction -- queries fail or return wrong data | Use child's FK field (e.g., `QuoteId`, `QuoteLineGroupId`) |
| Use `Currency` instead of `CurrencyRounded` for money fields | Inconsistent formatting | Always use `CurrencyRounded` for `$` + commas + 2 decimal places |
| Manually author `FormulaConverted` (RPN) | Error-prone, auto-generated on save | Set `FormulaExpression` only; platform generates RPN |

---

## Extract Engine Reference

### Object Query Item Fields

| Field | Purpose | Example |
|-------|---------|---------|
| `InputObjectName` | SObject to query | `QuoteLineItem` |
| `InputFieldName` | Field on target to match against FilterValue | `QuoteLineGroupId` (child-of) or `Id` (FK lookup/root) |
| `FilterValue` | Value or path to match | `Quote:QLG:Id` (parent ref) or `Id` (root input) |
| `FilterOperator` | Match operator | `=`, `ORDER BY` |
| `FilterGroup` | AND grouping (always `0.0` in this project) | `0.0` |
| `OutputFieldName` | Internal hierarchy path | `Quote:QLG:Line` |
| `OutputObjectName` | Always `json` | `json` |
| `InputObjectQuerySequence` | Execution order | `1.0`, `2.0`, `3.0`... |
| `OutputCreationSequence` | Always `0.0` for object queries | `0.0` |

### Join Patterns

| Pattern | InputFieldName | FilterValue | Generated WHERE | Use Case |
|---------|---------------|-------------|-----------------|----------|
| **Root** | `Id` | `Id` | `WHERE Id = '<inputId>'` | First object (Quote) |
| **FK Lookup** (many:1) | `Id` | `Parent:FKField` | `WHERE Id = '<parentFKValue>'` | Account by `Quote:AccountId` |
| **Child-of** (1:many) | Child's FK field | `Parent:Id` | `WHERE ChildFK = '<parentId>'` | QLI by `Quote:QLG:Id` |
| **ORDER BY** | *(omitted)* | `FieldName` | `ORDER BY FieldName` | Separate item, same `InputObjectQuerySequence` |

**Critical rule**: `InputFieldName: "Id"` is correct ONLY for FK lookups. For child-of joins, use the child's FK field name (e.g., `QuoteId`, `QuoteLineGroupId`, `QuoteLineItemId`).

### Field Mapping Items

| Field | Value | Notes |
|-------|-------|-------|
| `OutputCreationSequence` | `1.0` | Always 1.0 for field extracts |
| `InputFieldName` | Colon-separated source path | `Quote:QLG:Line:RLM_ProductName__c` |
| `OutputFieldName` | Colon-separated output key | `Group:Line:ProductName` |
| `OutputObjectName` | `json` | Required -- null causes silent drop |

### Internal vs Output Paths

The Extract engine maintains two independent path systems:

1. **Internal hierarchy** (object query `OutputFieldName`) -- defines join scope and filter resolution
2. **Output structure** (field mapping `OutputFieldName`) -- defines the JSON shape

These are decoupled. The internal hierarchy determines WHICH records are queried. The field mapping output paths determine WHERE the data lands in the final JSON. They share colon notation but are independent namespaces.

### Extract Formula Items

Used in Pattern C (IF_IsChild) to compute boolean flags at Extract time:

| Field | Value |
|-------|-------|
| `OutputCreationSequence` | `0.0` |
| `OutputFieldName` | `Formula` |
| `OutputObjectName` | `Formula` |
| `FormulaExpression` | `ISNOTBLANK(Quote:QuoteLineGroup:QLI:ParentQuoteLineItemId)` |
| `FormulaResultPath` | `Quote:QuoteLineGroup:QLI:IsChild` |
| `FormulaSequence` | Must run AFTER the object query populates data (e.g., `3.0` if QLI query is seq `3.0`) |

### Hierarchy Depth Rule

ALL field mappings that output to the same array path MUST read from the same depth in the internal hierarchy. Mixing parent-level and child-level `InputFieldName` paths targeting the same output array causes phantom entries.

**Safe**: FK lookups (many:1) at mixed depths -- the engine resolves parent fields down.
**Dangerous**: Child-of (1:many) at mixed depths -- causes N+M phantom expansion.

---

## Transform Engine Reference

### Execution Order (OutputCreationSequence)

| OCS | Phase | What runs | Can reference |
|-----|-------|-----------|---------------|
| 0 | Formula phase | All formula items (by `FormulaSequence` order) | Raw input fields only |
| 1 | Mapping phase | All pass-throughs and field mappings | Raw input fields + formula results |

### Pass-through Mappings

| Field | Value | Notes |
|-------|-------|-------|
| `OutputCreationSequence` | `1.0` | Runs after formulas |
| `InputFieldName` | Source key from Extract | `AccountName` or `Quote:QuoteLineGroup` (for array pass-through) |
| `OutputFieldName` | Template token name | `CustomerOrgName` or `QuoteLineGroup` |
| `OutputObjectName` | `json` | Required |
| `OutputFieldFormat` | *(see table below)* | Optional formatting |

### Formula Items

| Field | Value | Notes |
|-------|-------|-------|
| `OutputCreationSequence` | `0.0` | Runs before mappings |
| `OutputFieldName` | `Formula` | Required literal |
| `OutputObjectName` | `Formula` | Required literal |
| `FormulaExpression` | Infix expression | `IF(ISBLANK(Group:Line:ParentId),true,false)` |
| `FormulaConverted` | RPN (auto-generated on save) | Do NOT manually author |
| `FormulaResultPath` | Output key for the result | `Group:Line:IF_Parent` or `IF_HasGroups` |
| `FormulaSequence` | Execution order within OCS=0 | `1.0`, `2.0`, `3.0` |

### Supported Formula Functions

| Category | Supported | NOT Supported (saves silently, produces no output) |
|----------|-----------|---------------|
| Arithmetic | `+`, `-`, `*`, `/` | |
| Comparison | `>`, `<`, `>=`, `<=`, `==`, `!=` | |
| Logical | `IF`, `NOT`, `AND`, `OR` | `CASE` |
| Math | `ABS`, `ROUND`, `FLOOR`, `CEILING`, `MAX`, `MIN`, `SQRT` | `MOD`, `POWER` |
| String | `CONCAT`, `SUBSTRING` | `LEN`, `UPPER`, `LOWER`, `TEXT`, `FORMAT` |
| Null check | `ISBLANK`, `ISNOTBLANK` | `ISNULL`, `NULLVALUE`, `BLANKVALUE` |
| Array | `LIST` | |
| Type conversion | *(none)* | `VALUE`, `TEXT`, `FORMAT` |

### OutputFieldFormat Values

| Format | Effect | Used For |
|--------|--------|----------|
| `Boolean` | Coerces string `"true"`/`"false"` to boolean | `IF_` conditional tokens |
| `CurrencyRounded` | Adds `$` prefix + comma separators + rounds to 2 decimal places | All money fields (GrandTotal, ListPrice, NetUnitPrice, NetTotalPrice) |
| `List<Map>` | Marks value as array pass-through | Nested data (QuoteLineGroup -> template sections) |
| *(blank)* | Pass-through unchanged | Most scalar fields |

### The Cartesian Product Anti-Pattern

**Root cause**: The Transform engine iterates arrays. When you map fields from TWO different input arrays (or from nested + flat paths) to the SAME output, it produces N*M entries.

**In this project**: The pass-through `Quote:QuoteLineGroup` -> `QuoteLineGroup` carries ALL nested data. If you ALSO add individual mappings like `Quote:QuoteLineGroup:Name` -> `QuoteLineGroup:Name`, the engine treats them as separate sources -- creating a cross-join.

**What IS safe alongside a pass-through**:
- Flat token renames (non-nested paths like `AccountName` -> `CustomerOrgName`)
- Formulas with `FormulaResultPath` targeting INTO the nested structure
- `OutputFieldFormat` entries that format existing tokens
- Conditional booleans (`IF_QuoteLineGroupYes`, `IF_HasGroups`)

**What is DANGEROUS alongside a pass-through**:
- Individual field mappings at nested paths (`Quote:QuoteLineGroup:Name` -> `QuoteLineGroup:Name`)
- Any mapping where `InputFieldName` traverses into the same array the pass-through covers

---

## Template Architecture Patterns

### Pattern A: Flat (IM_QuoteOrderForm)

Single `{{#Line}}` repeating section for QuoteLineItems.

```
{{#Line}} {{Line:ProductName}} | {{Line:Quantity}} | ... {{/Line}}
```

- **Extract**: 5 query sequences (Quote -> Account -> Contact -> QuoteLineItem -> Product2)
- **Transform**: Simple pass-through renames
- **Use when**: No grouping needed, simple flat table of line items

### Pattern B: Grouped + CQL (IM_QuoteGroupedOrderForm)

`{{#Group}}{{#Line}}{{#CQL}}` three-level nesting via QuoteLineRelationship.

```
{{#Group}} {{Group:GroupName}}
  {{#Line}} {{Group:Line:ProductName}} | ...
    {{#CQL}} {{Line:CQL:ProductName}} | ... {{/CQL}}
  {{/Line}}
{{/Group}}
```

- **Extract**: 6 query sequences (Quote -> Account -> QLG -> QLI -> QuoteLineRelationship -> ChildQLI)
- **Transform**: Pass-through (`Quote:QuoteLineGroup` -> `Group`) + IF_Parent formula
- **IF_Parent formula**: `IF(ISBLANK(Group:Line:ParentId), true, false)`
- **Use when**: Products have child/bundle items accessed via QuoteLineRelationship

### Pattern C: Grouped + IF_IsChild (IM_QuoteGroupedChildFilter)

`{{#QLI}}` with `{{IF_IsChild:DocProductName}}` conditional formatting -- no CQL nesting.

```
{{#QuoteLineGroup}}
  {{#QLI}} {{QuoteLineGroup:QLI:DocProductName}} | ...
    {{IF_IsChild:DocProductName}} (shown differently for children)
  {{/QLI}}
{{/QuoteLineGroup}}
```

- **Extract**: 4 query sequences (Quote -> Account -> QLG -> QLI with ORDER BY)
- **IsChild formula** (in Extract): `ISNOTBLANK(ParentQuoteLineItemId)` at sequence 3.0
- **Transform**: DocProductName formula + pass-through
- **Key advantage**: Simpler query structure -- ALL QLIs in one pass, parent/child distinguished at render time
- **Use when**: You want parent/child differentiation without QuoteLineRelationship traversal

### Pattern D: Dual-Path Conditional (Customer_Quote)

`{{#IF_HasGroups}}` / `{{#IF_no_Groups}}` mutually exclusive sections.

```
{{#IF_HasGroups}}
  {{#Group}} ... {{#Line}} ... {{#CQL}} ... {{/CQL}} {{/Line}} {{/Group}}
{{/IF_HasGroups}}
{{#IF_no_Groups}}
  {{#Line}} ... {{/Line}}
{{/IF_no_Groups}}
```

- **Extract**: 7 query sequences (Quote -> Account -> QLG -> QLI-grouped -> QLR -> QLI-child -> QLI-flat)
- **Transform formulas**: IF_HasGroups (`ISNOTBLANK(Group:GroupName)`), IF_no_Groups (inverse), IF_Parent
- **FlatLine path**: Queries ALL QLIs by `QuoteId` (ungrouped fallback)
- **Use when**: Quote may or may not have QuoteLineGroups; need both rendering paths

---

## Token Syntax Reference

| Type | Syntax | Example | Notes |
|------|--------|---------|-------|
| Scalar | `{{Name}}` | `{{QuoteNumber}}` | Simple value replacement |
| Section open | `{{#Name}}` | `{{#Group}}` | Begins repeating/conditional block |
| Section close | `{{/Name}}` | `{{/Group}}` | Ends repeating/conditional block |
| Conditional section | `{{#IF_x}}...{{/IF_x}}` | `{{#IF_HasGroups}}...{{/IF_HasGroups}}` | Renders when `IF_x` is boolean `true` |
| Conditional field | `{{IF_x:Field}}` | `{{IF_IsChild:DocProductName}}` | Field rendered conditionally |
| Nested section | `{{#Outer}}{{#Inner}}...{{/Inner}}{{/Outer}}` | `{{#Group}}{{#Line}}...` | Array within array |
| Table row | `{{#Section}}...{{/Section}}` in first/last cell | See DOCX layout | Entire row repeats per element |

**Truthy/falsy behavior**: `{{#Field}}` renders when value is non-empty string, non-empty array, or non-empty object. Skips when absent, `null`, `false`, empty string, or empty array.

---

## Adding a New Token (Checklist)

1. **Extract ODT**: Add field mapping item (`InputFieldName`: source path, `OutputFieldName`: output key, `OutputCreationSequence: 1.0`, `OutputObjectName: json`). If the field is on a new object, add an object query item first.
2. **Transform ODT**: Add pass-through mapping (`InputFieldName` = Extract output key, `OutputFieldName` = template token name, `OutputCreationSequence: 1.0`, `OutputObjectName: json`). For computed values, add a formula item at `OutputCreationSequence: 0.0`.
3. **Python build script**: Insert `{{TokenName}}` at the appropriate location in the DOCX layout.
4. **`.dt-meta.xml`**: Add the token name to the `tokenList` (comma-separated, must match DOCX token exactly).
5. **Rebuild DOCX**: Run `python3 scripts/build_template_<name>.py`
6. **Deploy**: Follow the deployment workflow for that template (ODTs -> activate -> template binary).

---

## Adding a New Template (Checklist)

1. **Choose pattern** (A/B/C/D) based on data requirements
2. **Create Extract ODT** (`.rpt-meta.xml`):
   - Unique `name` (alphanumeric camelCase)
   - Unique `globalKey` prefix (new UUID base)
   - Object queries for all needed SObjects
   - Field mappings for all output fields
3. **Create Transform ODT** (`.rpt-meta.xml`):
   - Unique `name` (alphanumeric camelCase)
   - Unique `globalKey` prefix (new UUID base)
   - Pass-through mappings and/or formulas
   - Set `targetOutputFileName` to match template name
4. **Create Python build script** (`scripts/build_template_<name>.py`):
   - Use `python-docx` to generate DOCX with `{{token}}` placeholders
   - Output to `force-app/main/default/documentTemplates/<name>.dt`
5. **Create `.dt-meta.xml`**:
   - Reference Extract and Transform ODT names (without `_1` suffix)
   - List all tokens in `tokenList`
   - Set `uniqueName` with `_1` suffix and `versionNumber: 1`
6. **Create deployment manifest** (`manifest/package-<name>.xml`)
7. **Build DOCX binary**: Run the Python script
8. **Deploy** following the standard workflow
9. **Update CLAUDE.md and README.md** with new template documentation

---

## Deployment Workflow

### Template 1: IM_QuoteOrderForm (Flat)

```bash
python3 scripts/build_template.py
sf project deploy start --manifest manifest/package-odts-only.xml --target-org maintwo --wait 10
sf data update record -s OmniDataTransform -w "Name='IMQuoteOrderFormExtract'" -v "IsActive=true" --target-org maintwo
sf data update record -s OmniDataTransform -w "Name='IMQuoteOrderFormTransform'" -v "IsActive=true" --target-org maintwo
sf project deploy start --manifest manifest/package-templates-only.xml --target-org maintwo --wait 10
```

### Template 2: IM_QuoteGroupedOrderForm (CQL)

```bash
python3 scripts/build_template_groups.py
sf project deploy start --manifest manifest/package-grouped-odts-only.xml --target-org maintwo --wait 10
sf data update record -s OmniDataTransform -w "Name='IMQuoteGroupedOrderFormExtract'" -v "IsActive=true" --target-org maintwo
sf data update record -s OmniDataTransform -w "Name='IMQuoteGroupedOrderFormTransform'" -v "IsActive=true" --target-org maintwo
sf project deploy start --metadata "DocumentTemplate:IM_QuoteGroupedOrderForm_1" --target-org maintwo --wait 10
```

### Template 3: IM_QuoteGroupedChildFilter (IF_IsChild)

```bash
python3 scripts/build_template_grouped_childfilter.py
sf project deploy start --manifest manifest/package-grouped-childfilter.xml --target-org maintwo --wait 10
sf data update record -s OmniDataTransform -w "Name='IMQuoteGroupedChildFilterExtract'" -v "IsActive=true" --target-org maintwo
sf data update record -s OmniDataTransform -w "Name='IMQuoteGroupedChildFilterTransform'" -v "IsActive=true" --target-org maintwo
sf project deploy start --metadata "DocumentTemplate:IM_QuoteGroupedChildFilter_1" --target-org maintwo --wait 10
```

### Template 4: Customer_Quote (Dual-Path)

```bash
python3 scripts/build_template_customer_quote.py
sf project deploy start --manifest manifest/package-customer-quote.xml --target-org maintwo --wait 10
sf data update record -s OmniDataTransform -w "Name='CustomerQuoteExtract'" -v "IsActive=true" --target-org maintwo
sf data update record -s OmniDataTransform -w "Name='CustomerQuoteTransform'" -v "IsActive=true" --target-org maintwo
sf project deploy start --metadata "DocumentTemplate:Customer_Quote_1" --target-org maintwo --wait 10
```

---

## Validation & Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| All tokens blank | Extract failing -- duplicate queries or null `OutputObjectName` | Check for duplicate `InputObjectQuerySequence` + `OutputFieldName` combos; ensure all items have `OutputObjectName` |
| Duplicate rows (cartesian product) | Individual field mappings alongside pass-through in Transform | Remove individual nested mappings -- keep only pass-through + formulas |
| Conditional section always shows | `IF_` token not receiving Boolean type | Ensure formula produces `true`/`false` (not string); set `OutputFieldFormat: Boolean` if coercion needed |
| Conditional section never shows | Formula references field that doesn't exist | Check `FormulaResultPath` matches what template expects; verify Extract populates the referenced field |
| Repeating section empty | Missing pass-through or wrong output path | Trace: Extract produces array? -> Transform maps it? -> tokenList includes it? |
| Wrong sort order | Missing ORDER BY item | Add separate `omniDataTransformItem` with `FilterOperator="ORDER BY"` at same `InputObjectQuerySequence` |
| Child items missing | Wrong `InputFieldName` on child-of join | Use child's FK field (e.g., `QuoteLineGroupId`) not `Id` |
| Formula produces no output | Unsupported function used | Check `FormulaConverted` in XML -- if empty/null, the function isn't supported |
| Template deploy GACK | ODTs not yet deployed/activated | Always deploy ODTs first and activate before template binary |
| N*M*G explosion | Multiple FilterGroups on nested sequence | Use only FilterGroup `0` on nested child sequences |
| `[object Object]` for token | Nested object passed where string expected | Check Transform output -- ensure scalar tokens produce strings, not objects |
| Field silently blank | Relationship traversal without explicit join sequence | Add explicit object query for the related object, or use a denormalized field |

---

## GlobalKey Convention

| ODT | GlobalKey Prefix |
|-----|-----------------|
| `IMQuoteOrderFormExtract` | `a1b2c3d4-1111-4000-8000-*` |
| `IMQuoteOrderFormTransform` | `b2c3d4e5-2222-4000-8000-*` |
| `IMQuoteGroupedOrderFormExtract` | `c1d2e3f4-3333-4000-8000-*` |
| `IMQuoteGroupedOrderFormTransform` | `d2e3f4a5-4444-4000-8000-*` |
| `IMQuoteGroupedChildFilterExtract` | `e1f2a3b4-5555-4000-8000-*` |
| `IMQuoteGroupedChildFilterTransform` | `f2a3b4c5-6666-4000-8000-*` |
| `CustomerQuoteExtract` | `e4a1b2c3-5000-4000-8000-*` |
| `CustomerQuoteTransform` | `f5b2c3d4-6000-4000-8000-*` |

**Convention**: Each new ODT gets a unique 8-char hex prefix + 4-digit group identifier. Items within an ODT increment the final 12 digits (e.g., `...000000000001`, `...000000000002`).

---

## Metadata XML Structure

### Extract ODT (`.rpt-meta.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<OmniDataTransform xmlns="http://soap.sforce.com/2006/04/metadata">
    <active>false</active>
    <assignmentRulesUsed>false</assignmentRulesUsed>
    <deletedOnSuccess>false</deletedOnSuccess>
    <errorIgnored>false</errorIgnored>
    <fieldLevelSecurityEnabled>false</fieldLevelSecurityEnabled>
    <inputType>JSON</inputType>
    <name>MyExtractName</name>
    <nullInputsIncludedInOutput>false</nullInputsIncludedInOutput>
    <omniDataTransformItem>
        <disabled>false</disabled>
        <filterGroup>0.0</filterGroup>
        <globalKey>unique-guid-here</globalKey>
        <inputFieldName>Id</inputFieldName>
        <inputObjectName>Quote</inputObjectName>
        <inputObjectQuerySequence>1.0</inputObjectQuerySequence>
        <name>MyExtractName</name>
        <outputCreationSequence>0.0</outputCreationSequence>
        <outputFieldName>Quote</outputFieldName>
        <outputObjectName>json</outputObjectName>
    </omniDataTransformItem>
    <!-- field mapping items at outputCreationSequence 1.0 -->
    <outputType>JSON</outputType>
    <type>Extract</type>
    <versionNumber>1.0</versionNumber>
</OmniDataTransform>
```

### Transform ODT (`.rpt-meta.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<OmniDataTransform xmlns="http://soap.sforce.com/2006/04/metadata">
    <active>false</active>
    <assignmentRulesUsed>false</assignmentRulesUsed>
    <deletedOnSuccess>false</deletedOnSuccess>
    <errorIgnored>false</errorIgnored>
    <fieldLevelSecurityEnabled>false</fieldLevelSecurityEnabled>
    <inputType>JSON</inputType>
    <name>MyTransformName</name>
    <nullInputsIncludedInOutput>false</nullInputsIncludedInOutput>
    <omniDataTransformItem>
        <!-- Formula items (OCS 0.0) -->
        <!-- Pass-through items (OCS 1.0) -->
    </omniDataTransformItem>
    <outputType>JSON</outputType>
    <targetOutputFileName>TemplateName(Version 1)</targetOutputFileName>
    <type>Transform</type>
    <versionNumber>1.0</versionNumber>
</OmniDataTransform>
```

---

## File Reference

| File | Purpose |
|------|---------|
| `scripts/build_template.py` | Generates flat order form DOCX |
| `scripts/build_template_groups.py` | Generates grouped (CQL) DOCX |
| `scripts/build_template_grouped_childfilter.py` | Generates grouped (IF_IsChild) DOCX |
| `scripts/build_template_customer_quote.py` | Generates dual-path conditional DOCX |
| `force-app/main/default/omniDataTransforms/*Extract_1.rpt-meta.xml` | Extract ODT metadata |
| `force-app/main/default/omniDataTransforms/*Transform_1.rpt-meta.xml` | Transform ODT metadata |
| `force-app/main/default/documentTemplates/*.dt` | Template binary (generated) |
| `force-app/main/default/documentTemplates/*.dt-meta.xml` | Template metadata (ODT refs, tokenList) |
| `manifest/package-odts-only.xml` | Flat template ODT deployment |
| `manifest/package-templates-only.xml` | Flat template binary deployment |
| `manifest/package-grouped-odts-only.xml` | Grouped (CQL) ODT deployment |
| `manifest/package-grouped-childfilter.xml` | ChildFilter ODT deployment |
| `manifest/package-customer-quote.xml` | Customer Quote ODT deployment |
