# customer-quote-docgen-project

Salesforce Document Generation template for a dual-path Customer Quote (Pattern D). Contains the DOCX template binary, Extract ODT, and Transform ODT metadata -- deployable to any RLM org running API v67.0+.

## What's Included

| Component | File | Purpose |
|-----------|------|---------|
| Template binary | `documentTemplates/Customer_Quote_1.dt` | DOCX with tokenized placeholders |
| Template metadata | `documentTemplates/Customer_Quote_1.dt-meta.xml` | Wires template to Extract + Transform ODTs |
| Extract ODT | `omniDataTransforms/CustomerQuoteExtract_1.rpt-meta.xml` | Queries Quote, Account, QLGs, QLIs, QuoteLineRelationships, child QLIs |
| Transform ODT | `omniDataTransforms/CustomerQuoteTransform_1.rpt-meta.xml` | Maps Extract output to template tokens; computes IF_HasGroups/IF_Parent conditionals |

## Architecture: Dual-Path Conditional (Pattern D)

Uses `{{#IF_HasGroups}}` / `{{#IF_no_Groups}}` mutually exclusive sections:

- **IF_HasGroups** -- renders grouped table with Group -> Line -> IF_Parent -> CQL nesting
- **IF_no_Groups** -- renders flat table with simple Line repeating section

### Extract Structure (7 query sequences)

1. Quote (root)
2. Account (FK lookup by AccountId)
3. QuoteLineGroup (child-of Quote)
4. QuoteLineItem - grouped (child-of QLG)
5. QuoteLineRelationship (child-of QLI)
6. QuoteLineItem - child (FK lookup by ChildQuoteLineItemId)
7. QuoteLineItem - flat (child-of Quote, ungrouped fallback)

### Transform Formulas

- `IF_HasGroups`: `ISNOTBLANK(Group:GroupName)` -- boolean, renders grouped path
- `IF_no_Groups`: `IF(ISBLANK(Group:GroupName), true, false)` -- boolean, renders flat path
- `IF_Parent`: `IF(ISBLANK(Group:Line:ParentId), true, false)` -- parent-only row styling

## Deployment

Deploy to any target org with the following sequence (ODTs first, then template):

```bash
# 1. Deploy ODTs
sf project deploy start --metadata "OmniDataTransform:CustomerQuoteExtract_1" "OmniDataTransform:CustomerQuoteTransform_1" --target-org <alias> --wait 10

# 2. Activate ODTs
sf data update record -s OmniDataTransform -w "Name='CustomerQuoteExtract'" -v "IsActive=true" --target-org <alias>
sf data update record -s OmniDataTransform -w "Name='CustomerQuoteTransform'" -v "IsActive=true" --target-org <alias>

# 3. Deploy template binary
sf project deploy start --metadata "DocumentTemplate:Customer_Quote_1" --target-org <alias> --wait 10

# 4. Activate template (optional -- verify first)
sf data update record -s DocumentTemplate -w "UniqueName='Customer_Quote_1'" -v "Status='Active' IsActive=true" --target-org <alias>
```

## Token Convention

| Scope | Prefix | Example |
|-------|--------|---------|
| Flat (header/footer) | none | `{{QuoteNumber}}`, `{{CustomerOrgName}}` |
| Group-level | `Group:` | `{{Group:GroupName}}` |
| Line within group | `Group:Line:` | `{{Group:Line:ProductName}}` |
| CQL child line | `Line:CQL:` | `{{Line:CQL:ProductName}}` |
| Flat line (no groups) | `Line:` | `{{Line:ProductName}}` |
| Conditional section | `#IF_` | `{{#IF_HasGroups}}...{{/IF_HasGroups}}` |
| Parent-only format | `#IF_Parent` | `{{#IF_Parent}}...{{/IF_Parent}}` |

## Key Rules

1. All currency fields use `CurrencyRounded` format (not `Currency`)
2. ODT naming is alphanumeric camelCase only (no underscores)
3. Deploy ODTs BEFORE template binary (bundling causes GACK)
4. Never add individual field mappings alongside a pass-through (cartesian product)
5. The `.dt` binary is generated -- edit the Python build script in the source project, not the binary

## Source Project

This is an export from `maintwo-docgen-project`. The Python build script (`scripts/build_template_customer_quote.py`) and full template development workflow live there.

## Skill Reference

See `SKILLS.md` for comprehensive ODT authoring guidance including:
- Extract/Transform engine behavior
- All template architecture patterns (A/B/C/D)
- Formula function catalog
- Troubleshooting table
