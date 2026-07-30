# Customer Quote Document Generation

Salesforce Document Generation template implementing a **dual-path conditional** pattern for Customer Quotes. Automatically renders either a grouped or flat line-item table depending on whether the Quote has QuoteLineGroups.

Deployable to any Revenue Lifecycle Management org running API v67.0+.

## What's Included

```
force-app/main/default/
├── documentTemplates/
│   ├── Customer_Quote_1.dt              # DOCX binary with tokenized placeholders
│   └── Customer_Quote_1.dt-meta.xml     # Wires template to Extract + Transform ODTs
└── omniDataTransforms/
    ├── CustomerQuoteExtract_1.rpt-meta.xml   # Queries Quote, Account, QLGs, QLIs, relationships
    └── CustomerQuoteTransform_1.rpt-meta.xml # Maps extract output to template tokens
```

## How It Works

The template uses two mutually exclusive conditional sections:

- **`{{#IF_HasGroups}}`** — Renders a grouped table: Group → Line → CQL child lines, with `IF_Parent` formatting for parent rows
- **`{{#IF_no_Groups}}`** — Renders a flat table with a simple `{{#Line}}` repeating section

The Extract ODT queries 7 object sequences (Quote, Account, QuoteLineGroup, grouped QLIs, QuoteLineRelationships, child QLIs, and flat QLIs). The Transform ODT computes the conditional booleans and passes nested data through to the template engine.

## Prerequisites

- Salesforce org with Revenue Lifecycle Management enabled
- API version 67.0+
- `sf` CLI installed ([Salesforce CLI](https://developer.salesforce.com/tools/salesforcecli))

## Deployment

Deploy components in order — ODTs first, then the template binary:

```bash
# 1. Deploy ODTs
sf project deploy start \
  --metadata "OmniDataTransform:CustomerQuoteExtract_1" \
  --metadata "OmniDataTransform:CustomerQuoteTransform_1" \
  --target-org <alias> --wait 10

# 2. Activate ODTs
sf data update record -s OmniDataTransform \
  -w "Name='CustomerQuoteExtract'" -v "IsActive=true" --target-org <alias>
sf data update record -s OmniDataTransform \
  -w "Name='CustomerQuoteTransform'" -v "IsActive=true" --target-org <alias>

# 3. Deploy template binary
sf project deploy start \
  --metadata "DocumentTemplate:Customer_Quote_1" \
  --target-org <alias> --wait 10

# 4. Activate template
sf data update record -s DocumentTemplate \
  -w "UniqueName='Customer_Quote_1'" -v "Status='Active' IsActive=true" \
  --target-org <alias>
```

> **Important:** Deploying ODTs and templates together in the same operation causes platform GACK errors. Always deploy and activate ODTs first.

## Token Reference

| Scope | Prefix | Example |
|-------|--------|---------|
| Header/footer | *(none)* | `{{QuoteNumber}}`, `{{CustomerOrgName}}` |
| Group-level | `Group:` | `{{Group:GroupName}}` |
| Line within group | `Group:Line:` | `{{Group:Line:ProductName}}` |
| CQL child line | `Line:CQL:` | `{{Line:CQL:ProductName}}` |
| Flat line (no groups) | `Line:` | `{{Line:ProductName}}` |
| Conditional section | `#IF_` | `{{#IF_HasGroups}}...{{/IF_HasGroups}}` |
| Parent-only format | `#IF_Parent` | `{{#IF_Parent}}...{{/IF_Parent}}` |

## Key Rules

1. All currency fields use `CurrencyRounded` format (rounds to 2 decimal places with `$` prefix)
2. ODT naming must be alphanumeric camelCase (no underscores or spaces)
3. Never add individual field mappings alongside a pass-through for the same nested path (causes cartesian product / duplicate rows)
4. The `.dt` binary is generated — do not edit it directly

## Additional Documentation

- **`CLAUDE.md`** — Project-specific conventions and deployment details
- **`SKILLS.md`** — Comprehensive ODT authoring reference (Extract/Transform engine, all architecture patterns, formula catalog, troubleshooting)

## License

MIT
