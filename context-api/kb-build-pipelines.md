# Knowledge Base Build Pipelines

**Version:** 1.0  
**Status:** Architecture reference  
**Scope:** AI-assisted pipelines for constructing industry ontology packs, entity taxonomies, metric definitions, relationship types, and source binding templates

---

## Design goals

Industry packs cannot be authored from scratch by a single LLM pass — the output is too variable, too incomplete, and not verifiable. The pipeline instead uses:

- **Decomposition:** break each domain into independently scoped sub-problems small enough to extract reliably
- **Multi-pass consensus:** run extraction independently multiple times and treat agreement as signal, disagreement as a steward queue item
- **Schema-constrained output:** every LLM output is a JSON Schema-validated struct, never free text that the system must interpret
- **Evaluation gates:** each stage has explicit pass/fail criteria before output advances to the next stage
- **Reference anchoring:** extraction is grounded in authoritative reference sources, not open-ended generation
- **Steward finalization:** every pack version requires domain expert sign-off before publication

The goal is not zero human involvement — it is to shift human effort from blank-slate authoring to reviewing and correcting structured outputs that are already 80–90% correct.

---

## Pipeline overview

```
Reference corpus assembly
        │
        ▼
Domain decomposition
        │
        ▼
Parallel extraction tracks ──────────────────────────────────────────────┐
  Track A: Entity type extraction + taxonomy                              │
  Track B: Relationship type extraction                                   │
  Track C: Metric formalization                                           │
  Track D: Source binding template generation                             │
        │                                                                 │
        ▼                                                                 │
Cross-track consistency evaluation ◄─────────────────────────────────────┘
        │
        ▼
Steward review and conflict resolution
        │
        ▼
Pack compilation and regression evaluation
        │
        ▼
Publication
```

Each extraction track uses the same underlying pattern: decompose → extract (multiple passes) → compare → flag disagreements → evaluate → advance or queue for steward.

---

## Stage 1: Reference corpus assembly

Before any LLM extraction runs, the pipeline assembles and normalizes a reference corpus for the target domain. Extraction grounded in authoritative sources is significantly more consistent than extraction from model knowledge alone.

### Source categories

| Source type | Examples | Used for |
|---|---|---|
| Process reference frameworks | APICS SCOR, VDA, ODETTE | Entity types, process vocabulary, relationship patterns |
| Data standards bodies | GS1, EDIFACT, ANSI X12 | Identifier formats, transaction entity types |
| ERP system documentation | SAP S/4HANA CDS view catalog, Oracle SCM data model docs | Source binding templates, field definitions |
| Industry-specific standards | JEDEC (semiconductor), IATF 16949 (automotive), ICH Q7 (pharma) | Domain-specific entity subtypes and terminology |
| Academic/open ontologies | Schema.org, GoodRelations, RosettaNet PIPs | Baseline concept structures for cross-reference |
| Analyst and practitioner glossaries | APICS dictionary, Gartner glossaries, domain textbooks | Metric definitions, terminology disambiguation |

### Corpus normalization

Each source document is processed into a normalized corpus record:

```json
{
  "corpus_id": "apics-scor-v13",
  "source_type": "process_framework",
  "domain": "supply_chain",
  "authority_class": "INDUSTRY_STANDARD",
  "sections": [
    {
      "section_id": "scor-plan",
      "title": "Plan process category",
      "content": "...",
      "entity_hints": ["demand_forecast", "supply_plan", "inventory_policy"],
      "relationship_hints": ["ALIGNS_WITH", "DRIVES"]
    }
  ],
  "version": "13.0",
  "ingested_at": "2026-06-01"
}
```

`authority_class` controls how disagreements are resolved: a concept found in an `INDUSTRY_STANDARD` source overrides a concept found only in `PRACTITIONER_GLOSSARY` sources.

---

## Stage 2: Domain decomposition

Before extraction, the domain is decomposed into a tree of non-overlapping, jointly exhaustive sub-domains. Each sub-domain is small enough to extract reliably in a single focused pass.

### Decomposition prompt pattern

```
Given the domain "{domain}" with scope "{scope_statement}",
decompose it into a set of sub-domains where:
- each sub-domain has a clear, single-sentence scope boundary
- sub-domains are non-overlapping
- together they cover the full domain
- each sub-domain is small enough to enumerate its primary concepts exhaustively

Output a JSON array. Each element: { "sub_domain": str, "scope": str, "primary_concept_types": [str] }
```

### Decomposition validation

The decomposition output is validated before extraction begins:

- **Coverage check:** sample 20 known domain concepts and verify each falls in exactly one sub-domain
- **Overlap check:** for each pair of sub-domains, generate 5 concept candidates and verify neither sub-domain claims both
- **Depth check:** no sub-domain should yield more than 20–30 entity type candidates; if it does, decompose further

If validation fails, the decomposer runs again with the failure cases as negative examples.

### Example: supply chain decomposition

```json
[
  { "sub_domain": "procurement", "scope": "acquiring goods and services from external suppliers", "primary_concept_types": ["entity", "transaction", "status", "metric"] },
  { "sub_domain": "inventory_management", "scope": "tracking quantities and locations of materials and finished goods", "primary_concept_types": ["entity", "location", "policy", "metric"] },
  { "sub_domain": "supplier_management", "scope": "governing the relationship with and performance of external supply partners", "primary_concept_types": ["entity", "metric", "agreement"] },
  { "sub_domain": "demand_planning", "scope": "forecasting and communicating future requirements to the supply chain", "primary_concept_types": ["entity", "metric", "process"] },
  { "sub_domain": "logistics_and_fulfillment", "scope": "moving goods from origin to destination including carrier and shipment management", "primary_concept_types": ["entity", "transaction", "metric"] },
  { "sub_domain": "bill_of_materials", "scope": "structured composition relationships between products, assemblies, and components", "primary_concept_types": ["entity", "relationship", "version"] }
]
```

---

## Stage 3: Entity type extraction (Track A)

For each sub-domain, entity types are extracted in three independent passes. Consensus is computed across passes.

### Extraction pass prompt pattern

```
Sub-domain: {sub_domain}
Scope: {scope}
Reference sections: {relevant_corpus_sections}

Extract all primary entity types for this sub-domain.
An entity type is a distinct, named class of business object that:
- has a stable identity (can be referenced by an ID)
- persists over time
- has attributes and participates in relationships
- is relevant to an enterprise agent resolving business questions

For each entity type output:
{
  "entity_type_id": "snake_case_identifier",
  "label": "Human readable name",
  "definition": "One-sentence precise definition",
  "examples": ["example1", "example2"],
  "is_a": "parent_entity_type_id or null",
  "source_references": ["corpus_id:section_id"]
}

Do not output: processes, attributes, statuses (unless a status has its own identity), abstract concepts.
```

### Three-pass consensus

Each sub-domain runs three independent extraction passes. A consensus record is built:

```json
{
  "entity_type_id": "purchase_order",
  "passes_proposed": 3,
  "passes_agreed": 3,
  "consensus": "FULL",
  "definition_variants": [
    { "pass": 1, "definition": "A formal commitment to purchase a specified quantity from a supplier at agreed terms." },
    { "pass": 2, "definition": "A legally binding document issued to a supplier committing to purchase goods or services." },
    { "pass": 3, "definition": "A buyer-issued document authorizing a supplier to deliver goods at agreed price and quantity." }
  ],
  "canonical_definition": null,
  "status": "CONSENSUS_REQUIRES_DEFINITION_MERGE"
}
```

| Consensus level | Condition | Action |
|---|---|---|
| FULL | All 3 passes propose the concept with consistent definition | Auto-advance to taxonomy stage |
| MAJORITY | 2 of 3 passes agree | Advance with flag; steward reviews definition |
| MINORITY | Only 1 pass proposes | Queue for steward review; do not auto-advance |
| CONFLICT | Proposed by all 3 but with contradictory `is_a` | Block; steward resolves taxonomy placement |

### Definition merge pass

Where FULL consensus exists but definitions vary in wording, a merge pass produces the canonical definition:

```
Given these three definitions of "{entity_type_id}", produce a single canonical definition that:
- is precise and unambiguous
- covers all aspects present in any of the three definitions
- is one sentence, under 25 words
- uses the vocabulary of {domain} practitioners, not technical jargon

Definition 1: ...
Definition 2: ...
Definition 3: ...
```

The merge pass runs once. The output is accepted if all three original definitions are strictly subsumed by it; otherwise the item goes to steward review.

---

## Stage 4: Taxonomy construction (Track A continued)

After entity types are extracted and stabilized, the `is_a` hierarchy is constructed and validated.

### Pairwise is-a resolution

For every pair (A, B) where either extraction pass proposed `A is_a B` or `B is_a A`:

```
Is "{entity_type_a}" a subtype of "{entity_type_b}"?
A is a subtype of B if every instance of A is necessarily an instance of B.
Answer only: YES, NO, or AMBIGUOUS. Then provide a one-sentence justification.
```

Run this query three times. Majority vote determines the edge. AMBIGUOUS results go to steward.

### Structural validation

After the hierarchy is assembled, run automated checks:

- **Acyclicity:** detect and reject any cycle
- **Single inheritance:** flag any entity type with more than one `is_a` parent for steward review (multiple inheritance is permitted but must be explicit)
- **Orphan detection:** any entity type with no `is_a` and not a root concept is flagged
- **Depth check:** hierarchies deeper than 5 levels are reviewed for over-specificity
- **Sibling coherence:** for each parent, verify siblings are truly parallel alternatives, not one being a subtype of another

### Example taxonomy fragment (semiconductor extension)

```
component
  └── electronic_component
        ├── memory_component
        │     ├── dram
        │     │     └── hbm
        │     │           ├── hbm2e
        │     │           └── hbm3e
        │     └── nand_flash
        ├── logic_component
        │     ├── gpu_die
        │     └── cpu_die
        └── passive_component
```

---

## Stage 5: Relationship type extraction (Track B)

Relationship types are extracted by examining entity type pairs, not by open-ended extraction from text. This significantly constrains the output space.

### Candidate pair generation

Generate the candidate entity type pairs to evaluate:

```
Given this list of entity types: [...]
Generate all entity type pairs (A, B) where a directional relationship
"A --[VERB]--> B" is plausible in the context of {domain}.
Exclude pairs where no meaningful business relationship exists.
Output: [{ "from": "entity_type_a", "to": "entity_type_b" }]
```

This narrows the evaluation set from O(n²) to a manageable list.

### Relationship extraction per pair

For each candidate pair:

```
Entity A: {entity_type_a} — {definition}
Entity B: {entity_type_b} — {definition}
Domain: {domain}

What directional relationships can exist between A and B?
For each relationship:
{
  "relationship_type_id": "UPPER_SNAKE_CASE",
  "label": "Human readable verb phrase",
  "direction": "A_TO_B or B_TO_A or BOTH",
  "cardinality": "ONE_TO_ONE | ONE_TO_MANY | MANY_TO_MANY",
  "temporal": true/false,
  "definition": "one sentence",
  "example": "Concrete example using real-world entities",
  "source_references": [...]
}

If no meaningful relationship exists, output an empty array.
```

Three independent passes per pair. Consensus rules same as entity extraction.

### Relationship validation

- **Directionality consistency:** if A --R--> B is extracted, verify B --R--> A is not also extracted (unless cardinality is BOTH)
- **Transitivity check:** for each relationship type, determine whether it is transitive; flag transitive relationships for explicit declaration (they affect closure computation)
- **Redundancy check:** if `A --SUPPLIES--> B` and `A --PROVIDES--> B` both exist, a merge pass determines whether they are synonyms, distinct, or one is a subtype

---

## Stage 6: Metric formalization (Track C)

Metrics require more structure than entity types. The extraction template enforces precision.

### Metric candidate collection

Before formalization, collect metric candidates from:

1. Reference corpus named metrics (SCOR KPIs, Gartner supply chain metrics)
2. ERP system standard reports and field names (SAP standard analyses, Snowflake warehouse views)
3. Practitioner glossaries
4. Common agent questions from domain research ("what metrics do supply chain planners ask about most?")

Deduplicate by running pairwise synonym checks:

```
Are "{metric_a}" and "{metric_b}" the same metric with different names,
overlapping but distinct metrics, or unrelated metrics?
Answer: SYNONYM | OVERLAPPING | DISTINCT
If SYNONYM: which name is more canonical in {domain} practitioner usage?
If OVERLAPPING: what is the precise difference in scope or calculation?
```

### Metric formalization template

```json
{
  "metric_id": "open_po_quantity",
  "canonical_name": "Open Purchase Order Quantity",
  "definition": "The total quantity on purchase orders issued to suppliers that have not yet been fully goods-receipted.",
  "grain": ["supplier", "material", "purchase_org", "date"],
  "time_semantics": "POINT_IN_TIME",
  "standard_dimensions": ["supplier", "material_group", "plant", "currency"],
  "standard_filters": {
    "include_statuses": ["open", "partially_received"],
    "exclude_deletion_flag": true
  },
  "do_not_confuse_with": [
    {
      "metric_id": "demand_backlog",
      "distinction": "open_po_quantity tracks supplier-side commitments; demand_backlog tracks unfulfilled customer orders"
    }
  ],
  "source_system_expressions": {
    "sap_s4hana": {
      "objects": ["PurchaseOrderItem"],
      "key_fields": ["Supplier", "Material", "OrderQuantity", "OpenQuantity", "DeliveryCompletionFlag"],
      "standard_filter": "DeliveryCompletionFlag = false AND DeletionIndicator = false"
    },
    "snowflake_generic": {
      "pattern": "SUM(open_quantity) from purchase_order_items where status IN ('open', 'partial')"
    }
  },
  "owner_persona": "procurement_analyst",
  "risk_tier_default": "PLANNING_DECISION",
  "source_references": ["apics-scor-v13:rs.3.2", "sap-mm-docs:me2m"]
}
```

### Metric evaluation checks

- **Grain completeness:** verify all dimensions reference defined entity types
- **Disambiguation quality:** for each `do_not_confuse_with` entry, verify the distinction is testable (can you construct a question where both metrics are plausible answers?)
- **Source expression correctness:** source expressions are tested against the binding template library for syntactic validity

---

## Stage 7: Source binding template generation (Track D)

Source binding templates translate the connector-neutral metric and operation definitions into parameterized templates for known ERP and data systems. This is the most deterministic track because ERP system schemas are documented and stable.

### Template generation approach

Rather than open-ended LLM extraction, binding templates are generated from:

1. **ERP schema documentation** (SAP CDS view catalog, Oracle table definitions) — parsed and ingested as structured corpus
2. **Template library** — a set of known patterns (SAP MM procurement template, Snowflake analytical template) that are parameterized, not generated
3. **LLM gap filling** — for fields or conditions not covered by the documented schema, the LLM proposes a candidate that is then validated against a test system or flagged for DBA review

### Template structure

```json
{
  "template_id": "sap_mm_open_po_v3",
  "target_system_type": "sap_s4hana",
  "operations_covered": ["procurement.open_purchase_order_quantity"],
  "template_parameters": {
    "required": ["company_codes", "purchasing_orgs"],
    "optional": ["material_groups", "exclude_reason_codes"]
  },
  "cds_view": "C_PurchaseOrderItemApi",
  "key_fields": {
    "supplier": "Supplier",
    "material": "Material",
    "open_quantity": "OpenPurchaseOrderQuantity",
    "currency": "DocumentCurrency",
    "delivery_date": "ScheduleLineDeliveryDate"
  },
  "standard_filters": [
    "DeliveryIsCompleted = false",
    "PurchaseOrderDeletionIndicator = false"
  ],
  "authorization_objects": ["M_BEST_BSA", "M_BEST_EKO"],
  "api_type": "OData_V4",
  "sap_versions_tested": ["2023", "2022", "2021"],
  "schema_version": "v3",
  "source_references": ["sap-s4-api-hub:purchaseorderitem"]
}
```

### Template validation

Templates are validated by running them against a sandbox or synthetic SAP/Snowflake instance before inclusion in the pack. A template that cannot be executed against the target system type does not ship.

---

## Stage 8: Cross-track consistency evaluation

After all four tracks complete, the assembled ontology is evaluated for internal consistency before steward review.

### Automated checks

| Check | Description | Failure action |
|---|---|---|
| Entity coverage | Every relationship endpoint references a defined entity type | Block; flag undefined entity |
| Metric grain coverage | All metric grain dimensions reference defined entity types | Block; flag undefined entity |
| Binding completeness | Every canonical operation has at least one binding template | Warn; flag for steward |
| Relationship completeness | Every entity type participates in at least one relationship | Warn; flag isolated entity |
| Taxonomy consistency | No entity type appears in multiple non-overlapping taxonomy branches | Block; flag conflict |
| Metric disambiguation | For every metric pair marked do_not_confuse_with, verify a discrimination test passes | Warn; flag for steward |
| Circular relationship | No directed relationship path forms a cycle through the same entity type twice | Block; flag cycle |
| Version reference integrity | All source_references point to ingested corpus records | Block; flag missing reference |

### Benchmark question evaluation

A benchmark question set is maintained for each domain. Before any pack version is published, the assembled ontology is evaluated against these questions:

```json
{
  "question": "What is the difference between backlog and open PO?",
  "expected_outcome": "disambiguated to demand_backlog vs open_po_quantity with distinct definitions",
  "evaluation_type": "metric_disambiguation"
},
{
  "question": "Which entity type represents a supplier's promised delivery quantity?",
  "expected_outcome": "committed_supply metric or supplier_commitment entity",
  "evaluation_type": "entity_resolution"
},
{
  "question": "What is the path from a supplier to a finished product?",
  "expected_outcome": "supplier --SUPPLIES--> component --USED_IN--> bom --BUILDS--> sku",
  "evaluation_type": "relationship_traversal"
}
```

A pack version may not advance to steward review if it fails more than 5% of benchmark questions, or if it fails any benchmark question in the `relationship_traversal` category (these are high-confidence structure questions).

---

## Stage 9: Steward review

The steward review interface presents the assembled ontology in structured review sections, not as raw JSON.

### Review workflow per section

```
Proposed concept presented to steward
        │
   Steward action:
        ├── APPROVE  →  status: APPROVED
        ├── MODIFY   →  steward edits definition/hierarchy; status: APPROVED_WITH_MODIFICATION
        ├── REJECT   →  concept excluded from pack; reason logged
        └── DEFER    →  held for future version; status: DEFERRED
```

### Steward review priorities

The review queue is ordered by:

1. **Conflicts and MINORITY consensus** items first — these require the most judgment
2. **New entity types not present in previous pack version** — additive changes
3. **Modified definitions for existing entity types** — breaking changes that affect existing tenants
4. **New relationship types** — structural changes
5. **New metrics** — additive
6. **Binding template changes** — reviewed by integration engineer, not domain expert

### Domain expert vs. integration engineer review split

| Artifact type | Reviewer |
|---|---|
| Entity type definitions and taxonomy | Domain expert (supply chain, semiconductor) |
| Relationship types and cardinality | Domain expert |
| Metric definitions and disambiguation | Domain expert + business analyst |
| Source expressions in metric definitions | Integration engineer |
| Binding templates | Integration engineer |
| Authorization objects and API versions | Integration engineer |

---

## Stage 10: Pack compilation and publication

### Pack version structure

```
supply_chain_pack/
  version: 1.4
  ontology/
    entity_types.json          # all entity types and taxonomy
    relationship_types.json    # all relationship type definitions
  metrics/
    metric_definitions.json    # all metric definitions
    disambiguation_pairs.json  # do_not_confuse_with index
  bindings/
    sap_s4hana/
      template_*.json
    snowflake/
      template_*.json
    oracle_scm/
      template_*.json
  evaluation/
    benchmark_questions.json
    benchmark_results.json     # must pass before publication
  manifest.json                # version, digests, signatures, compatibility
```

### Compilation steps

1. Validate all steward decisions are recorded (no items left in `PENDING_REVIEW`)
2. Run consistency checks (Stage 8) against the steward-approved set
3. Run benchmark evaluation (Stage 8) against the steward-approved set — must pass
4. Serialize to canonical form (deterministic key ordering, UTF-8, no trailing whitespace)
5. Compute SHA-256 digest of each artifact file
6. Compile `manifest.json` with all artifact digests and compatibility metadata
7. Sign manifest with pack-compiler key
8. Publish to object storage; update PostgreSQL pack registry

### Compatibility metadata

```json
{
  "pack_id": "supply_chain_base",
  "version": "1.4",
  "min_compatible_version": "1.0",
  "breaking_changes": [],
  "additive_changes": [
    "entity_type: consignment_stock added",
    "metric: supplier_otd_rate added"
  ],
  "deprecated": [
    "metric: backlog_aging (use demand_backlog_by_age instead)"
  ]
}
```

Breaking changes (entity type removed, metric definition semantically changed, relationship cardinality changed) require a major version increment and a tenant migration review before activation.

---

## Pack hierarchy and extension pattern

Industry packs are layered. Each layer extends the one above it without mutating it.

```
Global foundation ontology  (organization, supplier, product, order, facility)
        │
        └── Supply chain base pack  (procurement, inventory, logistics, BOM, demand)
                │
                ├── Semiconductor extension  (fab, die, HBM, wafer, process node, allocation)
                │         │
                │         └── Tenant overlay  (NVIDIA-specific aliases, metric overrides, source bindings)
                │
                ├── Automotive extension  (VDA, IATF, multi-tier BOM, recall, homologation)
                │
                └── Pharma extension  (ICH, batch, serialization, cold chain, regulatory)
```

### Extension pipeline

An extension pack runs the same pipeline as a base pack, with one additional input: the parent pack's entity types, relationship types, and metrics are provided as context so the extension can reference, subtype, or extend them rather than duplicate them.

The extraction prompt is modified:

```
You are building an extension to the supply chain base pack.
The following entity types already exist in the base pack: [...]
The following relationship types already exist: [...]

Extract ONLY concepts that are:
- specific to the semiconductor domain AND
- not already covered by the base pack concepts above

For concepts that extend a base pack concept, output:
{ ..., "extends": "base_pack_entity_type_id" }
```

---

## Continuous improvement loop

The pack is not static. It improves through:

| Signal | Source | Pipeline action |
|---|---|---|
| Unresolved tenant terms | Runtime resolver (terms not matching any alias or ontology concept) | Aggregated weekly; proposed as candidate additions |
| Steward corrections | Production steward overrides | Added to benchmark question set; re-evaluate affected pack sections |
| Metric disambiguation failures | Production clarification requests for known metrics | Review do_not_confuse_with entries; add disambiguation guidance |
| New ERP versions | SAP/Oracle release notes | Binding template compatibility review; update tested versions |
| New industry standards | APICS SCOR revisions, GS1 updates | Corpus update; re-run affected sub-domain extraction |
| Customer-contributed aliases | Tenant overlays that are common across multiple tenants | Candidate promotion to base pack or extension |

Proposed additions from production signals enter the same pipeline as new concepts: extraction → consensus → steward review → compilation → publication. They do not bypass any stage.

---

## Pipeline tooling requirements

| Requirement | Implementation note |
|---|---|
| Multi-pass extraction with deterministic sampling | Temperature 0; same prompt template across all passes; log all pass outputs |
| Schema-constrained LLM output | JSON Schema validation on every LLM response; reject and retry on schema violation (max 3 retries before logging as extraction failure) |
| Consensus computation | Exact string match for `entity_type_id`; semantic similarity (embedding cosine > 0.92) for definition comparison |
| Reference corpus indexing | Chunked, embedded corpus with citation retrieval so every extraction prompt cites specific corpus sections |
| Benchmark evaluation harness | Run against the current ontology as a ContextGraph instance; measure resolution accuracy, relationship completeness, metric disambiguation accuracy |
| Steward review interface | Structured diff view against previous pack version; bulk approve for FULL consensus items; item-by-item for conflicts |
| Pack registry | PostgreSQL-backed manifest registry with digest verification, activation pointers, and rollback support |
