# LLM Use in ContextGraph

**Version:** 1.0  
**Status:** Architecture reference  
**Scope:** Where LLMs are used, where they are prohibited, and why the boundary is where it is

---

## Core principle

LLMs generate candidates. They do not authorize decisions.

Every resolution outcome, execution plan, source selection, and policy decision must be traceable to deterministic, evidence-backed logic. An LLM may propose a candidate that feeds into that process, but the candidate must be validated, calibrated, and arbitrated before it can influence a signed plan. The system must function correctly — at reduced recall — when every LLM call is unavailable.

---

## Where LLMs are used

### 1. Intent and domain classification

**What:** Classify the incoming request into an approved operation family and risk tier.

**How:** The classifier maps the normalized request text to one of the registered operation families (`procurement.open_purchase_order_quantity`, `supply_chain.supplier_exposure`, etc.) and assigns a risk tier (`INFORMATIONAL`, `PLANNING_DECISION`, `OPERATIONAL_ACTION`).

**LLM role:** Propose the operation family and risk tier as structured output. The output is validated against the approved operation family registry — if the proposed family is not registered, the request is rejected or returned as `UNSUPPORTED`. The LLM cannot invent a new operation family.

**Fallback:** Rule-based classifier using term matching and entity type context. LLM is invoked only when the rule-based path produces no confident match.

**Trust level:** Untrusted. The proposed classification is confirmed against the registry before any downstream work proceeds.

---

### 2. Optional entity candidate proposer

**What:** Suggest entity candidates when the deterministic resolver pipeline (canonical ID → approved alias → lexical/fuzzy → vector) fails to produce a candidate above the evidence sufficiency threshold.

**How:** The orchestrator invokes the LLM proposer as a late-stage, budget-gated evidence producer. The LLM receives the normalized request text, entity type context, tenant ontology summary, and nearby terms. It returns a structured list of candidate entity IDs with a stated basis — not free text.

**LLM role:** One evidence producer among several. Its output is an `EvidenceRecord` with:
- `producer: LLM_CANDIDATE_PROPOSER`
- `authority_class: UNTRUSTED`
- `raw_score`: model-stated confidence (not used directly — recalibrated by the arbiter)
- `candidate_id`: must be a known canonical entity ID, not a name

The arbiter treats the LLM evidence record with the lowest precedence. It cannot override an exact alias match or an approved canonical identifier.

**Constraints:**
- Only invoked when deterministic producers have been exhausted and the latency budget permits
- Only invoked for risk tiers below `OPERATIONAL_ACTION`
- Proposer cannot emit a candidate that does not exist in the entity registry — the arbiter validates the proposed `candidate_id` before including it
- If the LLM proposes a candidate that conflicts with an existing approved alias for a different entity, the conflict is routed to the steward queue, not silently resolved

**Fallback:** Skip proposer; return `INSUFFICIENT_EVIDENCE` or `AMBIGUOUS` if no deterministic candidate was found.

**Trust level:** Untrusted. Treated as the weakest evidence class. Requires corroboration from at least one deterministic producer before selection.

---

### 3. Missing-term and ontology gap detection

**What:** When a term in the request does not match any entity, alias, metric, or ontology concept, flag it as a potential gap in the tenant ontology.

**How:** After the resolution pipeline, unresolved terms are passed to an LLM that proposes: (a) what the term likely refers to, (b) which existing ontology concept it most closely maps to, and (c) whether it warrants a new entity or alias.

**LLM role:** Generate a structured proposal for the governance queue. The proposal includes a suggested mapping, a confidence class, and a suggested steward action (`ADD_ALIAS`, `ADD_ENTITY`, `ADD_METRIC`, `REJECT`).

**Constraints:**
- Output goes directly to the steward review queue — it is never auto-published
- High-risk mappings (new canonical entities, metric definition changes) always require steward approval regardless of confidence
- Only low-risk, low-impact suggestions (new alias for an existing entity with high confidence) may be eligible for auto-approval after calibration data establishes a reliable threshold

**Fallback:** Log the unresolved term; queue for manual steward review without an LLM proposal.

**Trust level:** Untrusted. Steward must approve all proposals before publication.

---

### 4. Explanation generation

**What:** Produce the human-readable explanation of how a resolution was reached, included in `GET /v1/resolutions/{id}` responses.

**How:** The explanation is generated from the structured `resolution_event` record — the same record used for audit. The LLM receives the decision log (entity candidates, evidence records, arbiter outcome, guardrail results, source selection) and renders it as a readable explanation.

**LLM role:** Rendering only. The LLM is given the facts; it converts them to prose. It does not have latitude to introduce facts not present in the decision record, reinterpret the resolution outcome, or omit negative decisions (denials, abstentions, ambiguities).

**Constraints:**
- The explanation is generated from the `resolution_event`, not from a fresh reasoning pass over the original question
- If the explanation text conflicts with the structured decision record (e.g., describes a different entity as selected), the structured record is authoritative and the explanation is flagged for review
- For audit purposes, the structured `resolution_event` is the record of truth — the explanation is a derived view

**Fallback:** Return structured decision fields directly without rendered prose. All explanation fields are optional in the API contract.

**Trust level:** Rendering assistant. Output is bounded by the structured input it receives.

---

### 5. Evaluation case proposal (offline)

**What:** In the evaluation pipeline, propose new evaluation cases from sampled production divergence and correction events.

**How:** When a steward override, user clarification, or production/candidate divergence is detected, an LLM reviews the event and proposes: (a) a labeled evaluation case (input → expected output), (b) whether it represents a new failure mode or an existing known category, and (c) a suggested priority for steward review.

**LLM role:** Triage and labeling assistant for the steward review backlog. Reduces manual work in identifying novel failure modes from high-volume divergence streams.

**Constraints:**
- No observed outcome is automatically treated as ground truth (Architecture §19.1)
- All proposed cases enter `PENDING_STEWARD_REVIEW` state
- Steward must confirm, reject, or mark ambiguous before the case affects calibration or training

**Fallback:** Queue all divergence events for manual steward triage without LLM prioritization.

**Trust level:** Untrusted. Labeling proposals require steward approval.

---

## Where LLMs are prohibited

These are hard boundaries, not performance recommendations. The system must not use LLM inference in these paths under any conditions.

| Path | Reason |
|---|---|
| Authorizing a resolution outcome | Only the calibrated arbiter, using deterministic evidence records, may select a canonical entity |
| Authorizing or signing an execution plan | Plans are signed by the plan compiler with pinned, versioned, deterministic inputs |
| Policy evaluation or guardrail decisions | Guardrail outcomes (PERMIT, DENY, APPROVAL_REQUIRED, etc.) are produced by the deterministic guardrail engine — Cedar/OPA for authorization, product-owned logic for semantic/temporal/evidence/risk layers |
| Source routing | Source selection is a lookup against the precompiled route registry, not inference |
| Relationship traversal or path finding | Paths come from the precomputed graph and indexed edges — not LLM inference |
| Bundle content or ontology publication | All published artifacts are human-steward-approved |
| Tenant derivation | Tenant comes from verified JWT/OIDC claims only |
| Idempotency replay decisions | Replay is determined by matching the canonical request digest against stored records |
| Confidence calibration | Calibration is statistical, computed from the evaluation corpus — not LLM-stated |

---

## How LLM outputs are bounded in practice

Three mechanisms enforce the constraint that LLMs generate candidates but do not authorize:

**1. Structured output with registry validation**  
LLM outputs are always structured (JSON schema), never free text that the system must parse and act on. Every entity ID, operation name, or metric ID proposed by an LLM is validated against the registry before it can proceed. An invented or hallucinated identifier fails validation and is dropped.

**2. Evidence record encapsulation**  
LLM outputs enter the pipeline as `EvidenceRecord` objects with `authority_class: UNTRUSTED`. The arbiter applies authority-class-aware precedence rules. An untrusted record cannot override a deterministic exact match and cannot alone meet the evidence sufficiency threshold for any risk tier above INFORMATIONAL.

**3. Budget and risk-tier gates**  
The LLM proposer is only invoked when: (a) the latency budget has not been exhausted, (b) no deterministic producer has reached the confidence threshold, and (c) the operation risk tier permits LLM assistance. For OPERATIONAL_ACTION tier requests, the LLM proposer is not invoked — the system returns INSUFFICIENT_EVIDENCE and requires human input.

---

## Availability and degradation behavior

The system must not depend on LLM availability for its core correctness guarantees.

| LLM component | Unavailable behavior |
|---|---|
| Intent classifier | Fall back to rule-based classifier; if that fails, return `UNSUPPORTED` |
| Candidate proposer | Skip; return `INSUFFICIENT_EVIDENCE` or `AMBIGUOUS` — do not block the deterministic path |
| Explanation generator | Return structured decision fields; omit rendered prose |
| Missing-term proposer | Log unresolved term; queue for manual steward review |
| Evaluation case proposer | Queue divergence events for manual triage |

The deterministic fast path — exact alias match → unique candidate → guardrail pass → signed plan — involves zero LLM calls and must remain available regardless of LLM service health.

---

## MVP vs. later phases

LLM use is introduced incrementally.

**MVP (Phase 1):** No LLM in the runtime path. Intent classification is rule-based. Resolution uses canonical ID, alias, and lexical/fuzzy resolvers only. Explanation generation returns structured fields. This proves the core value proposition — sub-200 ms governed context compilation — without any LLM runtime dependency.

**Phase 2:** Enable the optional LLM candidate proposer for INFORMATIONAL risk tier requests where deterministic resolvers produce insufficient evidence. Measure candidate quality against the evaluation corpus before enabling for higher risk tiers.

**Phase 3+:** Enable explanation prose generation after the explanation template is validated against the structured decision record for fidelity. Enable missing-term detection and evaluation case triage in the offline pipeline.

The LLM is introduced into the runtime path only after the deterministic system is proven and after there is calibration data to measure the LLM's marginal contribution to resolution quality.

---

## Model selection guidance

No specific model is mandated. Selection criteria for each use case:

| Use case | Key requirements |
|---|---|
| Intent classification | Fast, low latency, structured output, deterministic at temperature 0 |
| Candidate proposer | Strong entity linking, structured output, can follow schema constraints reliably |
| Explanation generation | Faithful summarization, does not add facts not present in input |
| Missing-term proposal | Strong semantic similarity reasoning, structured output |
| Evaluation triage | Good at categorization and labeling from structured event logs |

All LLM calls must use deterministic sampling (temperature 0 or equivalent) for any output that feeds a resolution decision. Explanation generation may use non-zero temperature.

Prompts for structured-output calls (classification, candidate proposal, missing-term detection) are versioned and pinned to the semantic bundle. A prompt change constitutes a semantic change and triggers evaluation regression before deployment.
