ContextGraph / Lattice — Architecture Review & Implementation Plan
Part I: Correctness Analysis
Critical Issues (production blockers)
C1: Client-controlled tenant_id across all product spec API endpoints

The Product Spec defines X-Tenant-ID: {tenant_id} as a request header and passes tenant_id in query parameters (GET /relationships?…&tenant_id=nvidia-prod) and request bodies (POST /metrics, POST /route, POST /resolve, POST /context).

The Architecture doc (§6.1, §13.1, ADR-004) explicitly states: "Tenant selection is never trusted solely from a request header" and lists "confused-deputy execution caused by client-supplied tenant or source identifiers" as a named threat.

These two documents directly contradict each other. Any client can set an arbitrary tenant_id and impersonate another tenant. The spec's API surface must be revised: tenant_id must be derived exclusively from verified OIDC/JWT claims, never from client-supplied fields.

C2: /route endpoint leaks raw SAP physical schema into the public API

The Product Spec §7 /route response returns:


"table_hints": ["EKKO", "EKPO", "EKET"],
"filters": { "supplier_lifnr": "SK_HYNIX_LIFNR", "status": ["B", "C"] },
"aggregate": "SUM(MENGE) as committed_quantity"
This violates ADR-003 ("Public API exposes business operations, not SQL/SAP table hints") and the Architecture §6.10 principle that the IR uses "stable business operation identifier; never a raw SQL fragment." The route response couples the public contract to SAP R/3 internal table names, field names, and status codes. A SAP upgrade, custom field rename, or switch to S/4HANA ABAP-managed views breaks every API consumer. It also leaks internal data model details. The public /route response must return only the connector-neutral operation IR, not physical table hints.

C3: Governance atomicity gap — multi-step API allows bypassing integrated guardrails

The Product Spec defines six independently callable endpoints (/resolve, /context, /route, /metrics, /relationships, /explain). The Architecture's core POST /v1/compile endpoint is designed to enforce governance atomically: one request → classification → resolution → guardrails → authorization → plan, all within a single transaction that pins versions and produces a signed result.

An agent that calls /resolve → constructs its own source query → calls /route separately never triggers the integrated guardrail engine. The classification, relationship validation, evidence sufficiency check, risk-tier approval, ambiguity policy, and temporal validity evaluation from §6.8 are skipped. The "govern before execution" principle from §3 is satisfied only if these checks happen within a single atomic compile call. If the Product Spec's individual endpoints are preserved, they must each run the full guardrail pipeline — or they must be explicitly scoped as read-only registry lookups with no execution planning authority.

C4: Clarification API missing from Product Spec

The Architecture defines POST /v1/clarifications/{id} as a first-class endpoint for submitting selected clarification choices when the compiler returns CLARIFICATION_REQUIRED. The Product Spec does not include this endpoint. The AMBIGUOUS and CLARIFICATION_REQUIRED outcomes are described as valid system outputs (Architecture §6.8, §7.3) but the consumer-facing API to resolve them does not exist in the spec. The clarification flow — return candidates → agent selects → submit selection → recompile — is broken without this endpoint.

High Severity Issues
H1: Compiled-plan cache key missing semantic bundle / ontology version

The processing-paths document (§8) defines the compiled-plan cache key as:


tenant + intent signature + entity IDs + metric version + policy version
The semantic bundle version and ontology version are absent. When the ontology is updated (e.g., a new relationship type is added, an entity hierarchy changes, a metric formula is revised), cached compiled plans continue to be served against the old semantic model. The Architecture §12 and §16 require that every response pin semantic_bundle and semantic_bundle_digest. The cache key must include both.

H2: Query-result cache authorization scope is ambiguous

Processing-paths §8 specifies the query-result cache key must include "principal scope where relevant" without defining when scope is "relevant." In a multi-user tenant, two principals with different authorization scopes (one with supplier PII access, one without) could share a cache entry if scope is omitted as "not relevant" for a given operation. This is a potential authorization bypass. The cache key policy must define: scope is always included for any operation with field-level or row-level authorization, with no exceptions. "Where relevant" must be replaced with a deterministic rule.

H3: Uncalibrated confidence scores in MVP

Architecture §10.2 is explicit: "Confidence is exposed only where evaluation data supports calibration." The Product Spec shows numeric confidence scores (0.98, 0.96, 0.93, 0.91) in every API response example. MVP has no evaluation corpus and no calibration data. Exposing uncalibrated numeric confidence values as if they were probabilities is misleading. Enterprise consumers will build decision logic on these numbers. Without calibration, the scores are raw model outputs, not reliability estimates. MVP responses should expose evidence_strength (categorical: EXACT_MATCH, HIGH, MEDIUM, LOW, INSUFFICIENT) and assertion_class, not numeric probabilities, until calibration data exists.

H4: Plan signing key infrastructure undefined

The Architecture §8 IR shows "verification_requirements": ["signature"] and the delegated execution SDK must support "key discovery and rotation support." Neither document specifies: signing algorithm, key format, key distribution mechanism, how an executor discovers the signing public key, how key rotation is announced and processed, or what the revocation model is. Without this, delegated execution cannot be implemented — and the conformance suite (§18) cannot test what it cannot specify.

Medium Severity Issues
M1: GET /relationships accepts unresolved entity names

GET /relationships?from=SK+Hynix&to=product passes free-text entity names as query parameters. This either forces the endpoint to silently run entity resolution (hidden governance scope, no idempotency key, no version pinning) or it accepts ambiguous names without resolution, returning results for whichever interpretation the system picks first. The endpoint should require pre-resolved entity IDs (from=SUP-0042) and document that callers must go through /resolve first — or it must run the full resolution pipeline and pin versions accordingly.

M2: Stale relationship behavior for high-risk decisions

Processing-paths §7 specifies stale_behavior: "return_with_warning" for relationship snapshots (maximum_age: 24h). But Architecture §6.4 requires risk-tier-specific thresholds. A PLANNING_DECISION risk-tier request that uses stale BOM relationships (e.g., a component was removed from a product BOM since the last snapshot) returns an incorrect execution plan with a warning that is easily ignored by an automated agent. Stale behavior must be risk-tier-aware: return_with_warning is acceptable for INFORMATIONAL but not for PLANNING_DECISION or above, where the correct behavior is STALE_CONTEXT or mandatory revalidation.

Architecture Design Gaps
D1: Plane model inconsistency between documents

The Architecture uses Control Plane / Data Plane / Execution Plane. Processing-paths introduces Build Plane / Runtime Plane / Execution Plane / Feedback Plane. The four-plane model is strictly more accurate: it correctly separates the offline context-building pipeline from the real-time compilation path, and it names the feedback loop as a first-class plane rather than a footnote. The Architecture document should adopt the four-plane model. This isn't cosmetic — the four planes have different deployment topologies, scaling strategies, and SLOs.

D2: Bundle granularity and atomic activation are in tension

Processing-paths §3 describes separate refresh cadences per content type (entity aliases: hourly, policies: on publication event, graph snapshots: daily, ontology: on approved change). The Architecture §6.12 describes atomic bundle activation from a single signed manifest. These are in tension: if entity aliases are versioned hourly but the policy bundle only changes weekly, atomic activation either requires rebundling all content on every alias change (expensive) or means sub-components are independently versioned (then "atomic activation" applies to sub-bundles, not the composite). The design must explicitly define whether the bundle is a composite of independently versioned sub-components or a single monolith, and what "atomic activation" means at each granularity.

D3: Relationship lifecycle state machine not specified

Architecture §9 mentions separate relationship_assertion and accepted_relationship_fact tables and lists states asserted, derived, inferred, ambiguous, overridden, deprecated. Processing-paths mentions approval state on each edge. But the state machine — how a relationship moves from proposed → candidate → steward-reviewed → approved → published → deprecated — is not defined. This is foundational to the offline pipeline (§2.8 in processing-paths), the governance model, and the evaluation corpus. Without it, the ingestion pipeline cannot determine what to load into the runtime store.

D4: Phase 0 Graphiti evaluation criteria are undefined

ADR-007 says Graphiti is evaluated behind an internal interface. Phase 0 exit criteria include "evidence-based infrastructure decision." But neither document defines what to benchmark, what SLO thresholds would make Graphiti preferable, what the cost comparison model is, or which workloads (bounded traversal, closure queries, temporal filtering) to stress. Without specific criteria, Phase 0 evaluation stalls or produces an inconclusive result.

D5: Connector capability schema undefined

Architecture §6.14 requires "capability declaration and allowlist" and "signed connector manifests." The connector SDK (Appendix A, /packages/connectors) depends on this schema. Without a defined capability format, connector certification cannot proceed, WASM vs. out-of-process decisions cannot be evaluated per connector, and the allowlist cannot be enforced.

Part II: Implementation Plan
The plan follows the phased delivery architecture (§20) with corrections for the issues above. Each phase has concrete deliverables, exit criteria, and critical-path sequencing.

Pre-Phase: Repository and Foundations (2 weeks, parallel with Phase 0 spike)
Goal: Runnable skeleton with correct contracts before any feature work.

Initialize monorepo at /contextgraph per Appendix A structure.
Define the canonical contracts package first — Pydantic/JSON Schema models for:
CompileRequest / CompileResponse (with decision, risk_tier, evidence_refs, pinned_versions)
ExecutionPlanIR (§8 schema, with all required fields)
EvidenceRecord (producer, authority, raw score, semantic version)
GuardrailOutcome enum: PERMIT / DENY / CLARIFICATION_REQUIRED / APPROVAL_REQUIRED / STALE_CONTEXT / INSUFFICIENT_EVIDENCE / UNSUPPORTED
ClarificationContract (candidates + machine-readable selection schema)
Problem-details error taxonomy with stable reason_code values
Fix C1 immediately: Define TenantContext as derived-only from JWT/OIDC claims. No tenant_id in public request schema.
Set up local development environment: PostgreSQL 17, Redis, object storage stub, mock gateway.
Establish CI pipeline: schema compatibility checks, contract tests, migration dry-runs.
Phase 0: Architecture Spike (Weeks 1–6)
Goal: Settle infrastructure decisions before building the runtime.

Track A — Graph substrate evaluation (addresses D4)

Define specific benchmarks before starting:

Benchmark	PostgreSQL typed edges	Graphiti
Bounded traversal (depth 4, typed) p95	target < 50 ms	same
Direct adjacency query (tenant-scoped)	target < 10 ms	same
Temporal filter (valid_time range)	target < 20 ms	same
Closure rebuild (1000-node subgraph)	target < 30 s	same
Run benchmarks against a synthetic dataset representative of one enterprise tenant (10K entities, 100K relationships). Decision rule: if PostgreSQL meets all targets under this load, use PostgreSQL. Adopt Graphiti only if it beats PostgreSQL by >30% on traversal and closure and its interface can be hidden behind /packages/graph.

Track B — Typed IR prototype

Implement the execution-plan IR (§8) as a real serializable, signable struct. Verify: canonical serialization → SHA-256 digest → Ed25519 signature → verification round-trip. This resolves H4: establish the signing key infrastructure (Ed25519 keypair, public key in well-known endpoint, key ID in plan header) before any execution work starts.

Track C — Evidence arbiter prototype

Implement a minimal resolver pipeline: canonical ID resolver → alias resolver → evidence record emitter → arbiter with precedence rules. No LLM, no vector. Verify that the arbiter correctly returns early on an exact match and produces an INSUFFICIENT_EVIDENCE outcome when no resolver fires. This validates the resolution model before the full pipeline is built.

Exit criteria: Benchmark results documented, graph substrate decision made with data, IR signing proven in code, evidence arbiter returning correct outcomes for 20 test cases.

Phase 1: MVP — One Tenant, One Workflow Family (Months 1–4)
Build order is sequenced by dependency. Each layer depends on the one above it.

Layer 1: Data model and offline pipeline foundation (Weeks 1–4)
Start here because runtime compilation queries prebuilt data.

PostgreSQL schema — implement all tables from §9 with correct bitemporal columns (valid_time tstzrange, system_time tstzrange) and RLS by tenant. Tables in build order:

tenant, principal
entity, entity_alias
source_system, source_binding
ontology_term, metric_definition
relationship_assertion, accepted_relationship_fact
relationship_closure (initially empty, populated by pipeline)
resolution_event, execution_plan, idempotency_record
semantic_bundle_manifest, evaluation_case
Source registration (processing-paths §2.1) — implement the source_system registry with capability declarations (addresses D5: define the capability schema as part of the registry record, not deferred).

Offline ingestion pipeline for one SAP S/4HANA instance:

Metadata discovery scan (processing-paths §2.2): collect approved CDS views, table/field metadata, status code definitions
Metadata normalization (§2.3): canonical physical_asset model
Entity population (§2.4): supplier master, customer master, material master
Entity reconciliation (§2.5): alias generation, exact + fuzzy matching → steward queue for conflicts
Metric mapping (§2.7): define open_po_quantity and 14 other MVP metrics with source bindings
Relationship population (§2.8): SUPPLIES, USED_IN, BUILDS edges from BOM extract
Route compilation (§2.10): precompile routing rules for each canonical operation
Governance queue (processing-paths §2.13): basic steward UI or CLI for approving entity matches and relationship assertions. This is blocking for entity publication — do not skip.

Bundle compiler (§6.12): canonical serialization → SHA-256 digest → signed manifest → object storage. Implement atomic activation via version pointer in PostgreSQL. Test rollback to previous bundle.

Layer 2: Real-time context compilation API (Weeks 3–7, overlaps with Layer 1)
API gateway (/apps/api):

TLS termination, OIDC token validation
Tenant derivation from JWT claims only (fixes C1)
Idempotency key validation and replay logic
Request digest (SHA-256 of canonical request)
Rate limiting with retry-after headers
Single compile endpoint POST /v1/compile (fixes C3):

Validates idempotency key; replays original pinned result if match
Pins semantic_bundle, policy_bundle, api_schema versions at request start
Calls orchestrator; returns signed ExecutionPlanIR or ClarificationContract
Context compiler orchestrator (/packages/compiler):

Domain and intent classifier (rule-based for MVP, no LLM dependency)
Dispatches to resolution evidence producers
Calls calibrated arbiter
Calls guardrail engine
Calls source router
Calls plan compiler
Persists resolution event
Resolution evidence producers (/packages/resolver):

Canonical identifier resolver (exact match on entity_alias table)
Approved alias resolver (normalized form lookup)
Lexical/fuzzy resolver (pg_trgm on alias index)
Contextual scorer (entity type + operation family rules)
Omit LLM and vector resolvers for MVP
Calibrated arbiter (/packages/arbiter):

Early return on unique exact match
Explicit ambiguity detection (competing candidates above threshold)
Returns ClarificationContract with candidates (fixes C4: this feeds the clarification API)
Categorical evidence strength output (fixes H3: no numeric probability until calibration data exists)
Clarification API POST /v1/clarifications/{id} (fixes C4):

Accepts selected candidate from agent
Recompiles with disambiguated entity
Returns signed plan
Guardrail engine (/packages/guardrails):

Authorization check via Cedar/OPA (RBAC + ABAC)
Semantic admissibility (operation family allowed for this entity type)
Temporal validity (entity effective dates)
Evidence sufficiency (minimum evidence strength by risk tier)
Ambiguity policy (no silent selection for PLANNING_DECISION+)
Returns typed GuardrailOutcome
Source router (/packages/bindings):

Lookup-only against precompiled route registry (processing-paths §2.10)
Never broadcasts to discover capabilities at runtime
Selects primary source based on freshness requirement, health, permissions
Plan compiler (/packages/compiler):

Produces connector-neutral ExecutionPlanIR
Business operation identifier only — no SQL, no SAP table names in output (fixes C2)
Signs plan with Ed25519, includes expiry and nonce
Supporting endpoints:

GET /v1/resolutions/{id} — retrieve decision + evidence + explanation
GET /v1/metrics/{id} — approved metric definition (no tenant_id in path/body)
GET /v1/entities/{id} — canonical entity view for authorized principal
GET /v1/bundles/{version}/manifest — signed bundle manifest
GET /v1/health, GET /v1/ready
Layer 3: Execution gateway (Weeks 5–9)
Build in Rust (/apps/gateway-rust) per ADR-008.

Plan signature verification (Ed25519)
Tenant, expiry, nonce, binding version validation
Local policy re-check (independent from compile-time check)
Secret retrieval from customer vault (no credentials in ContextGraph SaaS)
Operation translation: procurement.open_purchase_order_quantity → SAP BAPI/CDS view call
Result normalization to expected_result_schema
Source freshness and partial-result state attached to response
Out-of-process connector runner (/apps/connector-runner):

Typed IPC/gRPC contract between gateway and connector
Resource limits, timeout, crash containment
SAP S/4HANA connector (MVP)
Snowflake connector (MVP)
Layer 4: Observability, caches, and reliability (Weeks 6–10)
Redis caches with correct key structures (fixes H2):

Resolution cache: {tenant}:{bundle_digest}:{normalized_phrase}:{domain}
Compiled-plan cache: {tenant}:{semantic_bundle_digest}:{policy_bundle_digest}:{intent_sig}:{entity_ids}:{metric_version} (fixes H1: adds bundle digest)
Relationship cache: {tenant}:{semantic_bundle_digest}:{supplier_id} for impact sets
Query-result cache: always includes {principal_scope_hash} for any authorized operation (fixes H2: removes "where relevant" ambiguity)
Fallback behaviors:

Redis unavailable → PostgreSQL read path, degraded-latency signal in response
Vector index unavailable → continue deterministic path (no MVP vector)
Policy service unavailable → fail closed
Bundle digest mismatch → reject, keep previous bundle
OpenTelemetry instrumentation: trace spans per stage (auth → resolution → guardrails → plan), producer latency metrics, cache hit rates, ambiguity rate, abstention rate, exact-match fast-path rate.

Layer 5: Evaluation foundation (Weeks 8–12)
Build evaluation_case table and steward review queue.
Capture clarification outcomes and steward overrides as pending cases.
Implement golden-query runner against 100–300 manually curated queries.
Implement policy leakage tests (cross-tenant entity access attempts).
Implement temporal tests (as-of queries against backdated alias changes).
Do not expose numeric confidence until calibration is run (fixes H3). Expose evidence_strength enum in all MVP responses.
Phase 1 exit criteria:

100–300 golden queries against one tenant passing at ≥85% accuracy
p95 context compilation < 200 ms (excluding source execution)
Plan signatures, expiry, nonce, tenant, bundle digest verified end-to-end
Tenant isolation tests passing (cross-tenant entity query attempts rejected)
No tenant_id in any public request schema
No physical SAP table names in any public API response
Clarification flow working end-to-end
Rollback proven for application, semantic bundle, policy bundle
Phase 2: Productize (Months 5–9)
Focus: repeatable onboarding, second and third customers, stronger governance.

API surface refinements:

GET /v1/relationships endpoint — requires pre-resolved entity IDs, runs through full guardrail pipeline, pins versions (fixes M1). Accepts ?from_entity_id=SUP-0042&to_type=product&direction=downstream.
Full metric dictionary API GET /v1/metrics with cursor-based pagination.
Relationship snapshot freshness disclosure in all responses that use precomputed paths (processing-paths §4 step 6 pattern).
Risk-tier-aware stale behavior: PLANNING_DECISION+ requests with stale relationship snapshots return STALE_CONTEXT, not a warning (fixes M2).
Governance:

Self-service tenant onboarding workflow in admin service.
Ontology mapping tool: import customer alias CSV, review AI-suggested mappings, approve/reject.
Formalize relationship lifecycle state machine (fixes D3): PROPOSED → CANDIDATE → STEWARD_REVIEW → APPROVED → PUBLISHED → DEPRECATED with explicit transition triggers and approval requirements per assertion class.
Connector SDK with defined capability schema (fixes D5): capability is a versioned JSON Schema in the connector manifest, checked against the allowlist at load time.
Bundle architecture clarification (fixes D2):

Define composite bundle as independent versioned sub-bundles: semantic, policy, metrics, route_templates, graph_snapshot, each with their own digest and activation pointer.
Atomic activation = all sub-bundles for a given composite version are pointer-swapped in a single PostgreSQL transaction.
Cache invalidation on sub-bundle activation is scoped: a metrics bundle update invalidates only metric-bearing cache keys.
Calibration:

After 60 days of production data and steward-reviewed evaluation cases, run first calibration sweep.
Measure ECE (expected calibration error) by producer class.
Only after calibration passes acceptance threshold: add confidence numeric field to API responses alongside evidence_strength. Document calibration methodology.
Connector expansion: Oracle, Databricks, Snowflake (second instance).

Phase 2 exit criteria:

Three tenants onboarded with repeatable process
Connector SDK with capability schema and certification suite
Calibration data sufficient for numeric confidence on at least two query classes
Policy leakage regression suite runs in CI on every bundle publish
Phase 3: Scale (Months 10–18)
Multi-region and delegated execution:

Delegated execution SDK with full verification suite: signature, key rotation, expiry, nonce, schema, binding version (resolves H4 gap completely).
Key management: Ed25519 keypairs rotated on configurable schedule, public keys discoverable at GET /v1/keys/{kid}, key ID in plan header.
Conformance suite for delegated executors: must pass before certification.
Graph and relationship scaling:

Targeted precomputed closures for the three highest-value workflow families (supplier→product exposure, component→product usage, product→customer exposure).
Route template versioning with depth and edge-type constraints per workflow family.
Async deep traversal mode for requests exceeding sync budget.
Embedding and vector retrieval:

Enable tenant-scoped vector resolver for candidate generation.
Per-tenant collection or namespace — never shared index.
Add LLM candidate proposer behind a risk-tier gate (INFORMATIONAL only for MVP of this feature).
Run calibration on vector-augmented resolution separately from deterministic-only calibration.
Evaluation maturity:

Production divergence sampling: sampled comparison of current vs. candidate bundle, no source execution.
Correction loop: unresolved terms, steward overrides, clarification choices automatically proposed as evaluation cases.
Calibration becomes a blocking gate on bundle publication for ontology updates.
Phase 3 exit criteria:

Delegated execution SDK conformance suite passing for one partner
Vector resolver enabled for one tenant with isolation tests passing
Precomputed closures for three workflow families with < 20 ms p95 lookup
Multi-region bundle replication with < 5 s activation lag
Phase 4 and conditional advanced work
WASM connector runtime: evaluate only for connectors with HTTP-only interfaces and no vendor-native driver dependency. Benchmark against out-of-process runner on real connector workload before committing.
Merkle DAG bundle deltas: introduce only when cross-region bundle transfer exceeds 100 MB or rollout time exceeds 30 s.
Industry packs (automotive, pharma): only after semiconductor pack economics are proven and base ontology machinery is stable.
Summary: Highest-Priority Fixes Before Writing Code
In order of consequence:

Priority	Fix	Blocking
1	Remove tenant_id from all public request schema; derive from JWT only	All security guarantees
2	Remove SAP table hints from /route response; emit connector-neutral IR only	ADR-003, public API contract
3	Add POST /v1/clarifications/{id} to product spec	CLARIFICATION_REQUIRED flow
4	Add semantic_bundle_digest to compiled-plan cache key	Cache correctness after ontology updates
5	Specify Ed25519 signing key infrastructure before building delegated execution	Delegated execution mode
6	Define relationship lifecycle state machine (PROPOSED→PUBLISHED)	Offline pipeline publication
7	Replace numeric confidence with evidence_strength enum in all MVP responses	Calibration honesty
8	Define Phase 0 Graphiti benchmark criteria explicitly	Phase 0 won't produce a decision
9	Define connector capability schema in source registry	Connector certification
10	Adopt four-plane model in architecture document	Architecture clarity and correct SLO scoping
