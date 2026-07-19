# KB Build Pipeline — Prompt Architecture

**Version:** 1.0  
**Status:** Architecture reference  
**Scope:** How the KB build pipeline decomposes into discrete, chainable prompt calls with defined input/output schemas

---

## Design rules for all prompts

- Temperature 0 on every prompt that produces structured output fed into downstream stages
- Every prompt outputs a single JSON object matching a declared schema — no free text
- Schema validation runs before the output is accepted; on failure, retry up to 3 times with the validation error appended to the prompt
- Each prompt has a single responsibility; no prompt extracts and classifies and validates simultaneously
- System prompt carries domain context and output schema; user message carries the specific inputs
- All prompts are versioned; a prompt change is treated as a pipeline change and triggers re-evaluation

---

## Prompt catalog

Prompts are grouped by pipeline stage. Each entry shows: inputs, output schema, model settings, validation, and how it chains.

---

### Group COR — Corpus processing

---

#### P-COR-01: Section relevance scoring

**Purpose:** Given a corpus section and a sub-domain, score how relevant the section is so extraction prompts receive only high-signal content.

**Inputs:**
```
section_title: str
section_content: str (truncated to 800 tokens)
sub_domain: str
sub_domain_scope: str
```

**System prompt:**
```
You are classifying the relevance of a document section to a specific business domain.
Output only valid JSON matching the schema below.
Schema: { "relevance": "HIGH" | "MEDIUM" | "LOW" | "NONE", "reason": str (max 20 words), "concept_hints": [str] }
```

**User message:**
```
Sub-domain: {{sub_domain}}
Scope: {{sub_domain_scope}}

Section title: {{section_title}}
Section content: {{section_content}}

Score the relevance of this section to the sub-domain.
In concept_hints, list any named business concepts you notice in this section.
```

**Output schema:**
```json
{
  "relevance": "HIGH",
  "reason": "Section defines procurement KPIs directly relevant to purchasing sub-domain",
  "concept_hints": ["purchase_order", "supplier", "open_quantity", "goods_receipt"]
}
```

**Validation:** `relevance` must be one of the four enum values. `concept_hints` max 10 items.  
**Chains to:** P-COR-02 for HIGH/MEDIUM sections only.  
**Model settings:** temperature 0, max_tokens 300.

---

#### P-COR-02: Entity hint extraction from corpus section

**Purpose:** Extract named business concepts from a corpus section as unstructured hints that seed the entity extraction stage.

**Inputs:**
```
section_content: str
sub_domain: str
existing_entity_hints: [str]  # from previous sections, to avoid duplicates
```

**System prompt:**
```
You extract named business entity concepts from supply chain and enterprise domain text.
An entity concept is a named class of business object with stable identity.
Do not extract: processes, verbs, adjectives, metrics, statuses, or abstract principles.
Output only valid JSON matching the schema below.
Schema: { "entity_hints": [{ "term": str, "definition_fragment": str (max 15 words), "source_sentence": str }] }
```

**User message:**
```
Sub-domain: {{sub_domain}}
Already found in previous sections: {{existing_entity_hints}}

Section text:
{{section_content}}

Extract entity concepts not already in the existing list.
```

**Output schema:**
```json
{
  "entity_hints": [
    {
      "term": "Purchase Order",
      "definition_fragment": "formal commitment to purchase from a supplier",
      "source_sentence": "A purchase order is issued by the buyer to authorize the supplier..."
    }
  ]
}
```

**Validation:** `term` must not be empty; `definition_fragment` max 15 words enforced.  
**Chains to:** P-ENT-01 (entity hints become seeds for structured extraction).  
**Model settings:** temperature 0, max_tokens 600.

---

### Group DEC — Domain decomposition

---

#### P-DEC-01: Domain decomposition

**Purpose:** Split a domain into non-overlapping, jointly exhaustive sub-domains.

**Inputs:**
```
domain: str
scope_statement: str
target_sub_domain_count: int  # 4-8 recommended
```

**System prompt:**
```
You decompose enterprise business domains into sub-domains for ontology extraction.
Each sub-domain must have a clear single-sentence scope boundary.
Sub-domains must be non-overlapping and together cover the full domain.
Output only valid JSON matching the schema below.
Schema: { "sub_domains": [{ "id": str, "label": str, "scope": str, "rationale": str }] }
```

**User message:**
```
Domain: {{domain}}
Scope: {{scope_statement}}
Target sub-domain count: {{target_sub_domain_count}}

Decompose this domain. Each sub-domain scope must be stated as a single sentence
beginning with a gerund (e.g. "Tracking quantities and locations of...").
```

**Output schema:**
```json
{
  "sub_domains": [
    {
      "id": "procurement",
      "label": "Procurement",
      "scope": "Acquiring goods and services from external suppliers through purchase orders.",
      "rationale": "Procurement has distinct entity types (PO, supplier) and metrics (open PO qty) not shared with other sub-domains."
    }
  ]
}
```

**Validation:** `id` must be snake_case; `scope` must start with a gerund; no duplicate IDs.  
**Chains to:** P-DEC-02 (coverage validation), P-DEC-03 (overlap check).  
**Model settings:** temperature 0, max_tokens 1000.

---

#### P-DEC-02: Sub-domain coverage validation

**Purpose:** Verify a candidate concept falls in exactly one sub-domain. Run against 20 sampled concepts.

**Inputs:**
```
concept: str
sub_domains: [{ id, label, scope }]
```

**System prompt:**
```
You assign a business concept to the most appropriate sub-domain from a provided list.
Output only valid JSON matching the schema below.
Schema: { "assigned_sub_domain_id": str, "confidence": "HIGH" | "MEDIUM" | "LOW", "reasoning": str (max 20 words) }
If the concept fits multiple sub-domains equally, set confidence to LOW.
```

**User message:**
```
Concept: {{concept}}

Sub-domains:
{{sub_domains_formatted}}

Which single sub-domain best contains this concept?
```

**Output schema:**
```json
{
  "assigned_sub_domain_id": "procurement",
  "confidence": "HIGH",
  "reasoning": "Purchase order is a procurement transaction entity by definition."
}
```

**Validation:** `assigned_sub_domain_id` must exist in the provided sub-domain list.  
**Chains to:** If any concept scores LOW confidence, feeds back to P-DEC-01 as a failure example.  
**Model settings:** temperature 0, max_tokens 200.

---

#### P-DEC-03: Sub-domain overlap check

**Purpose:** For each pair of adjacent or related sub-domains, verify they do not claim the same concept space.

**Inputs:**
```
sub_domain_a: { id, label, scope }
sub_domain_b: { id, label, scope }
```

**System prompt:**
```
You detect overlap between two business sub-domain scopes.
Output only valid JSON matching the schema below.
Schema: { "overlap": "NONE" | "PARTIAL" | "SIGNIFICANT", "overlap_description": str | null, "suggested_resolution": str | null }
```

**User message:**
```
Sub-domain A: {{sub_domain_a.label}} — {{sub_domain_a.scope}}
Sub-domain B: {{sub_domain_b.label}} — {{sub_domain_b.scope}}

Is there conceptual overlap between these two sub-domains?
If PARTIAL or SIGNIFICANT, describe what concepts could be claimed by both.
```

**Output schema:**
```json
{
  "overlap": "PARTIAL",
  "overlap_description": "Supplier master data could belong to either procurement or supplier_management.",
  "suggested_resolution": "Assign supplier master to supplier_management; procurement references it."
}
```

**Chains to:** Significant overlaps feed back to P-DEC-01.  
**Model settings:** temperature 0, max_tokens 300.

---

### Group ENT — Entity extraction

---

#### P-ENT-01: Entity type extraction (single pass)

**Purpose:** Extract structured entity type definitions for one sub-domain from corpus sections. Called three times independently per sub-domain.

**Inputs:**
```
sub_domain: str
sub_domain_scope: str
relevant_corpus_sections: [{ title, content }]  # HIGH/MEDIUM from P-COR-01
entity_hints: [str]  # from P-COR-02, used as seeds not constraints
pass_number: int  # 1, 2, or 3 — included in prompt for audit, not logic
```

**System prompt:**
```
You extract structured entity type definitions from enterprise domain documentation.
An entity type is a named class of business object that:
  - has a stable identity (referenced by an ID)
  - persists over time
  - has attributes
  - participates in relationships
  - is relevant to enterprise agents answering business questions

Do NOT extract: processes, status codes, attributes, abstract principles, metrics.

Output only valid JSON matching the schema below.
Schema: {
  "entity_types": [{
    "entity_type_id": str,         // snake_case
    "label": str,                  // human-readable
    "definition": str,             // one sentence, max 25 words
    "examples": [str],             // 2-3 real-world examples
    "is_a": str | null,            // parent entity_type_id or null
    "source_ref": str              // corpus section title
  }]
}
```

**User message:**
```
Sub-domain: {{sub_domain}}
Scope: {{sub_domain_scope}}
Hint terms to consider (not exhaustive): {{entity_hints}}

Reference sections:
{{relevant_corpus_sections_formatted}}

Extract all entity types for this sub-domain.
```

**Output schema:**
```json
{
  "entity_types": [
    {
      "entity_type_id": "purchase_order",
      "label": "Purchase Order",
      "definition": "A formal commitment issued by a buyer to a supplier to deliver goods at agreed terms.",
      "examples": ["SAP PO 4500012345", "Oracle PO-2026-00891"],
      "is_a": "procurement_document",
      "source_ref": "apics-scor-v13:rs.3.2"
    }
  ]
}
```

**Validation:** `entity_type_id` must be snake_case; `definition` max 25 words; `examples` 2-3 items.  
**Chains to:** P-ENT-02 (consensus across 3 passes).  
**Model settings:** temperature 0, max_tokens 2000.

---

#### P-ENT-02: Extraction consensus comparison

**Purpose:** Compare three extraction pass outputs and classify each concept by consensus level.

**Inputs:**
```
entity_type_id: str
pass_1_entry: { entity_type_id, label, definition, is_a, source_ref } | null
pass_2_entry: { ... } | null
pass_3_entry: { ... } | null
```

**System prompt:**
```
You compare three independently extracted entity type definitions and determine consensus.
Output only valid JSON matching the schema below.
Schema: {
  "entity_type_id": str,
  "consensus": "FULL" | "MAJORITY" | "MINORITY" | "CONFLICT",
  "is_a_consensus": "FULL" | "MAJORITY" | "MINORITY" | "CONFLICT" | "NA",
  "definition_variance": "LOW" | "MEDIUM" | "HIGH",
  "notes": str | null
}
FULL = all present passes agree on concept existence and is_a placement.
MAJORITY = 2 of 3 agree.
MINORITY = only 1 proposes it.
CONFLICT = all propose it but with contradictory is_a.
```

**User message:**
```
Entity type: {{entity_type_id}}

Pass 1: {{pass_1_entry | "not proposed"}}
Pass 2: {{pass_2_entry | "not proposed"}}
Pass 3: {{pass_3_entry | "not proposed"}}

Classify the consensus on this entity type.
```

**Output schema:**
```json
{
  "entity_type_id": "purchase_order",
  "consensus": "FULL",
  "is_a_consensus": "FULL",
  "definition_variance": "LOW",
  "notes": null
}
```

**Chains to:** FULL + LOW variance → P-ENT-03 (definition merge). MAJORITY → P-ENT-03 + steward flag. MINORITY/CONFLICT → steward queue directly.  
**Model settings:** temperature 0, max_tokens 300.

---

#### P-ENT-03: Definition merge

**Purpose:** Produce a single canonical definition from 2-3 variant definitions that passed consensus.

**Inputs:**
```
entity_type_id: str
label: str
definitions: [str]  # 2 or 3 variants
domain: str
```

**System prompt:**
```
You merge multiple definitions of the same business concept into one canonical definition.
The canonical definition must:
  - be one sentence
  - be under 25 words
  - subsume all meaning present in any of the input definitions
  - use practitioner vocabulary for the domain, not technical jargon
  - be unambiguous

Output only valid JSON matching the schema below.
Schema: { "canonical_definition": str, "fully_subsumes_all_inputs": bool, "notes": str | null }
If you cannot produce a definition that fully subsumes all inputs, set fully_subsumes_all_inputs to false.
```

**User message:**
```
Concept: {{label}} ({{entity_type_id}})
Domain: {{domain}}

Definition variants:
{{definitions_formatted}}

Produce a single canonical definition.
```

**Output schema:**
```json
{
  "canonical_definition": "A formal commitment issued by a buyer to a supplier to deliver specified goods or services at agreed price and quantity.",
  "fully_subsumes_all_inputs": true,
  "notes": null
}
```

**Validation:** `canonical_definition` max 30 words; `fully_subsumes_all_inputs` must be true to auto-advance.  
**Chains to:** Accepted definitions populate the entity type record. If `fully_subsumes_all_inputs` is false, goes to steward queue.  
**Model settings:** temperature 0, max_tokens 200.

---

### Group TAX — Taxonomy construction

---

#### P-TAX-01: Pairwise is-a check

**Purpose:** For a proposed parent-child pair, determine whether the is-a relationship holds.

**Inputs:**
```
child_type: { entity_type_id, label, definition }
parent_type: { entity_type_id, label, definition }
domain: str
```

**System prompt:**
```
You determine whether one business entity type is a subtype of another.
A is a subtype of B if and only if every instance of A is necessarily an instance of B.
Output only valid JSON matching the schema below.
Schema: { "is_subtype": true | false | null, "justification": str (max 20 words) }
Use null only if the relationship is genuinely ambiguous given the definitions provided.
```

**User message:**
```
Domain: {{domain}}

Candidate child: {{child_type.label}} — "{{child_type.definition}}"
Candidate parent: {{parent_type.label}} — "{{parent_type.definition}}"

Is "{{child_type.label}}" a subtype of "{{parent_type.label}}"?
```

**Output schema:**
```json
{
  "is_subtype": true,
  "justification": "Every purchase order item is necessarily part of a purchase order."
}
```

**Called:** 3 times per pair. Majority vote determines edge. null counts as abstain.  
**Chains to:** Confirmed edges build the taxonomy DAG, validated by P-TAX-02.  
**Model settings:** temperature 0, max_tokens 150.

---

#### P-TAX-02: Sibling coherence check

**Purpose:** Given a parent entity type and its proposed children, verify the siblings are parallel alternatives at the same level of abstraction.

**Inputs:**
```
parent_type: { entity_type_id, label, definition }
proposed_children: [{ entity_type_id, label, definition }]
```

**System prompt:**
```
You verify that a set of entity type siblings form a coherent partition under their parent.
Problems to detect:
  1. One sibling is actually a subtype of another sibling (misplaced in hierarchy)
  2. Siblings are at different levels of abstraction (mixing general and specific)
  3. Siblings overlap significantly in meaning

Output only valid JSON matching the schema below.
Schema: {
  "coherent": bool,
  "issues": [{
    "type": "SUBTYPE_OF_SIBLING" | "ABSTRACTION_MISMATCH" | "OVERLAP",
    "entity_type_ids": [str],
    "description": str (max 20 words)
  }]
}
```

**User message:**
```
Parent: {{parent_type.label}} — "{{parent_type.definition}}"

Proposed siblings:
{{proposed_children_formatted}}

Are these valid parallel subtypes of the parent?
```

**Output schema:**
```json
{
  "coherent": false,
  "issues": [
    {
      "type": "SUBTYPE_OF_SIBLING",
      "entity_type_ids": ["hbm3e", "memory_component"],
      "description": "hbm3e is a specific subtype of memory_component, not a sibling."
    }
  ]
}
```

**Chains to:** Issues go to steward queue with specific correction guidance.  
**Model settings:** temperature 0, max_tokens 500.

---

### Group REL — Relationship extraction

---

#### P-REL-01: Candidate pair generation

**Purpose:** From the full entity type list, generate the candidate pairs worth evaluating for relationships. Avoids O(n²) full evaluation.

**Inputs:**
```
entity_types: [{ entity_type_id, label, definition }]
domain: str
```

**System prompt:**
```
You identify entity type pairs that likely have meaningful directional business relationships.
A pair is worth evaluating if a directional verb relationship between them is plausible
in real enterprise operations (not theoretical or trivial identity relationships).
Do NOT include: pairs where one is_a the other (handled by taxonomy), self-referential pairs.

Output only valid JSON matching the schema below.
Schema: { "candidate_pairs": [{ "from_type": str, "to_type": str, "likely_verb": str }] }
likely_verb is a one-word hint (e.g. "supplies", "contains", "references") — not authoritative.
```

**User message:**
```
Domain: {{domain}}

Entity types:
{{entity_types_formatted}}

Generate candidate pairs where a meaningful directional relationship likely exists.
```

**Output schema:**
```json
{
  "candidate_pairs": [
    { "from_type": "supplier", "to_type": "component", "likely_verb": "supplies" },
    { "from_type": "component", "to_type": "bom", "likely_verb": "used_in" }
  ]
}
```

**Chains to:** Each pair goes to P-REL-02.  
**Model settings:** temperature 0, max_tokens 2000.

---

#### P-REL-02: Relationship type extraction per pair

**Purpose:** For a single entity pair, extract all meaningful directional relationship types.

**Inputs:**
```
from_type: { entity_type_id, label, definition }
to_type: { entity_type_id, label, definition }
domain: str
relevant_corpus_sections: [{ title, content }]
```

**System prompt:**
```
You extract directional relationship types between two specific business entity types.
For each relationship:
  - direction must be FROM_TO, TO_FROM, or BIDIRECTIONAL (only if truly symmetric)
  - cardinality must be ONE_TO_ONE, ONE_TO_MANY, or MANY_TO_MANY
  - temporal means the relationship has effective dates that can change over time
  - relationship_type_id must be UPPER_SNAKE_CASE

Output only valid JSON matching the schema below.
Schema: {
  "relationships": [{
    "relationship_type_id": str,
    "label": str,
    "direction": "FROM_TO" | "TO_FROM" | "BIDIRECTIONAL",
    "cardinality": str,
    "temporal": bool,
    "definition": str,
    "example": str,
    "transitive": bool,
    "source_ref": str
  }]
}
If no meaningful relationship exists, return { "relationships": [] }.
```

**User message:**
```
Domain: {{domain}}
From: {{from_type.label}} — "{{from_type.definition}}"
To:   {{to_type.label}} — "{{to_type.definition}}"

Reference:
{{relevant_corpus_sections_formatted}}

What directional business relationships can exist between these two entity types?
```

**Output schema:**
```json
{
  "relationships": [
    {
      "relationship_type_id": "SUPPLIES",
      "label": "supplies",
      "direction": "FROM_TO",
      "cardinality": "ONE_TO_MANY",
      "temporal": true,
      "definition": "A supplier provides a component to a buying organization under active commercial terms.",
      "example": "SK Hynix SUPPLIES HBM3E to NVIDIA",
      "transitive": false,
      "source_ref": "apics-scor-v13:source"
    }
  ]
}
```

**Called:** 3 times per pair. Consensus via P-REL-03.  
**Model settings:** temperature 0, max_tokens 1000.

---

#### P-REL-03: Relationship synonym check

**Purpose:** Detect when two proposed relationship types for the same entity pair are synonyms, overlapping, or genuinely distinct.

**Inputs:**
```
relationship_a: { relationship_type_id, label, definition }
relationship_b: { relationship_type_id, label, definition }
from_type: str
to_type: str
```

**System prompt:**
```
You determine whether two relationship types between the same entity pair are synonyms,
overlapping (one is a subtype of the other), or genuinely distinct.
Output only valid JSON matching the schema below.
Schema: {
  "verdict": "SYNONYM" | "SUBTYPE" | "DISTINCT",
  "canonical_id": str | null,
  "justification": str (max 20 words)
}
If SYNONYM, canonical_id is the more precise or widely-used identifier.
If SUBTYPE, canonical_id is the more general relationship.
```

**Chains to:** SYNONYM/SUBTYPE collapses into one record. DISTINCT both advance.  
**Model settings:** temperature 0, max_tokens 200.

---

### Group MET — Metric formalization

---

#### P-MET-01: Metric candidate collection

**Purpose:** Enumerate all candidate metrics for a domain from the reference corpus.

**Inputs:**
```
domain: str
sub_domains: [str]
relevant_corpus_sections: [{ title, content }]
```

**System prompt:**
```
You extract named business metrics from enterprise domain documentation.
A metric is a named quantitative measure with a precise definition and business meaning.
Do NOT extract: entity types, processes, statuses, or qualitative assessments.

Output only valid JSON matching the schema below.
Schema: { "metric_candidates": [{ "name": str, "definition_fragment": str (max 20 words), "source_ref": str }] }
```

**User message:**
```
Domain: {{domain}}
Sub-domains in scope: {{sub_domains}}

Reference sections:
{{relevant_corpus_sections_formatted}}

List all named metrics mentioned or implied in these sections.
```

**Chains to:** P-MET-02 (synonym check between all pairs), then P-MET-03 (formalization).  
**Model settings:** temperature 0, max_tokens 1500.

---

#### P-MET-02: Metric synonym check

**Purpose:** Determine whether two metric candidates are the same metric under different names.

**Inputs:**
```
metric_a: { name, definition_fragment }
metric_b: { name, definition_fragment }
domain: str
```

**System prompt:**
```
You determine whether two business metrics are synonyms, overlapping, or distinct.
SYNONYM: same underlying measure, different names in different systems or contexts.
OVERLAPPING: related measures that differ in scope, grain, or filter.
DISTINCT: genuinely different measures with different business meanings.

Output only valid JSON matching the schema below.
Schema: {
  "verdict": "SYNONYM" | "OVERLAPPING" | "DISTINCT",
  "canonical_name": str | null,
  "distinction_note": str | null
}
```

**User message:**
```
Domain: {{domain}}

Metric A: "{{metric_a.name}}" — {{metric_a.definition_fragment}}
Metric B: "{{metric_b.name}}" — {{metric_b.definition_fragment}}
```

**Chains to:** SYNONYM collapses to canonical_name. OVERLAPPING/DISTINCT both advance to P-MET-03.  
**Model settings:** temperature 0, max_tokens 200.

---

#### P-MET-03: Metric formalization

**Purpose:** Produce a fully structured metric definition record from a deduped metric candidate.

**Inputs:**
```
metric_name: str
definition_fragment: str
domain: str
entity_types: [{ entity_type_id, label }]  # for grain validation
related_metrics: [{ name, definition_fragment }]  # candidates for do_not_confuse_with
relevant_corpus_sections: [{ title, content }]
```

**System prompt:**
```
You produce a fully structured enterprise metric definition.

Rules:
- grain items must be entity_type_ids from the provided entity type list
- time_semantics: POINT_IN_TIME (snapshot), PERIOD (aggregated over time), or FLOW (rate)
- do_not_confuse_with: only include entries where genuine confusion is likely in practice
- source_system_expressions are illustrative patterns, not executable queries

Output only valid JSON matching the schema below.
Schema: {
  "metric_id": str,
  "canonical_name": str,
  "definition": str,
  "grain": [str],
  "time_semantics": "POINT_IN_TIME" | "PERIOD" | "FLOW",
  "standard_dimensions": [str],
  "standard_filters": { "description": str },
  "do_not_confuse_with": [{ "metric_id": str, "distinction": str }],
  "source_system_expressions": {
    "sap_s4hana": { "pattern": str } | null,
    "snowflake_generic": { "pattern": str } | null,
    "oracle_scm": { "pattern": str } | null
  },
  "owner_persona": str,
  "risk_tier_default": "INFORMATIONAL" | "PLANNING_DECISION" | "OPERATIONAL_ACTION",
  "source_ref": str
}
```

**User message:**
```
Domain: {{domain}}
Metric: {{metric_name}}
Description fragment: {{definition_fragment}}

Available entity types for grain: {{entity_types_formatted}}
Related metrics (check for do_not_confuse_with): {{related_metrics_formatted}}

Reference sections:
{{relevant_corpus_sections_formatted}}

Produce the full metric definition.
```

**Chains to:** P-MET-04 (discrimination test for each do_not_confuse_with pair).  
**Model settings:** temperature 0, max_tokens 1000.

---

#### P-MET-04: Discrimination test generation

**Purpose:** For each do_not_confuse_with pair, generate a test question that a resolver must answer differently for each metric. Verifies the distinction is real and testable.

**Inputs:**
```
metric_a: { metric_id, canonical_name, definition }
metric_b: { metric_id, canonical_name, definition }
```

**System prompt:**
```
You generate discrimination test cases that verify two metrics are genuinely distinct.
A test case is a natural language question where one metric is the correct answer
and the other is a plausible but wrong answer.

Output only valid JSON matching the schema below.
Schema: {
  "test_cases": [{
    "question": str,
    "correct_metric_id": str,
    "confusable_metric_id": str,
    "discrimination_key": str  // the feature of the question that makes the distinction clear
  }]
}
Generate 3 test cases.
```

**User message:**
```
Metric A: {{metric_a.canonical_name}} — "{{metric_a.definition}}"
Metric B: {{metric_b.canonical_name}} — "{{metric_b.definition}}"

Generate 3 questions where a respondent might confuse these metrics,
where one is clearly correct and the other is plausible but wrong.
```

**Output schema:**
```json
{
  "test_cases": [
    {
      "question": "How many customer orders have not shipped past their committed date?",
      "correct_metric_id": "demand_backlog",
      "confusable_metric_id": "open_po_quantity",
      "discrimination_key": "subject is customer orders not supplier POs"
    }
  ]
}
```

**Chains to:** These test cases become pack benchmark questions (evaluated in Stage 8).  
**Model settings:** temperature 0, max_tokens 600.

---

### Group BND — Binding template generation

---

#### P-BND-01: Binding gap detection

**Purpose:** Identify fields or filter conditions required by a metric formalization that are not covered by the documented ERP schema.

**Inputs:**
```
metric: { metric_id, definition, standard_filters, grain }
system_type: str  # sap_s4hana | snowflake | oracle_scm
documented_schema: { objects: [str], fields: [{ name, type, description }] }
```

**System prompt:**
```
You identify gaps between a metric definition's requirements and a documented ERP schema.
A gap exists when the metric requires a field, filter, or join that is not present
in the documented schema objects and fields provided.

Output only valid JSON matching the schema below.
Schema: {
  "gaps": [{
    "requirement": str,
    "gap_type": "MISSING_FIELD" | "MISSING_FILTER" | "MISSING_JOIN" | "AMBIGUOUS_MAPPING",
    "description": str (max 20 words)
  }]
}
Return { "gaps": [] } if the schema fully covers the metric requirements.
```

**User message:**
```
Metric: {{metric.metric_id}} — "{{metric.definition}}"
Required grain: {{metric.grain}}
Required filters: {{metric.standard_filters.description}}

Available schema for {{system_type}}:
Objects: {{documented_schema.objects}}
Fields: {{documented_schema.fields_formatted}}

Identify any requirements the schema does not satisfy.
```

**Chains to:** Gaps without a proposed fill are flagged for integration engineer review. Gaps feed P-BND-02.  
**Model settings:** temperature 0, max_tokens 500.

---

#### P-BND-02: Binding gap fill proposal

**Purpose:** For each detected gap, propose a field mapping or filter condition. Output is always a PROPOSED draft, never auto-published.

**Inputs:**
```
gap: { requirement, gap_type, description }
metric_id: str
system_type: str
documented_schema: { objects, fields }
```

**System prompt:**
```
You propose candidate ERP field mappings or filter conditions for metric binding gaps.
Your output is a draft proposal that requires integration engineer review.
Never invent table or field names — use only names present in the documented schema
or explicitly flag when you cannot resolve the gap from the provided schema.

Output only valid JSON matching the schema below.
Schema: {
  "proposed_fill": str | null,
  "confidence": "HIGH" | "MEDIUM" | "LOW",
  "requires_review": bool,
  "notes": str
}
Set proposed_fill to null if you cannot resolve the gap from the provided schema.
```

**Chains to:** All outputs enter the steward queue as PROPOSED. Integration engineer confirms before the binding template is published.  
**Model settings:** temperature 0, max_tokens 300.

---

### Group EVL — Evaluation and benchmark

---

#### P-EVL-01: Benchmark question generation

**Purpose:** Generate benchmark questions for the assembled ontology, used to evaluate pack correctness before publication.

**Inputs:**
```
entity_types: [{ entity_type_id, label, definition }]
relationship_types: [{ relationship_type_id, label, from_type, to_type }]
metrics: [{ metric_id, canonical_name }]
domain: str
question_category: "entity_resolution" | "relationship_traversal" | "metric_disambiguation" | "source_routing"
count: int
```

**System prompt:**
```
You generate benchmark questions for evaluating an enterprise ontology.
Each question must have an unambiguous expected answer derivable from the ontology.
Questions must test real business user language, not technical ontology terms.

Output only valid JSON matching the schema below.
Schema: {
  "benchmark_questions": [{
    "question": str,
    "category": str,
    "expected_entity_type_ids": [str],
    "expected_relationship_type_ids": [str],
    "expected_metric_ids": [str],
    "expected_outcome_description": str
  }]
}
```

**User message:**
```
Domain: {{domain}}
Category: {{question_category}}
Generate {{count}} benchmark questions.

Available entity types: {{entity_types_formatted}}
Available relationship types: {{relationship_types_formatted}}
Available metrics: {{metrics_formatted}}
```

**Chains to:** Benchmark question set is stored in pack `/evaluation/benchmark_questions.json`. Evaluated by running each question through the ContextGraph resolver against the pack.  
**Model settings:** temperature 0.2 (intentional: slight variation to produce diverse questions), max_tokens 2000.

---

#### P-EVL-02: Consistency failure explanation

**Purpose:** When an automated consistency check fails, produce a human-readable explanation for the steward review queue.

**Inputs:**
```
check_type: str
failed_items: [str]
ontology_context: str  # relevant excerpt
```

**System prompt:**
```
You explain a consistency check failure in an enterprise ontology in plain language.
Your explanation is for a domain expert reviewer, not an engineer.
Output only valid JSON matching the schema below.
Schema: {
  "plain_language_description": str,
  "likely_cause": str,
  "suggested_fix": str
}
Each field max 30 words.
```

**Chains to:** Appended to steward review queue item.  
**Model settings:** temperature 0, max_tokens 400.

---

## Prompt chain orchestration

The following shows how prompts chain for a single sub-domain through the full pipeline. Parallel branches run concurrently.

```
P-DEC-01 (decompose domain)
    │
    ├── P-DEC-02 × 20 (coverage validation)  ──► re-run P-DEC-01 if fails
    └── P-DEC-03 × n_pairs (overlap check)   ──► re-run P-DEC-01 if fails
    │
    ▼
For each sub-domain (parallel):
    │
    ├── P-COR-01 × n_sections (relevance scoring)
    │       └── HIGH/MEDIUM sections → P-COR-02 (entity hints)
    │
    ├── [Pass 1] P-ENT-01  ─┐
    ├── [Pass 2] P-ENT-01  ─┼── P-ENT-02 (consensus per concept)
    └── [Pass 3] P-ENT-01  ─┘       │
                                     ├── FULL/MAJORITY → P-ENT-03 (definition merge)
                                     └── MINORITY/CONFLICT → steward queue
    │
    ▼
Across all sub-domains (sequential — requires all entity types):
    │
    ├── P-TAX-01 × n_proposed_pairs (pairwise is-a, 3 votes each)
    │       └── Confirmed edges → P-TAX-02 × n_parents (sibling coherence)
    │
    ├── P-REL-01 (candidate pair generation)
    │       └── For each pair (parallel):
    │               ├── [Pass 1] P-REL-02 ─┐
    │               ├── [Pass 2] P-REL-02 ─┼── consensus
    │               └── [Pass 3] P-REL-02 ─┘
    │                       └── Agreed relationships → P-REL-03 (synonym check)
    │
    └── P-MET-01 (metric candidates)
            └── P-MET-02 × n_pairs (synonym dedup)
                    └── Unique candidates → P-MET-03 (formalization)
                            ├── For each do_not_confuse_with: P-MET-04 (discrimination tests)
                            └── Gaps detected → P-BND-01 (gap detection)
                                    └── Each gap → P-BND-02 (gap fill proposal)
    │
    ▼
P-EVL-01 × 4 categories (benchmark question generation)
    │
    ▼
Automated consistency checks
    └── Failures → P-EVL-02 (human-readable failure explanation for steward queue)
    │
    ▼
Steward review queue
    │
    ▼
Pack compilation and publication
```

---

## Prompt versioning and change management

Each prompt is identified by its group and ID plus a semantic version:

```
P-ENT-01:v2.1
```

A prompt change is a pipeline change. Rules:

| Change type | Action required |
|---|---|
| Output schema change (additive) | Re-run affected sub-domains; compare output diff |
| Output schema change (breaking) | Full re-run of affected stage; steward re-reviews affected sections |
| Instruction change (significant) | Re-run all three passes for affected sub-domains; re-compute consensus |
| Instruction change (clarification only) | Re-run MINORITY/CONFLICT items only; FULL consensus items unaffected |
| System prompt change | Treat as significant instruction change |

Prompt versions are stored alongside pack versions. The manifest records which prompt version was used to produce each pack artifact.

---

## Token budget guidance

| Prompt | Typical calls per domain pack | Estimated tokens (in+out) |
|---|---|---|
| P-COR-01 | 200–500 | ~500 |
| P-COR-02 | 100–200 | ~800 |
| P-DEC-01 | 1–3 | ~1,200 |
| P-DEC-02/03 | 30–50 | ~400 |
| P-ENT-01 | 15–30 passes | ~2,500 |
| P-ENT-02 | 100–300 | ~400 |
| P-ENT-03 | 80–200 | ~300 |
| P-TAX-01 | 200–600 votes | ~200 |
| P-TAX-02 | 20–60 | ~600 |
| P-REL-01 | 1 | ~2,000 |
| P-REL-02 | 150–400 passes | ~1,200 |
| P-REL-03 | 50–150 | ~250 |
| P-MET-01 | 3–8 | ~1,500 |
| P-MET-02 | 100–400 pairs | ~250 |
| P-MET-03 | 30–80 | ~1,100 |
| P-MET-04 | 30–80 | ~700 |
| P-BND-01/02 | 20–60 | ~600 |
| P-EVL-01/02 | 10–20 | ~2,000 |

A full supply chain base pack build estimates 2–4M tokens total across all passes. An extension pack (narrower scope) estimates 500K–1.5M tokens.
