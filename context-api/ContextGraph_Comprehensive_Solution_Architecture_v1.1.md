# ContextGraph Comprehensive Solution Architecture

**Confidential | Version 1.1 | June 2026**

**Document status:** Proposed architecture / decision baseline  
**Audience:** Founders, engineering, security, product, investors, design partners  
**Source:** ContextGraph API Product Specification v0.1, architecture analysis, and production-readiness review

---

## 1. Executive Summary

ContextGraph is a governed context compiler for enterprise agents. It transforms a natural-language business request into a versioned, explainable, policy-checked execution plan grounded in tenant-specific entities, metrics, relationships, authoritative source bindings, and evidence.

ContextGraph is not positioned as another graph database, general-purpose RAG platform, federated SQL engine, or autonomous execution agent. Its differentiation is governed semantic-to-operational compilation: resolving business meaning, validating it against policy and temporal context, selecting approved operations and sources, and producing a signed execution contract.

The recommended architecture separates four concerns:

1. A **control plane** for semantic governance, policy, source binding, tenant configuration, releases, and evaluation.
2. A **low-latency context compilation data plane** that resolves, evaluates, and compiles requests.
3. An **execution plane** that may be managed, customer-hosted, delegated to a conforming customer platform, or omitted in compile-only deployments.
4. Reusable **graph, retrieval, bundle-distribution, and evaluation infrastructure** behind owned product interfaces.

The architecture follows a simple rule: design interfaces for the hard production case, while implementing the simplest viable mechanism first.

| Architectural decision | Recommendation | Reason |
|---|---|---|
| Product boundary | Compile and authorize plans; execution is optional and isolated | Avoid centralizing credentials and customer operational data |
| Primary runtime | Python/FastAPI modular monolith for MVP | Faster iteration and stronger semantic/ML ecosystem |
| Hot-path optimization | Rust only for measured bottlenecks and customer edge gateway | Tail latency and static deployment where commercially valuable |
| System of record | PostgreSQL | Transactional, temporal, tenancy, and governance support |
| Graph layer | Typed directed graph over PostgreSQL initially; Graphiti evaluation permitted | Avoid rebuilding commodity graph capabilities |
| Resolution model | Deterministic fast path plus evidence-producing resolvers and calibrated arbitration | Improves accuracy, explainability, cost control, and graceful abstention |
| Policy model | Authorization engine plus semantic/temporal/risk guardrails | Prevents overloading a binary policy engine with non-binary decisions |
| Cache and bundle delivery | Redis plus immutable, hashed bundles with atomic activation | Repeatability, integrity, rollback, and low-latency access |
| Integration | Connector-neutral typed operation IR plus source-specific adapters | Prevents raw SAP/SQL hints from becoming the public contract |
| Connector isolation | Out-of-process baseline; WASM evaluated for suitable connector classes | Limits connector failures and creates a future-safe plugin boundary |
| Deployment | SaaS compiler plus managed, private, delegated, or compile-only execution | Makes security obligations explicit across deployment modes |
| Vector retrieval | Tenant-isolated indexes or namespaces | Prevents cross-tenant candidate leakage |
| Evaluation | Pre-release regression plus sampled production divergence review | Keeps the corpus aligned with real-world behavior |

---

## 2. Scope, Goals, and Non-Goals

### 2.1 Goals

- Resolve enterprise terms and entities using tenant-approved definitions, aliases, identifiers, and context.
- Compile user intent into a typed, source-neutral, policy-aware execution plan.
- Route plans to authoritative enterprise systems through versioned source bindings.
- Provide evidence, lineage, semantic versions, calibrated confidence where supported, and reason codes for every decision.
- Abstain, clarify, or require approval when ambiguity, evidence weakness, temporal invalidity, or business risk exceeds policy thresholds.
- Support sub-200 ms p95 context compilation for defined workloads, excluding source execution and final LLM generation.
- Enable gradual adoption through API/SDK, MCP, private gateway, delegated execution, and eventually embedded ISV deployment.
- Preserve tenant isolation across relational data, caches, vector retrieval, graph traversal, logs, evaluation data, and exports.
- Support reproducible “as-of” answers using explicit bitemporal semantics.

### 2.2 Non-Goals for MVP

- General-purpose enterprise search or document RAG.
- Replacing MDM, catalogs, semantic layers, graph databases, or source-system authorization.
- Universal direct SQL or SAP query generation exposed to agents.
- Autonomous write actions without deterministic policy and approval.
- A multi-industry ontology marketplace.
- Fully automated ontology authoring without steward review.
- A general-purpose graph traversal platform.
- Mandatory WASM execution for every connector.
- Formal differential privacy guarantees without a complete privacy design and accounting model.

---

## 3. Architecture Principles

| Principle | Implication |
|---|---|
| Govern before execution | Resolution, validation, and authorization occur before an agent or connector accesses operational systems |
| Typed contracts over prompts | Components exchange versioned schemas rather than unstructured prompt fragments |
| Deterministic fast path | Exact identifiers, aliases, rules, and approved mappings are evaluated before expensive inference |
| Evidence fusion over rigid staging | Resolvers emit evidence into a calibrated decision process rather than blindly executing a fixed sequence |
| Abstention is a feature | `AMBIGUOUS`, `CLARIFICATION_REQUIRED`, and `APPROVAL_REQUIRED` are valid outcomes |
| Evidence is mandatory | Every selected entity, metric, relationship, and source route has traceable evidence |
| Temporal by default | Definitions, relationships, mappings, policies, and system ownership are effective-dated and bitemporal where operationally meaningful |
| Graph is infrastructure | The moat is compilation, policy, bindings, evaluation, and operational quality rather than graph storage |
| Data minimization | The SaaS service need not receive source credentials or complete query results |
| Explicit trust transfer | Omitting the gateway transfers validation obligations to a conforming execution environment; it does not remove them |
| Tenant isolation below the application layer | Database, cache, vector, graph, and storage boundaries enforce tenant scope |
| Profile before rewriting | Rust, specialized graph systems, Merkle subtrees, and other advanced infrastructure are introduced only against measured requirements |
| Portable customer context | Tenant mappings are exportable; product value comes from quality and operations rather than hostage-style lock-in |
| Immutable releases, atomic activation | Published semantic and policy artifacts are content-hashed, verified, and activated by version pointer |
| Production evidence improves evaluation | Corrections, clarifications, and sampled divergences feed a steward-reviewed corpus |

---

## 4. System Context and Boundaries

```text
+--------------------+      +----------------------+      +-----------------------+
| User / Business    | ---> | Enterprise Agent /   | ---> | ContextGraph Compiler |
| Application        |      | SDK / MCP / API      |      | SaaS or Private       |
+--------------------+      +----------------------+      +-----------+-----------+
                                                                      |
                                                        governed, signed plan
                                                                      |
                 +----------------------------------------------------+------------------+
                 |                                                    |                  |
                 v                                                    v                  v
      +----------------------+                          +----------------------+  +----------------------+
      | Managed Gateway      |                          | Private Gateway      |  | Delegated Executor   |
      | ContextGraph-hosted  |                          | Customer-hosted      |  | Customer agent/platform|
      +----------+-----------+                          +----------+-----------+  +----------+-----------+
                 |                                                 |                         |
                 v                                                 v                         v
      ERP / Warehouse / PLM / CRM / APIs                 Customer Vault + Sources        Customer Sources
```

The compiler resolves and compiles context. Execution is governed through one of four explicit modes.

### 4.1 Execution Modes

| Mode | Execution owner | Credential location | Compiler visibility | Security invariant |
|---|---|---|---|---|
| Managed | ContextGraph-managed gateway | Customer-controlled vault or secret broker | May receive minimized normalized results | Compiler never receives raw credentials |
| Private | Customer-hosted ContextGraph gateway | Customer environment | Plan only by default; results optional | Compiler sees neither credentials nor full results unless explicitly configured |
| Delegated | Customer agent or execution platform | Customer platform | Plan only | Customer executor validates signature, expiry, versions, policy obligations, and schema |
| Compile-only | No integrated executor | None | Plan only | ContextGraph makes no execution claim |

A deployment that omits the ContextGraph gateway must use a conforming verification SDK or equivalent implementation.

### 4.2 Inside and Outside the Product Boundary

| Inside ContextGraph boundary | Outside or optional boundary |
|---|---|
| Entity and metric resolution | Source-system business logic not represented as approved bindings |
| Ontology and tenant overlay versioning | Authoritative ERP transaction processing |
| Policy-aware plan compilation | Final LLM prose generation |
| Source routing and operation selection | Human business approval systems |
| Evidence, lineage, and decision records | Customer identity provider and source authorization enforcement |
| Resolution evaluation and telemetry | General enterprise ETL |
| Plan signing and verification contracts | Customer-specific connector operations |
| Bundle compilation and release | Customer network and vault operations |

---

## 5. Logical Architecture

```text
CONTROL PLANE
+----------------------------------------------------------------------------------+
| Tenant Admin | Ontology Registry | Metric Registry | Source Binding Studio       |
| Policy Admin | Mapping Workflow  | Version/Release | Evaluation Workbench        |
| Bundle Compiler | Connector Certification | Execution-Mode Configuration         |
+------------------------------------+---------------------------------------------+
                                     |
                      immutable, hashed, signed bundles
                                     v

DATA PLANE
+----------------------------------------------------------------------------------+
| API Gateway / Auth / Rate Limit / Idempotency / Tenant Derivation                |
+----------------------------------------------------------------------------------+
| Context Compiler Orchestrator                                                     |
|  + Domain and Intent Classifier                                                   |
|  + Resolution Evidence Producers                                                 |
|  + Candidate and Evidence Store                                                   |
|  + Calibrated Resolution Arbiter                                                  |
|  + Relationship Planner                                                          |
|  + Decision Guardrail Engine                                                      |
|  + Source Router                                                                  |
|  + Plan Compiler                                                                  |
|  + Explanation Builder                                                           |
+------------------------------------+---------------------------------------------+
                                     |
                   +-----------------+------------------+
                   |                                    |
                   v                                    v
+------------------------------------------+   +------------------------------------+
| PostgreSQL System of Record              |   | Redis / Read-Optimized Bundles     |
| bitemporal assertions, versions, policy, |   | aliases, compiled rules, caches,   |
| bindings, audit, idempotency, evidence   |   | active bundle pointers             |
+------------------------------------------+   +------------------------------------+
                   |
                   v
        Tenant-isolated vector indexes
        and optional graph adapters

EXECUTION PLANE
+----------------------------------------------------------------------------------+
| Signed Plan Validator | Version/Expiry/Replay Check | Local Policy Re-check      |
| Connector Supervisor  | Connector Adapters          | Normalizer                 |
| Secrets/Vault         | SAP/Snowflake/Oracle/PLM/API Drivers                     |
+----------------------------------------------------------------------------------+
```

---

## 6. Component Architecture

### 6.1 API Gateway and Edge

Terminates TLS, validates tokens, derives tenant from trusted claims, applies quotas, validates schemas, assigns trace and idempotency identifiers, and rejects untrusted tenant or source selection. Tenant selection is never trusted solely from a request header.

### 6.2 Context Compiler Orchestrator

Coordinates deterministic checks, evidence producers, graph validation, guardrails, routing, and plan compilation under a request latency and cost budget. It pins immutable semantic, policy, and API schema versions per accepted request.

### 6.3 Resolution Evidence Producers

Resolvers do not independently authorize a result. Each producer emits one or more evidence records with:

- candidate identifier
- producer and evidence type
- provenance
- raw score, when applicable
- calibrated likelihood, when evaluation supports it
- authority class
- latency and cost
- semantic version
- evidence references

Initial producers include:

- canonical identifier resolver
- approved alias resolver
- lexical and fuzzy resolver
- tenant-scoped vector resolver
- contextual and type resolver
- relationship validator
- optional structured LLM candidate proposer

### 6.4 Calibrated Resolution Arbiter

Combines evidence using producer-specific calibration, precedence rules, ambiguity thresholds, risk tier, and latency budget.

The arbiter implements:

- early return for unique, policy-valid exact matches
- suppression of expensive producers when confidence and authority are sufficient
- cancellation of optional work when the budget is exhausted
- explicit ambiguity handling
- evidence-aware abstention
- risk-tier-specific thresholds

Vector similarity, fuzzy scores, exact aliases, and LLM outputs are not treated as directly comparable without calibration.

### 6.5 Intent and Domain Classifier

Maps the request to an approved operation family and risk tier. LLM assistance may propose candidates but cannot independently authorize an operation.

### 6.6 Metric Resolver

Selects a versioned metric definition, dimensions, filters, allowed aggregations, and source bindings. Detects tenant-specific conflicts and temporal incompatibilities.

### 6.7 Relationship Planner

Finds typed, directed, effective-dated paths required by approved operation families. It distinguishes asserted, derived, inferred, ambiguous, overridden, and deprecated facts.

The planner uses:

1. direct indexed edge lookup
2. bounded traversal constrained by route templates
3. precomputed closures for selected high-value workflows
4. asynchronous deep traversal when the request exceeds the synchronous budget

It does not require a universal all-pairs materialized path index.

### 6.8 Decision Guardrail Engine

The outer decision layer composes multiple evaluators:

- identity authorization
- tenant and purpose authorization
- semantic admissibility
- temporal validity
- evidence sufficiency
- ambiguity policy
- business-risk approval policy
- source execution constraints
- geography and residency controls

A proven engine such as Cedar or OPA may handle RBAC/ABAC authorization, but semantic, temporal, confidence, and ambiguity rules remain explicit product-owned layers.

```text
Authorization
  × Semantic validity
  × Temporal validity
  × Evidence sufficiency
  × Ambiguity policy
  × Risk-tier approval
  × Source constraints
  = Guardrail outcome
```

Supported outcomes include:

- `PERMIT`
- `DENY`
- `CLARIFICATION_REQUIRED`
- `APPROVAL_REQUIRED`
- `STALE_CONTEXT`
- `INSUFFICIENT_EVIDENCE`
- `UNSUPPORTED`

### 6.9 Source Router

Selects authoritative or supplementary systems using data ownership, freshness, operation support, cost, residency, health, and availability. It never silently reroutes to an unapproved source.

### 6.10 Plan Compiler

Produces the connector-neutral execution-plan IR, required permissions, evidence references, pinned versions, expected result schema, execution mode, and verification requirements.

### 6.11 Explanation Builder

Generates a faithful explanation from recorded decisions and evidence rather than from a separate unconstrained LLM narrative.

### 6.12 Semantic Registry and Bundle Compiler

Stores and publishes ontology terms, metrics, relationships, operation schemas, policies, source bindings, route templates, and tenant overlays.

Published bundles use:

- canonical serialization
- cryptographic digest
- immutable object storage
- signed manifest
- atomic activation by version pointer
- local verification before load
- retention of previous versions for rollback

A component-level Merkle DAG may be introduced later if bundle size or cross-region transfer cost justifies it.

### 6.13 Evaluation Workbench

Runs golden-query suites, calibration tests, route accuracy, policy leakage tests, latency regression, ontology-version comparison, production correction review, and sampled shadow comparison.

### 6.14 Execution Gateway and Connector Supervisor

Validates plan signatures and versions, rechecks local policy, retrieves secrets, translates operations, executes, normalizes results, and returns source evidence.

Connector isolation baseline:

- connectors execute out of process
- narrow typed IPC or gRPC contract
- resource limits and timeouts
- signed connector manifests
- capability declaration and allowlist
- crash containment
- version compatibility tests

WASM/Wasmtime may be evaluated for lightweight HTTP, transformation, or customer-authored connectors. It is not mandatory for connectors requiring vendor-native drivers or unsupported system APIs.

---

## 7. Core Runtime Flows

### 7.1 Compile Request

1. Authenticate principal and agent; derive tenant and scopes from trusted claims.
2. Validate idempotency key scope and canonical request digest.
3. If a valid replay exists, return the original pinned result.
4. Otherwise pin published semantic, policy, API schema, and binding versions.
5. Normalize request text while preserving the original input.
6. Classify domain, operation family, and business-risk tier.
7. Run exact identifier and authoritative alias resolvers.
8. If a unique, policy-valid exact result is found, skip optional producers.
9. Otherwise run lexical, vector, contextual, and optional LLM producers under budget.
10. Fuse evidence and determine candidate ambiguity.
11. Validate required directed relationships and source freshness.
12. Evaluate authorization and semantic guardrails.
13. Compile a source-neutral typed plan and expected result schema.
14. Apply clarification or approval policy.
15. Persist the resolution event and idempotency record.
16. Return the signed response with evidence and pinned versions.

### 7.2 Execute Plan

1. Executor validates plan signature, tenant, expiry, nonce, pinned versions, and replay attributes.
2. Executor confirms that the configured execution mode is allowed.
3. Executor maps the source-neutral operation to a locally installed binding.
4. Local policy and source credentials are checked independently.
5. Connector executes with read-only or explicitly approved action scope.
6. Gateway or delegated executor normalizes results to the declared schema.
7. Source freshness, partial-result state, and evidence are attached.
8. Only the minimum necessary result is returned.

### 7.3 Ambiguous Request

When competing interpretations remain above the ambiguity threshold, the compiler returns candidate meanings and a machine-readable clarification contract. It does not silently select the highest-scoring candidate for high-risk operations.

### 7.4 Idempotent Replay

For the same tenant, operation namespace, idempotency key, and canonical request digest:

- return the original outcome
- preserve original semantic, policy, API, and binding versions
- increment replay count
- emit retry-storm telemetry when appropriate

If the same key is reused with a different request digest or explicitly requested version, return `409 IDEMPOTENCY_KEY_REUSED`.

Compile and execute use separate idempotency namespaces.

---

## 8. Typed Execution-Plan Intermediate Representation

The execution-plan IR is a core owned asset. It decouples business semantics from SAP tables, Snowflake SQL, or vendor-specific APIs. The IR is versioned, signed, deterministic to validate, and safe for policy evaluation.

```json
{
  "schema_version": "1.1",
  "plan_id": "pln_01...",
  "tenant": "derived-from-auth",
  "semantic_bundle": "semiconductor@2026.06.3",
  "semantic_bundle_digest": "sha256:...",
  "policy_bundle": "policy@2026.06.9",
  "api_schema": "compile@1.1",
  "decision": "RESOLVED",
  "risk_tier": "PLANNING_DECISION",
  "execution_mode": "PRIVATE_GATEWAY",
  "operation": "procurement.open_purchase_order_quantity",
  "arguments": {
    "supplier": {"entity_id": "SUP-0042"},
    "buyer_entity": {"entity_id": "ENT-001"},
    "horizon": {"start": "2026-07-01", "end": "2026-09-30"}
  },
  "metric": {"id": "open_po", "version": "3.2"},
  "source_binding": "sap-mm-prod@7",
  "required_permissions": ["procurement.po.read"],
  "expected_result_schema": "quantity_by_currency@1",
  "evidence_refs": ["ev_..."],
  "verification_requirements": [
    "signature",
    "expiry",
    "nonce",
    "tenant",
    "binding_version",
    "local_policy"
  ],
  "expires_at": "2026-06-28T18:00:00Z",
  "signature": "..."
}
```

| IR field | Requirement |
|---|---|
| Operation | Stable business operation identifier; never a raw SQL fragment |
| Arguments | Typed entities, dates, dimensions, filters, and thresholds |
| Metric | Approved definition and version |
| Source binding | Versioned mapping to an environment-specific connector |
| Execution mode | Managed, private, delegated, or compile-only |
| Permissions | Machine-readable scopes required at execution time |
| Expected result schema | Enables normalization and validation |
| Evidence | References to entity, metric, relationship, and routing evidence |
| Bundle digest | Enables integrity validation |
| Verification requirements | Makes transferred execution obligations explicit |
| Expiry/signature/nonce | Prevents replay and modification |

---

## 9. Data and Semantic Model

```text
Tenant
  -> SemanticBundle
      -> OntologyVersion
      -> MetricDefinitionVersion
      -> PolicyBundleVersion
      -> RouteTemplateVersion
      -> SourceBindingVersion

Entity --HAS_ALIAS--> EntityAlias
Entity --[Typed Relationship Assertion]--> Entity
Relationship Assertion --RESOLVED_AS--> Accepted Relationship Fact
MetricDefinition --BOUND_TO--> SourceOperation
SourceOperation --IMPLEMENTED_BY--> ConnectorBinding
ResolutionEvent --USES--> Bundle Versions / Evidence / GuardrailDecision
```

| Entity/table | Key properties |
|---|---|
| tenant | id, status, residency, deployment mode, default legal entities |
| principal/agent registration | identity mapping, scopes, allowed purposes, risk limits |
| entity | canonical id, type, tenant/global scope, lifecycle state |
| entity_alias | normalized form, locale, source, scope, valid time, system time, status |
| relationship_assertion | source, target, type, direction, valid time, system time, evidence, assertion class, source authority |
| accepted_relationship_fact | selected assertion set, bundle version, decision rationale, valid time, system time |
| ontology_term/version | hierarchy, constraints, release state, compatibility metadata |
| metric_definition/version | formula semantics, grain, dimensions, exclusions, owner, approval |
| route_template/version | operation family, allowed edge sequence, direction, depth, temporal constraints |
| relationship_closure | selected precomputed workflow paths, source bundle, validity, invalidation state |
| source_system | environment, type, residency, owner, health, freshness expectations |
| source_binding/version | operation mapping, required parameters, result schema, connector capability |
| policy/version | subject, resource, action, purpose, condition, effect, priority |
| evidence | source type, locator, checksum, steward, confidence class, timestamps |
| candidate_evidence | candidate, producer, authority, raw score, calibrated likelihood, evidence refs, timing |
| resolution_event | input hash, decisions, candidates, evidence, versions, timing, outcome |
| execution_plan | IR, signature, expiry, state, idempotency, execution references |
| idempotency_record | tenant, namespace, key, request digest, pinned versions, outcome, TTL, replay count |
| semantic_bundle_manifest | version, digest, object locator, signature, activation state, compatibility |
| evaluation_case | input, expected status, observed outcomes, steward decision, provenance |

### 9.1 Bitemporal Semantics

Operationally meaningful definitions, aliases, assertions, accepted facts, source ownership, and mappings support two independent time dimensions:

- **valid time:** when the fact was true in the business world
- **system time:** when ContextGraph knew or accepted the fact

Typical storage representation:

```sql
valid_time  tstzrange NOT NULL,
system_time tstzrange NOT NULL
```

The system must support queries such as:

> What did we believe on June 1 about the relationship that was valid on May 15?

Assertions and accepted facts are stored separately so that conflicting sources, corrections, and backdated updates do not erase historical knowledge.

Partitioning and indexes are selected from measured access patterns. Tenant-hash partitioning is the default candidate for high-cardinality shared tables; time subpartitioning is introduced only where volume justifies it.

### 9.2 Tenant Overlay Model

```text
Global upper ontology
  -> Supply-chain base pack
      -> Semiconductor extension
          -> Customer semantic overlay
              -> Environment-specific source bindings
```

Overrides are explicit, reviewed, conflict-checked, versioned, and reversible. A tenant overlay must not mutate a shared base pack.

### 9.3 Tenant-Isolated Vector Retrieval

The base embedding model may be shared, but tenant data is not co-mingled in retrieval.

Controls include:

- per-tenant index, collection, or enforced namespace
- tenant scope below the application layer where supported
- tenant-scoped cache keys
- no cross-tenant nearest-neighbor search
- no shared-model fine-tuning on tenant data without separate governance
- physical vector isolation for regulated deployments
- isolation tests and membership-inference testing for high-risk environments

Tenant-specific projection layers may be used for domain adaptation but are not treated as the primary security boundary.

---

## 10. Resolution Evidence and Decision Process

The resolver uses a deterministic fast path followed by adaptive evidence production.

| Producer or check | Technique | Trust role |
|---|---|---|
| Canonical identifier | ERP identifiers, registered external IDs | Deterministic authoritative match |
| Approved alias | Case, punctuation, locale, approved scope | Deterministic authoritative match |
| Lexical/fuzzy | Token, phonetic, and edit-distance matching | Candidate generation |
| Tenant-scoped vector | Embeddings over approved names and descriptions | Candidate generation only |
| Contextual scoring | Entity type, operation, neighboring terms, tenant defaults | Rule/model-assisted evidence |
| Relationship validation | Typed paths, direction, validity, evidence | Deterministic validation |
| Optional LLM proposal | Structured candidate selection or missing-term proposal | Untrusted until validated |
| Guardrail decision | Calibration, risk tier, authorization, ambiguity, abstention | Deterministic release decision |

### 10.1 Adaptive Execution

The orchestrator may:

- stop after a unique exact match
- skip vector or LLM work when confidence and authority are sufficient
- run compatible producers concurrently
- cancel remaining work when the decision threshold is reached
- return partial evidence with a degradation signal
- abstain rather than exceed risk or latency thresholds

### 10.2 Evidence Fusion

Scores are producer-specific. A cosine similarity, alias match, edit-distance score, and LLM ranking are not directly summed.

The arbiter uses:

- producer-specific calibration
- source authority
- temporal validity
- candidate consistency
- relationship support
- risk-tier threshold
- ambiguity margin
- evidence freshness
- explicit precedence rules

Confidence is exposed only where evaluation data supports calibration. Public responses also include assertion class, evidence strength, reason codes, and confirmation requirements.

---

## 11. Graph Infrastructure and Open-Source Reuse

Graphify may be reused selectively for ingestion patterns, content hashing, extraction schemas, confidence classifications, MCP patterns, and internal visualization. NetworkX and file-oriented graph state are not recommended for production runtime.

Graphiti may be evaluated as a temporal graph and hybrid-retrieval substrate behind an internal interface. ContextGraph-specific policy, metrics, source bindings, execution IR, accepted-fact logic, and evaluation remain owned product layers.

| Capability | Build / reuse decision |
|---|---|
| Temporal graph facts and provenance | Evaluate Graphiti behind internal interface |
| Document/code/schema ingestion | Adapt Graphify patterns or parsers |
| Production system of record | Build on PostgreSQL |
| Business metric registry | Build and own |
| Decision guardrails | Integrate proven authorization evaluator; own semantic/temporal/risk model |
| Execution-plan IR | Build and own |
| SAP/Snowflake adapters | Build through connector SDK |
| Connector isolation | Out-of-process baseline; evaluate WASM selectively |
| Visualization | Reuse for internal tooling; replace for production UX |
| Resolution evaluation corpus | Build and own |
| Workflow route templates and closures | Build selectively for approved operation families |

### 11.1 Relationship Traversal Strategy

The default PostgreSQL graph model uses indexed typed edges and bounded traversal.

Performance progression:

1. adjacency queries and bounded recursive traversal
2. route templates that constrain edge type, direction, and maximum depth
3. targeted precomputed closures for approved workflow families
4. specialized graph adapter only when benchmarks show material benefit

Universal path materialization is not an MVP requirement because it creates write amplification, invalidation complexity, and storage growth without guaranteed product value.

---

## 12. External API and SDK Architecture

| Endpoint | Purpose |
|---|---|
| `POST /v1/compile` | Resolve and compile a governed execution plan |
| `POST /v1/clarifications/{id}` | Submit selected clarification or additional context |
| `GET /v1/resolutions/{id}` | Retrieve decision, evidence, versions, and explanation |
| `POST /v1/plans/{id}/execute` | Optional managed/private execution trigger |
| `POST /v1/plans/{id}/verify` | Verify a plan for delegated execution |
| `GET /v1/metrics/{id}` | Retrieve an approved metric definition |
| `GET /v1/entities/{id}` | Retrieve an authorized canonical entity view |
| `GET /v1/bundles/{version}/manifest` | Retrieve signed bundle manifest and digest |
| `GET /v1/health` and `/ready` | Operational health and dependency readiness |

SDKs should be generated from OpenAPI where possible and augmented with typed helpers for compile, clarify, verify, and execute flows. MCP tools should expose business-level actions rather than generic graph traversal primitives.

Delegated-execution SDKs must provide:

- signature verification
- key discovery and rotation support
- schema validation
- expiry and nonce validation
- tenant and binding validation
- compatibility tests
- reason-code mapping

| Cross-cutting API requirement | Implementation |
|---|---|
| Tenant security | Tenant derived from trusted token; explicit tenant allowed only for authorized operators |
| Idempotency | Separate compile and execute namespaces; original outcome replayed for identical requests |
| Traceability | request_id, trace_id, resolution_id, plan_id |
| Version pinning | API schema, semantic bundle, policy bundle, connector binding |
| Bundle integrity | Canonical digest and signature |
| Errors | Stable problem-details taxonomy with machine-readable reason codes |
| Pagination | Cursor-based for registry and evidence collections |
| Rate limits | Tenant and principal quotas with retry metadata |
| Compatibility | Additive minor versions; explicit deprecation windows |

---

## 13. Security, Privacy, and Governance

### 13.1 Threat Model

- Cross-tenant data disclosure through cache keys, graph traversal, logs, vector retrieval, or inference.
- Prompt or data injection that attempts to alter ontology, policy, or source selection.
- Confused-deputy execution caused by client-supplied tenant or source identifiers.
- Plan tampering, replay, or execution after semantic/policy expiration.
- Over-broad connector credentials and data exfiltration through generated queries.
- Sensitive relationship leakage even when row-level source data is not returned.
- Inferred facts being treated as approved operational truth.
- Connector crashes or malicious plugins affecting gateway integrity.
- Bundle substitution, corruption, or inconsistent activation.
- Delegated executors omitting signature, policy, or schema validation.
- Production evaluation data retaining sensitive raw requests longer than necessary.

### 13.2 Controls

| Area | Controls |
|---|---|
| Identity | OIDC/OAuth2, service identities, agent registration, workload identity, short-lived tokens |
| Authorization | RBAC plus ABAC/purpose rules; compile and execute checks |
| Decision guardrails | Independent semantic, temporal, evidence, ambiguity, risk, and approval controls |
| Tenant isolation | RLS or database-per-tenant; tenant-scoped cache, graph, vector, encryption, and exports |
| Secrets | Customer vault integration; secrets remain in execution environment whenever possible |
| Plan integrity | Asymmetric signature, expiry, nonce, idempotency, pinned versions, bundle digest |
| Delegated execution | Verification SDK, conformance suite, key rotation, explicit transferred obligations |
| Network | PrivateLink/VPN/VPC peering, outbound allowlists, mTLS |
| Data protection | TLS, encryption at rest, field-level encryption for sensitive identifiers |
| Logging | No credentials; structured redaction; tenant-configurable retention |
| Connector isolation | Process boundary, resource limits, signed manifests, capability allowlist |
| Bundle integrity | Canonical serialization, digest verification, signed manifest, atomic activation |
| Supply chain | Pinned dependencies, SBOM, signed images, vulnerability scanning |
| Governance | Approval workflows, immutable releases, rollback, audit exports |
| Evaluation privacy | Sampling, redaction/tokenization, no shadow source execution, controlled retention |

---

## 14. Deployment Topologies

| Topology | Use case | Trade-offs |
|---|---|---|
| Multi-tenant SaaS compiler + private gateway | Default enterprise deployment | Best product velocity and credential isolation |
| Multi-tenant SaaS compiler + delegated executor | Customers with an existing execution platform | Requires conformance validation and SDK integration |
| Managed execution | Customers preferring a hosted integration plane | Higher ContextGraph operational responsibility |
| Single-tenant SaaS/VPC | High isolation or residency requirements | Higher operating cost; simpler customer risk review |
| Fully customer-hosted | Regulated or air-gapped customers | Slower upgrades; packaging and support discipline required |
| Embedded ISV runtime | OEM and product integrations | Strict compatibility, metering, and upgrade contracts |
| Compile-only | Planning, validation, or customer-controlled execution | No source execution support |
| Developer local mode | SDK evaluation and synthetic data | Not production |

```text
Cloud region
+-----------------------------+
| Load balancer / API gateway |
+-------------+---------------+
              v
+-----------------------------+      +---------------------------+
| Compiler service replicas   | ---> | Redis                     |
+-------------+---------------+      +---------------------------+
              |
              v
+-----------------------------+      +---------------------------+
| PostgreSQL HA               | ---> | Object storage / bundles  |
+-----------------------------+      +---------------------------+
              |
              +--> Tenant-isolated vector index
              |
              +--> Signed plans over mTLS/private link
                         |
                         v
Customer VPC: Gateway or Delegated Executor -> Vault -> SAP / Snowflake / PLM
```

---

## 15. Performance and Scalability

| Operation | Initial SLO |
|---|---|
| Cached exact entity resolution | p95 < 20 ms |
| Uncached deterministic resolution | p95 < 75 ms |
| Context compilation without source execution | p95 < 200 ms |
| Short bounded relationship traversal | p95 < 150 ms |
| Deep graph traversal | p95 < 500 ms or asynchronous |
| Bundle activation on replica | target < 5 s after announcement |
| Source execution | Source-dependent; separate SLO |
| Enterprise compiler availability | 99.9% after production maturity |

A request carries latency and optional cost budgets. The orchestrator may skip optional enrichment, use cached bundles, cancel evidence producers, return partial evidence with a degradation signal, or convert deep traversal to asynchronous work rather than silently exceed a hard budget.

### 15.1 Scaling Strategy

- Stateless compiler replicas scale horizontally.
- Immutable bundles are stored as canonical, compressed, content-hashed artifacts.
- PostgreSQL stores the source-of-truth release and active-version metadata.
- Replicas subscribe to version announcements but reconcile against PostgreSQL.
- Replicas verify the digest, load the new bundle, and atomically switch the active pointer.
- Previous versions remain available for rollback and idempotent replay.
- High-cardinality tables are partitioned by tenant and time only where measured.
- Common relationship closures are precomputed for approved workflow families.
- Read replicas serve registry and evidence-heavy reads.
- External dependencies use bounded concurrency and circuit breakers.
- Work queues handle ingestion, evaluation, closure rebuilds, and deep traversal.
- Vector indexes are tenant-isolated logically or physically.

### 15.2 Rust Decision Framework

The MVP remains Python/FastAPI.

Rust is recommended first for the customer execution gateway because static deployment, low memory, strong type safety, predictable latency, and a reduced runtime attack surface have direct commercial value.

Rust does not remove connector risk. Connectors must remain isolated through a process or sandbox boundary.

Resolver or graph services migrate only when:

- profiling shows Python CPU/runtime overhead exceeds 20% of end-to-end latency
- p95 or p99 remains unmet after data/query optimization
- per-tenant compute cost threatens margin
- measured concurrency requirements justify a dedicated service

### 15.3 Advanced Bundle Distribution

A Merkle DAG and component-level delta synchronization may be introduced when:

- bundle artifacts become large enough to materially affect rollout time
- multi-region bandwidth becomes significant
- subcomponent reuse yields measurable storage or transfer savings

CRDT metadata is not required while PostgreSQL remains the single publication authority.

---

## 16. Reliability and Failure Handling

| Failure | Required behavior |
|---|---|
| Redis unavailable | Fall back to PostgreSQL/read bundle with degraded-latency signal |
| Graph backend unavailable | Use deterministic registry path or return partial/unsupported; never invent paths |
| Vector index unavailable | Continue deterministic path; emit degraded retrieval state |
| Policy service unavailable | Fail closed for protected operations |
| Bundle announcement missed | Reconcile active version against PostgreSQL |
| Bundle digest mismatch | Reject activation and keep previous verified bundle |
| Source gateway unavailable | Return plan plus execution-unavailable state; do not reroute to unapproved source |
| Connector crash | Contain failure to connector process and return source-specific failure |
| Delegated verifier incompatible | Reject execution with a compatibility reason code |
| Stale semantic bundle | Return `STALE_CONTEXT` or use an approved previous bundle per tenant policy |
| LLM unavailable | Continue deterministic path; return insufficient context if model-only enrichment was required |
| Partial multi-source result | Return source-level status and no misleading aggregate |
| Duplicate request | Return prior version-pinned result when digest matches |
| Idempotency key conflict | Return `409 IDEMPOTENCY_KEY_REUSED` |
| Shadow evaluation unavailable | Do not affect production result |

All services implement timeouts, bounded retries with jitter, bulkheads, circuit breakers, and explicit degradation modes. Policy, semantic, route-template, and connector releases support rollback.

---

## 17. Observability and Audit

| Signal | Examples |
|---|---|
| Metrics | Latency by producer, cache hit rate, ambiguity rate, abstention rate, route accuracy, policy denials, plan validation failures |
| Evidence metrics | Producer contribution, cancellation rate, calibration drift, exact-match fast-path rate |
| Traces | API -> evidence producers -> arbiter -> graph -> guardrails -> plan |
| Logs | Structured decision events with reason codes and redacted identifiers |
| Audit | Who changed definitions, who approved releases, which versions answered each request |
| Quality | Top-1 accuracy, recall@k, expected calibration error, valid-path precision, unauthorized disclosure rate |
| Isolation | Cross-tenant query rejection, vector namespace violations, cache-key isolation failures |
| Bundle operations | Activation time, digest mismatch, rollback frequency, version skew |
| Connector health | Crash count, timeout count, capability violations, version compatibility |
| Business | Workflow completion, analyst time saved, correction rate, adoption by agent/use case |
| Evaluation loop | Clarification outcomes, corrections, production/candidate divergence, steward review backlog |

Explanation text is generated from the same structured decision record used for audit.

---

## 18. Engineering, CI/CD, and Release Management

- Monorepo initially, with clear package boundaries for compiler, registry, policies, connectors, SDKs, evaluation, and bundle distribution.
- Trunk-based development with short-lived branches, mandatory review, and automated schema compatibility checks.
- Unit, property, contract, integration, security, temporal, isolation, and golden-query tests in CI.
- Database migrations are forward-compatible and rehearsed against production-like volumes.
- Containers and connector packages are signed.
- SBOM and vulnerability reports are generated per release.
- Semantic, policy, route-template, and connector bundles have independent approvals, canaries, and rollback.
- Bundle artifacts are canonically serialized and digest-verified.
- Connector compatibility matrices are tested against supported source versions.
- Delegated executor SDKs have conformance suites.
- Production shadow comparison never executes source operations.

| Environment | Purpose |
|---|---|
| Local | Synthetic tenant, local PostgreSQL/Redis, mock gateway |
| Development | Shared services, fast bundle iteration |
| Staging | Production topology, masked/synthetic data, connector certification |
| Canary | Limited tenants or traffic with automated rollback |
| Production | Regional HA, immutable artifacts, controlled releases |

---

## 19. Verification and Evaluation Strategy

| Test family | Coverage |
|---|---|
| Resolver unit tests | Normalization, aliases, producer evidence, ambiguity, temporal validity |
| Evidence-fusion tests | Calibration, precedence, cancellation, threshold behavior |
| Golden business queries | Human-approved input, resolution, route, expected plan |
| Policy leakage tests | Cross-tenant, unauthorized field/entity/relationship attempts |
| Vector isolation tests | Namespace enforcement, cache isolation, cross-tenant retrieval attempts |
| Temporal tests | As-of behavior, superseded mappings, backdated corrections |
| Connector contract tests | Operation translation, capability limits, result normalization |
| Connector isolation tests | Crash containment, resource exhaustion, invalid manifest |
| Delegated execution tests | Signature, key rotation, expiry, nonce, schema, binding compatibility |
| Calibration tests | Confidence/reliability curves by query class and tenant |
| Chaos tests | Cache, database, graph, vector, policy, gateway, and source failures |
| Performance tests | Alias/relationship cardinality, concurrency, closure rebuilds |
| Bundle tests | Canonical hash, atomic activation, rollback, version skew |
| Semantic regression | Compare bundle versions against golden corpus before publish |
| Production divergence | Sampled current-versus-candidate comparison without source execution |
| Correction loop | User clarification and steward correction ingestion |

### 19.1 Continuous Evaluation Loop

Production evidence may propose new evaluation cases from:

- user clarification selections
- steward overrides
- explicit corrections
- execution failures caused by resolution or binding errors
- sampled production/candidate divergence
- downstream business validation where contractually available

No observed outcome is automatically treated as ground truth.

```text
Observed production outcome
Observed candidate outcome
        |
        v
PENDING_STEWARD_REVIEW
        |
        +--> confirmed expected outcome
        +--> rejected
        +--> retained as ambiguous
```

Controls include:

- configurable sampling
- risk-tier-specific rates
- deterministic query hashing
- redacted or tokenized input
- no shadow source execution
- controlled retention
- explicit steward approval

The system is not evaluated through one aggregate “accuracy” figure. Accuracy, recall, calibration, abstention, policy safety, latency, isolation, and risk tier are measured independently.

---

## 20. Phased Delivery Architecture

| Phase | Scope | Exit criteria |
|---|---|---|
| Phase 0 — Architecture spike | PostgreSQL graph vs Graphiti; one SAP/Snowflake workflow; typed IR; evidence arbiter prototype | Evidence-based infrastructure decision and benchmark |
| Phase 1 — MVP | One tenant, one workflow family, 10–15 metrics, compile API, private gateway, guardrail engine, hashed bundles, tenant-isolated vectors, evaluation | 100–300 golden queries; p95 target; signed expansion path |
| Phase 2 — Productize | Second/third customer, connector SDK, process isolation, admin workflows, stronger policy, telemetry, production correction loop | Repeatable onboarding and low regression rate |
| Phase 3 — Scale | Multi-region, delegated execution SDK, targeted closures, additional connectors, self-service mapping | Operational scalability and partner distribution |
| Phase 4 — Platform | Additional industry packs and marketplace after repeatable economics | Cross-industry reuse without weakening governance |
| Conditional advanced work | WASM connector runtime, Merkle DAG bundle deltas, specialized graph service | Introduced only against measured need |

---

## 21. Key Architecture Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Becoming a generic graph product | Weak differentiation | Keep graph behind interfaces; invest in IR, policy, metrics, bindings, and evaluation |
| Semantic onboarding becomes consulting-heavy | Poor margins | Importers, templates, stewardship workflow, reusable bindings, explicit implementation pricing |
| LLM-inferred facts enter production | Incorrect decisions | Candidate/approved/published states; deterministic validation and steward approval |
| Raw vendor queries leak into public API | Brittle contracts and security risk | Connector-neutral operations and gateway translation |
| Cross-tenant leakage | Critical breach | Trusted tenant derivation, RLS, scoped caches, vector isolation, graph isolation, private options |
| Connector failure compromises gateway | Availability or security incident | Out-of-process isolation, capabilities, limits, signed manifests |
| Delegated execution skips controls | Unauthorized or invalid execution | Verification SDK and conformance certification |
| Bundle version skew | Non-repeatable decisions | Digest verification, atomic activation, reconciliation, rollback |
| Overpromised latency | Commercial/SLA failure | Separate compilation, traversal, execution, and generation SLOs |
| Universal path precomputation explodes | Cost and invalidation burden | Restrict closures to approved workflow families |
| Misused confidence scores | False certainty | Producer-specific calibration and reason codes |
| Idempotency returns inconsistent results | Duplicate or stale operations | Replay original pinned outcome; reject request-digest conflict |
| Open-source dependency lock-in | Roadmap and maintenance risk | Internal abstraction, pinned versions, compatibility tests |
| Ontology content copied | Reduced moat | Focus on operational mappings, corrections, evaluation data, and workflow distribution |
| Evaluation loop captures sensitive data | Privacy breach | Sampling, redaction, retention limits, steward access controls |
| Premature advanced infrastructure | Slower delivery | Require benchmark, cost model, or customer requirement before adoption |

---

## 22. Architecture Decision Records — Updated Set

| ADR | Decision | Status |
|---|---|---|
| ADR-001 | ContextGraph compiles plans; direct execution is optional and isolated | Proposed |
| ADR-002 | PostgreSQL is the control-plane system of record | Proposed |
| ADR-003 | Public API exposes business operations, not SQL/SAP table hints | Proposed |
| ADR-004 | Tenant is derived from trusted identity claims | Proposed |
| ADR-005 | All semantic artifacts are versioned and effective-dated | Proposed |
| ADR-006 | LLMs generate candidates but cannot authorize execution | Proposed |
| ADR-007 | Graphiti is evaluated behind an internal graph abstraction | Proposed |
| ADR-008 | MVP runtime is Python/FastAPI; Rust starts at execution gateway | Proposed |
| ADR-009 | Policy is enforced at compile and execution boundaries | Proposed |
| ADR-010 | Explanations are generated from structured decision records | Proposed |
| ADR-011 | Resolution uses evidence-producing resolvers and a calibrated arbiter with a deterministic fast path | Proposed |
| ADR-012 | Managed, private, delegated, and compile-only execution modes have explicit security invariants | Proposed |
| ADR-013 | Authorization and semantic/temporal/risk guardrails are distinct layers | Proposed |
| ADR-014 | Idempotency replays the original version-pinned result and rejects key/body conflicts | Proposed |
| ADR-015 | Vector retrieval is tenant-isolated; cross-tenant candidate indexes are prohibited by default | Proposed |
| ADR-016 | Semantic bundles use canonical serialization, cryptographic digests, atomic activation, and rollback | Proposed |
| ADR-017 | Production corrections and sampled shadow comparisons feed a steward-reviewed evaluation corpus | Proposed |
| ADR-018 | Connector isolation is contractually defined; process isolation is the baseline and WASM is evaluated selectively | Proposed |
| ADR-019 | Relationship closures are materialized only for approved workflow families | Proposed |
| ADR-020 | Assertions and accepted relationship facts are stored separately with explicit bitemporal semantics | Proposed |

---

# Appendix A. Suggested Repository Structure

```text
/contextgraph
  /apps
    /api                 FastAPI edge and compile endpoints
    /admin               Control-plane administration service/UI
    /worker              Ingestion, evaluation, closure rebuild, deep traversal
    /gateway-rust        Customer-hosted execution gateway
    /connector-runner    Out-of-process connector supervisor/runtime
  /packages
    /contracts           Pydantic/JSON Schema/OpenAPI contracts
    /compiler            Orchestration and plan compiler
    /resolver            Evidence producers and candidate models
    /arbiter             Evidence fusion, calibration, decision thresholds
    /semantics           Ontology and bundle compiler
    /bundles             Canonical serialization, digest, activation
    /graph               Internal graph interface and adapters
    /routes              Route templates and workflow closures
    /policy              Authorization evaluator adapter
    /guardrails          Temporal, evidence, ambiguity, risk decisions
    /bindings            Operation and source-binding model
    /connectors          Connector SDK and reference adapters
    /verification        Plan verification SDK and conformance suite
    /evidence            Lineage and explanation records
    /evaluation          Golden suites, calibration, divergence review
    /observability       Metrics, tracing, audit helpers
  /ontology-packs
  /deploy
  /docs/adr
```

---

# Appendix B. MVP Reference Technology Stack

| Layer | Technology |
|---|---|
| API and compiler | Python 3.13, FastAPI, Pydantic, Uvicorn/Gunicorn-compatible process model |
| Execution gateway | Rust, Axum/Tonic, signed plan validator, connector supervisor |
| Connector isolation | Out-of-process runner using typed local gRPC/IPC; Wasmtime evaluation for suitable modules |
| Primary database | PostgreSQL 17+ with RLS, GiST ranges, pg_trgm, optional pgvector |
| Graph | PostgreSQL typed edges initially; Graphiti-backed adapter evaluation |
| Vector retrieval | Tenant-isolated pgvector collections or external vector namespaces |
| Cache/coordination | Redis |
| Bundle storage | Versioned object storage plus PostgreSQL manifest registry |
| Policy | Cedar/OPA-style authorization evaluator behind internal abstraction |
| Guardrails | Product-owned semantic, temporal, confidence, ambiguity, and risk evaluator |
| Messaging/jobs | Managed queue or Kafka only when volume requires it |
| Observability | OpenTelemetry, Prometheus-compatible metrics, centralized structured logs |
| Deployment | Kubernetes or managed container platform; Terraform/IaC |
| Secrets | Cloud KMS/Vault; customer vault for private/delegated execution |

---

# Appendix C. Definition of Done for Production Readiness

- Tenant isolation tests pass across database, cache, graph, vector, API, logs, evaluation data, and exports.
- No executable plan is emitted without semantic, policy, source-binding, API schema, and execution-mode versions.
- Golden-query and policy suites meet tenant-approved thresholds by risk tier.
- Producer-specific calibration is documented where confidence is exposed.
- Plan signatures, expiry, nonce, replay protection, and executor validation are verified.
- Delegated execution passes the conformance suite.
- Connector crashes and resource exhaustion are contained to the connector boundary.
- Bundle digest verification, atomic activation, version reconciliation, and rollback are proven.
- Idempotent replay returns the original pinned result; request-digest conflicts are rejected.
- Operational dashboards cover latency, failures, ambiguity, calibration drift, bundle skew, connector health, and policy denials.
- Production evaluation sampling is redacted, retention-controlled, and source-execution-free.
- Rollback is proven for application, database migration, semantic bundle, policy bundle, route template, and connector binding.
- Security review, threat model, SBOM, incident response, backup, and disaster recovery procedures are complete.
- Commercial SLOs explicitly exclude or separately define source execution and final model generation.
