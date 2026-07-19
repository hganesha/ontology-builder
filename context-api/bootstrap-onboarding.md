# ContextGraph Bootstrap and Cold-Start Onboarding

**Version:** 1.0  
**Status:** Architecture reference  
**Scope:** How a tenant gets value before data discovery completes, using LLM-assisted pre-population and standard templates

---

## The cold-start problem

The full offline ingestion pipeline — metadata scanning, entity reconciliation, BOM ingestion, relationship population, precomputed path materialization — takes days to weeks for a new tenant. Without a cold-start strategy, the system delivers no value until that pipeline completes.

This is a false constraint. A meaningful subset of ContextGraph's value is available immediately from pre-built industry packs, and the rest can be bootstrapped from a short intake session combined with LLM-assisted draft generation and steward bulk-approval — days before the first source scan runs.

---

## Value available at Day 0 (no customer data required)

The global ontology and industry packs are pre-built and tenant-independent. They are available to every new tenant immediately upon account creation.

| Capability | What works | Example |
|---|---|---|
| Definition questions | All base pack metric definitions | "What does backlog mean?" → full definition, disambiguation, do-not-confuse-with |
| Ontology hierarchy | Full semiconductor and supply chain concept hierarchy | "What is HBM?" → HBM → DRAM → memory component → component |
| Public entity resolution | Well-known companies, fabs, component classes | "SK Hynix" → SK Hynix Inc., "TSMC" → Taiwan Semiconductor Manufacturing Co. |
| Standard metric disambiguation | 30+ pre-defined supply chain metrics | "backlog vs open PO" → distinct definitions, field-level differences |
| Relationship type vocabulary | All standard relationship types | SUPPLIES, USED_IN, BUILDS, SOLD_TO, MANUFACTURED_AT |
| Source system templates | Standard binding templates for SAP MM, Snowflake, Oracle SCM | Available as drafts; require tenant parameterization before activation |

This covers Category A questions (definition) and partial Category B questions (entity/relationship using public entities) from the processing-paths model. An agent integrated on Day 0 can correctly disambiguate standard supply chain terms and resolve well-known public entities.

---

## Bootstrap intake: what the customer provides

A 30-minute structured intake session produces the inputs needed to generate tenant-specific drafts. This can be a web form, a guided CLI, or a conversational interface.

### Required inputs

```json
{
  "tenant_profile": {
    "company_name": "NVIDIA Corporation",
    "industry": "semiconductor",
    "primary_domain": "supply_chain",
    "company_role": "fabless_oem"
  },
  "source_systems": [
    {
      "system_type": "sap_s4hana",
      "label": "SAP Production",
      "primary_domains": ["procurement", "inventory", "materials"],
      "company_codes": ["1000", "1001"],
      "purchasing_orgs": ["1000"],
      "connection_mode": "private_gateway"
    },
    {
      "system_type": "snowflake",
      "label": "Analytics Warehouse",
      "primary_domains": ["supply_chain_analytics", "supplier_performance"],
      "database": "SUPPLY_CHAIN_DW",
      "connection_mode": "private_gateway"
    }
  ],
  "primary_entities": {
    "tenant_entity_name": "NVIDIA Corporation",
    "supplier_count_estimate": "50-200",
    "product_family_names": ["Hopper", "Blackwell", "Ada Lovelace"],
    "key_suppliers": [
      "SK Hynix",
      "Samsung",
      "TSMC",
      "CoWoS packaging"
    ]
  },
  "metric_overrides": [
    {
      "base_metric": "demand_backlog",
      "override_note": "Excludes orders with reason codes Z1 and Z3"
    }
  ]
}
```

### Optional enrichment inputs

- Supplier master CSV (name, internal vendor code, ERP ID) — unlocks bulk alias generation
- Product/SKU list — enables product entity pre-population
- Existing metric glossary or data dictionary — used to generate metric overlay drafts
- Current SAP company code / purchasing org structure — used to parameterize binding templates

---

## LLM-assisted draft generation

The bootstrap pipeline takes the intake inputs and generates a set of draft artifacts. Every artifact is labeled `PROPOSED` and enters the steward review queue — nothing is published to the runtime store automatically.

### Draft 1: Entity type scaffolding

From `industry: semiconductor` and `company_role: fabless_oem`, the LLM selects the relevant entity subtypes from the semiconductor extension that apply to this customer and suppresses irrelevant ones (e.g., a fabless OEM does not need Fab or Foundry entity types as primary entities).

```json
{
  "draft_type": "entity_type_selection",
  "proposed_active_types": [
    "supplier_tier1", "supplier_tier2",
    "component", "memory_component", "hbm",
    "sku", "product_family",
    "purchase_order", "purchase_order_item",
    "business_unit", "legal_entity"
  ],
  "proposed_inactive_types": ["fab", "foundry", "process_node"],
  "basis": "fabless_oem_semiconductor_template_v1",
  "status": "PROPOSED"
}
```

Steward reviews and approves the active/inactive split. This takes minutes, not days.

---

### Draft 2: Supplier alias candidates

From the key supplier list and optional supplier CSV, the LLM generates normalized alias candidates for each supplier mapped against known public entities.

```
Input:  "SK Hynix"
Output:
  Canonical match:  SK Hynix Inc. (public entity, pre-loaded)
  Proposed aliases: "SK hynix", "SK-Hynix", "Hynix", "SKH",
                    "Hynix Semiconductor", "SK_HYNIX", "SKHYNIX"
  Basis:            public_name_normalization + common_abbreviation_patterns
  Status:           PROPOSED
```

If a supplier CSV is provided with an internal vendor code (`SK_HYNIX_LIFNR`, `00010427`), the draft includes a source identifier mapping:

```json
{
  "canonical_entity": "SK Hynix Inc.",
  "source_identifiers": [
    { "source": "sap-s4-prod", "field": "LIFNR", "value": "00010427" },
    { "source": "snowflake-analytics", "field": "SUPPLIER_CODE", "value": "SK_HYNIX" }
  ],
  "status": "PROPOSED"
}
```

Steward bulk-approves or corrects the alias list. A 50-supplier list can typically be reviewed in under an hour via the steward bulk-review UI.

---

### Draft 3: SAP binding templates

SAP S/4HANA table structures for core procurement and inventory operations are well-known and stable. The bootstrap pipeline instantiates binding templates parameterized with the tenant's company codes and purchasing organizations.

```json
{
  "draft_type": "source_binding",
  "operation": "procurement.open_purchase_order_quantity",
  "source_id": "sap-s4-prod",
  "binding_template": "sap_mm_open_po_v3",
  "parameters": {
    "company_codes": ["1000", "1001"],
    "purchasing_orgs": ["1000"],
    "status_filter": ["A", "B"],
    "exclude_deletion_flag": true
  },
  "approval_required_fields": [
    "custom_status_codes",
    "additional_exclusion_filters"
  ],
  "status": "PROPOSED"
}
```

The template covers the standard case. `approval_required_fields` marks parameters the steward must confirm or override before the binding is published.

Binding templates available at bootstrap for SAP S/4HANA:
- `open_purchase_order_quantity`
- `goods_receipt_quantity`
- `supplier_master`
- `material_master`
- `inventory_on_hand`
- `sales_order_open_quantity`
- `purchase_order_schedule_lines`

Snowflake templates are more variable (schema names differ per tenant) but the pipeline generates parameterized drafts from the `database` name provided during intake.

---

### Draft 4: Source routing rules

From the stated system types and domains, the pipeline pre-populates routing rules without requiring a live metadata scan.

```json
{
  "draft_type": "route_rule",
  "operation": "procurement.open_purchase_order_quantity",
  "primary_source": "sap-s4-prod",
  "primary_freshness": "operational",
  "secondary_source": "snowflake-analytics",
  "secondary_freshness": "T-1_analytical",
  "routing_logic": "use_primary_for_current; use_secondary_for_historical_analysis",
  "status": "PROPOSED"
}
```

These rules are based on the standard pattern: SAP for current operational data, analytical warehouse for historical trends. The steward confirms or adjusts before publication.

---

### Draft 5: Metric overlay drafts

If the customer provides metric override notes (e.g., "our backlog excludes reason codes Z1 and Z3"), the LLM generates a tenant overlay on top of the base pack metric definition.

```json
{
  "draft_type": "metric_overlay",
  "base_metric": "demand_backlog",
  "tenant_override": {
    "additional_exclusions": {
      "sap_reason_codes": ["Z1", "Z3"],
      "description": "Excludes backlog orders in internal hold status per NVIDIA FY26 planning policy"
    }
  },
  "conflict_check": "PASS",
  "status": "PROPOSED"
}
```

---

### Draft 6: Relationship templates

From the product family names and supplier list, the pipeline proposes draft relationship types that likely apply, to be confirmed once BOM data is ingested.

```
Proposed draft assertions (confidence: TEMPLATE_BASED, require BOM confirmation):
  SK Hynix Inc.  --SUPPLIES-->  memory_component (HBM class)
  TSMC           --MANUFACTURES_DIES_FOR-->  product_family:Hopper
  product_family:Hopper  --BUILDS-->  sku:H100
  product_family:Blackwell  --BUILDS-->  sku:B100, B200
```

These are labeled `assertion_class: TEMPLATE_PROPOSED` — a new assertion class distinct from `asserted` (from a source) and `inferred` (from an LLM over existing facts). They are visible in the admin UI but do not enter the runtime graph until upgraded to `asserted` or `derived` after ingestion confirmation.

---

## Steward review and bulk approval

The bootstrap queue is designed for rapid bulk review, not item-by-item approval.

| Artifact type | Expected review time | Review mechanism |
|---|---|---|
| Entity type selection | 5 minutes | Toggle active/inactive per type |
| Supplier aliases (50 suppliers) | 30–60 minutes | Grouped by supplier; flag exceptions; approve-all-unmodified |
| SAP binding templates | 30 minutes | Review template params; confirm or override `approval_required_fields` |
| Source routing rules | 15 minutes | Review operation → source mapping table; adjust priority |
| Metric overlays | 15–30 minutes | Review each override; confirm definition text |
| Relationship templates | Deferred | Mark as TEMPLATE_PROPOSED; confirm individually after BOM ingestion |

Total steward time for a standard semiconductor tenant: 2–4 hours to go from intake submission to first bundle publication.

---

## Progressive value unlocking

```
Day 0   ─── Account created
        │   Industry packs loaded (no tenant data)
        │   Public entity resolution active
        │   All definition/metric disambiguation active
        │
Day 1   ─── Bootstrap intake completed (30 min)
        │   LLM draft generation runs (~5 min)
        │   Drafts available in steward review queue
        │
Day 2   ─── Steward bulk-approves aliases, bindings, routing rules
        │   First tenant bundle compiled and published
        │   Tenant-specific supplier resolution active
        │   SAP binding templates active (gateway configured)
        │   Routing rules active
        │   Custom metric overlays active
        │
Week 1  ─── Metadata discovery scan completes (SAP, Snowflake)
        │   Source-validated bindings replace template bindings
        │   Real SAP entity IDs confirmed and mapped
        │   Custom fields and status codes discovered and incorporated
        │
Week 2  ─── BOM and master data ingestion completes
        │   Full supplier/component/product relationship graph loaded
        │   Precomputed traversal paths available
        │   TEMPLATE_PROPOSED relationships upgraded to ASSERTED
        │
Week 3+ ─── Calibration data accumulating
            Confidence calibration sweep (after sufficient eval cases)
            Numeric confidence enabled for validated query classes
```

---

## What an agent can do at each stage

| Stage | Agent capability |
|---|---|
| Day 0 | Correctly defines all standard supply chain metrics; resolves well-known public entities; returns UNSUPPORTED for tenant-specific queries |
| Day 2 | Resolves the customer's suppliers; routes procurement questions to SAP; returns governed plans with tenant-specific bindings; disambiguates tenant metric overrides |
| Week 1 | Fully validated source bindings; real ERP entity IDs; custom field support; accurate source routing |
| Week 2 | Full relationship traversal; supply chain impact analysis; product exposure queries |

---

## Governance invariants that apply to all bootstrap artifacts

Bootstrap does not create a special governance path. The same rules apply:

- No draft artifact enters the runtime store without steward approval
- `TEMPLATE_PROPOSED` relationship assertions are not traversable in the production graph
- LLM-generated alias candidates are `PROPOSED` — the alias resolver ignores them until `PUBLISHED`
- Binding templates are not invoked at execution time until the binding is in `PUBLISHED` state
- Every published artifact carries a provenance record: `generated_by: bootstrap_pipeline_v1`, `reviewed_by: steward_id`, `approved_at: timestamp`
- Bootstrap bundle version is tracked separately from ingestion-derived bundle versions so a metadata scan result can be cleanly merged into the published state

---

## Extending the bootstrap pattern

The same pattern applies to subsequent onboarding events, not just initial setup:

- **New source system added:** intake form for the new system → LLM generates binding templates → steward approves → published before the metadata scan runs
- **New product family launched:** provide product family name and key components → LLM generates entity scaffolding and draft relationships → steward approves → available before PLM data is ingested
- **Ontology pack upgrade:** new pack version proposes additions → LLM generates tenant-specific delta overlays → steward reviews conflicts and additions → published before the new pack goes live
- **Acquisition integration:** new subsidiary's entity names and system configuration → bootstrap generates alias and routing drafts → steward merges with existing tenant graph
