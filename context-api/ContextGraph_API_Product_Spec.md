# ContextGraph API — Product Specification
### Agents don't need more data. They need business context at query time.

**Version:** 0.1 (Pre-Seed)  
**Date:** June 2026  
**Status:** Concept / Pre-Seed  
**Classification:** Confidential

---

## Table of Contents

1. [Vision & Thesis](#1-vision--thesis)
2. [Problem Statement](#2-problem-statement)
3. [Target Buyers & Personas](#3-target-buyers--personas)
4. [Competitive Landscape](#4-competitive-landscape)
5. [Differentiation](#5-differentiation)
6. [Product Architecture](#6-product-architecture)
7. [API Specification](#7-api-specification)
8. [Industry Ontology Packs](#8-industry-ontology-packs)
9. [MVP Scope & Roadmap](#9-mvp-scope--roadmap)
10. [Business Value & Outcomes](#10-business-value--outcomes)
11. [Go-to-Market Strategy](#11-go-to-market-strategy)
12. [Pricing & Packaging](#12-pricing--packaging)
13. [Risks & Mitigations](#13-risks--mitigations)

---

## 1. Vision & Thesis

### One-Line Pitch
> Enterprise agents fail because they don't understand company-specific meaning. ContextGraph gives agents a lightweight, governed business brain — resolving entities, relationships, metrics, and data routes in milliseconds before the agent acts.

### Core Thesis
Every enterprise AI agent deployment today faces the same hidden failure mode: the agent understands the question but not the business. "Pending orders from Hynix" means nothing to an LLM without knowing that Hynix is SK hynix, that "us" is the tenant company, that "pending" maps to `status IN ('open', 'partially_received')` in SAP, and that this data lives in a procurement module — not the data warehouse.

Current solutions tackle adjacent problems: semantic layers optimize SQL query performance for BI; data catalogs govern data assets; knowledge graphs store enterprise documents; ERP vendors lock context inside their own systems. None provide a lightweight, cross-system, agent-native **context resolution layer** that makes business meaning available at inference time.

ContextGraph is that layer.

### Design Principles
- **Lightweight over heavy**: An API call, not a 6-month ontology project
- **Additive not disruptive**: Works beside existing ERP, warehouse, and catalog investments
- **Agent-native**: Designed for agents that need answers in <200ms, not BI dashboards
- **Explainable by default**: Every answer carries its reasoning chain
- **Industry-specific**: Pre-built ontology packs beat blank-slate frameworks

---

## 2. Problem Statement

### The Agent Failure Pattern

Enterprise AI agents fail in a predictable sequence:

1. User asks a natural language question with business-specific entities
2. Agent passes raw question to an LLM or NL-to-SQL model
3. LLM makes plausible but wrong assumptions ("pending" ≈ `status = 'pending'`)
4. Query hits the wrong system or returns wrong records
5. Agent returns a confident but incorrect answer
6. Business user loses trust; agent project stalls

This failure is not a model problem. It is a **context problem**.

### Evidence

- Cloudera/HBR (March 2026): Only 7% of enterprises say their data is completely ready for AI
- MIT "State of AI in Business 2025": Most enterprise AI deployments fail due to "brittle workflows, lack of contextual learning, and misalignment with day-to-day operations"
- Gartner projects that by 2028, more than 50% of AI agent systems will leverage context graphs as foundational infrastructure
- SAP's acquisition of Reltio (announced March 2026) signals that ERP vendors recognize the entity resolution gap

### The Cost of Bad Context

For a semiconductor supply chain team at a company like NVIDIA, a wrong answer to "which products are exposed to HBM supply risk?" is not a data quality issue — it is a planning and inventory decision worth hundreds of millions of dollars. Bad agent context = bad business decisions.

### Why Nothing Existing Solves It

| Problem | Who "solves" it today | Why it falls short |
|---|---|---|
| Entity synonyms ("Hynix" = "SK hynix") | MDM / Reltio / Informatica | Heavy, months-long deployment, not agent API |
| Metric definitions ("backlog" vs. "open PO") | dbt semantic layer / AtScale | SQL-centric, not agent-ready relationship context |
| Relationship traversal (supplier → component → SKU) | Knowledge graphs / Neo4j | Schema-heavy, no pre-built supply chain logic |
| Source routing (which system has this answer?) | None | Gap — this does not exist as a product |
| Industry ontology (semiconductor terms) | None as API | Gap — only academic OWL files or consulting deliverables |
| Explainability (why did agent do that?) | None | Gap — agents are black boxes today |

---

## 3. Target Buyers & Personas

### Primary Buyers

**1. AI/ML Platform Teams** — Internal builders deploying enterprise copilots and agentic workflows  
- Title: VP AI Platform, Head of AI Engineering, Principal ML Engineer  
- Pain: Spend 60–80% of agent project time on data wiring and disambiguation logic  
- They want: A clean API that handles business context so they can focus on agent logic  
- Budget owner: Engineering or Data Platform budget  

**2. Data & Analytics Leaders** — Owners of the semantic and governance layer  
- Title: CDO, VP Data, Head of Data Engineering  
- Pain: Agents bypass governance; semantic layers are for BI, not agents  
- They want: Governed business meaning exposed to agents without rebuilding the stack  
- Budget owner: Data & Analytics budget  

**3. Supply Chain Operations Leaders** — Domain owners demanding faster, better answers  
- Title: VP Supply Chain, Head of Procurement, Supply Chain Ops Director  
- Pain: Spend hours in analyst meetings disambiguating data; no self-serve for agent queries  
- They want: Agents that answer supply chain questions correctly the first time  
- Budget owner: Operations budget  

**4. ISVs / AI Agent Builders** — Vendors embedding agents into enterprise products  
- Title: CPO, Head of Product, Platform Architect at SAP partners, ServiceNow ISVs, etc.  
- Pain: Their agents break on every customer's data model and terminology  
- They want: A context API they can bundle into their agent product  
- Budget owner: Product development budget  

### Secondary Buyers (Near-Term Expansion)
- Semiconductor manufacturers (Intel, AMD, Qualcomm, Micron)
- Automotive OEMs with complex multi-tier supply chains
- Pharma companies (regulatory + supply chain complexity)

---

## 4. Competitive Landscape

### Category Map

```
                        AGENT-NATIVE
                             ↑
                    [ ContextGraph API ]  ← Target position
                             |
     DOCUMENT-CENTRIC ←──────┼──────→ STRUCTURED DATA-CENTRIC
      (Glean, Guru)          |         (dbt, AtScale, Cube.dev)
                             |
                        DATA MGMT
                     (Reltio/SAP, Informatica)
                             ↓
```

### Direct Competitors

**None exist in the exact space.** This is a greenfield product category. However, adjacent players will inevitably expand into it.

### Adjacent Players — Detailed Analysis

#### Tier 1: Semantic Layer Vendors (BI-centric)

| Vendor | What they do | Why they won't beat ContextGraph |
|---|---|---|
| **dbt Semantic Layer (MetricFlow)** | Define metrics in dbt project; expose via SQL | SQL-only, no entity resolution, no relationship traversal, no source routing. Requires dbt adoption. No industry ontologies. |
| **AtScale** | Enterprise OLAP semantic layer; MDX/DAX for Excel/Power BI | BI-first heritage, limited agent support. No cross-system routing. No supply chain ontology. |
| **Cube.dev** | Open-source semantic layer; REST/GraphQL/MCP APIs | Metric-focused, not entity/relationship-aware. No industry context packs. No NL disambiguation. |
| **Snowflake Semantic View Autopilot** | Auto-generate semantic views via AI | Platform-locked. Snowflake data only. No ERP routing. No entity resolution for business terms. |

The Open Semantic Interchange (OSI) standard (Jan 2026, backed by Snowflake, Salesforce, dbt, Cube, AtScale) standardizes **metric metadata in YAML** — but says nothing about entity resolution, relationship graphs, source routing, or industry ontologies. OSI is a format; ContextGraph is a runtime.

#### Tier 2: Enterprise Knowledge Graphs / Search

| Vendor | What they do | Why they won't beat ContextGraph |
|---|---|---|
| **Glean** | Enterprise graph over documents, emails, Slack, wikis | Document/people-graph focus. Not structured ERP data. Not agent API. No supply chain ontology. ~$7B valuation building toward interface layer — but above the data plane, not in it. |
| **Guru** | Verified knowledge base + AI assistant | Document QA, not structured business context. No relationships, no source routing. |
| **Metaphor Data** | Data lineage + discovery (acquired by dbt) | Data catalog focus. Not agent runtime context. |

#### Tier 3: MDM / Data Quality

| Vendor | What they do | Why they won't beat ContextGraph |
|---|---|---|
| **Reltio** (acquired by SAP) | Cloud-native MDM; entity resolution | Heavy platform deployment (months). SAP-centric post-acquisition. No agent API. No ontology packs. |
| **Informatica** | Broad data management; AI agent data infrastructure guides | Historically slow, enterprise-heavy. No lightweight API. Not agent-native runtime. |
| **Senzing** | Embedded entity resolution engine | Developer library, not business context API. No ontology, no routing, no explainability. |

#### Tier 4: ERP Vendors (Walled Gardens)

| Vendor | What they do | Why they won't beat ContextGraph |
|---|---|---|
| **SAP Joule + Knowledge Graph** | Natural language copilot grounded in SAP data | SAP-only. Multi-system customers (SAP + Oracle + Snowflake) get no help. Competitor's customers can't use it. |
| **Oracle Fusion AI Agents** | NL query over Oracle SCM modules | Oracle-only. Same limitation. |

**Key insight**: The larger SAP and Oracle get in agent AI, the more they validate the problem — and the more their non-SAP/Oracle customers need a vendor-neutral solution.

#### Tier 5: Knowledge Graph Platforms (Infrastructure-heavy)

| Vendor | What they do | Why they won't beat ContextGraph |
|---|---|---|
| **Stardog** | Enterprise knowledge graph; SPARQL | Complex deployment. Requires building ontologies from scratch. Not agent API. |
| **PoolParty** | Semantic middleware; taxonomy management | Technical, not business-user oriented. No industry packs. |
| **Neo4j** | Graph database | Requires custom modeling. No business context layer. Competes with underlying infra, not with ContextGraph's API layer. |

#### Emerging / Potential Threats

- **Tellius**: Has a purpose-built enterprise context layer (metric ontology + knowledge graph). Closest adjacent threat. But BI/analytics-centric, not agent API.
- **AI agent platforms** (LangChain, Semantic Kernel, Vertex AI Agent Builder): May add context resolution as a feature. Risk: generic, not industry-specific.
- **Data catalog vendors** (Collibra, Alation): Moving toward AI governance. Risk if they add entity resolution + routing.

### Competitive Summary Table

| Capability | ContextGraph | dbt/AtScale | Glean | Reltio/SAP | Knowledge Graphs |
|---|:---:|:---:|:---:|:---:|:---:|
| Agent-native REST API | ✅ | Partial | ❌ | ❌ | ❌ |
| Entity resolution (business terms) | ✅ | ❌ | Partial | ✅ | ❌ |
| Industry ontology packs | ✅ | ❌ | ❌ | ❌ | ❌ |
| Supplier/component relationship graph | ✅ | ❌ | ❌ | ❌ | Custom |
| Cross-system source routing | ✅ | ❌ | ❌ | ❌ | ❌ |
| Metric disambiguation | ✅ | ✅ | ❌ | Partial | ❌ |
| Answer lineage / explainability | ✅ | Partial | Partial | ❌ | ❌ |
| Multi-ERP support | ✅ | Partial | ❌ | SAP only | Custom |
| Sub-200ms API latency | ✅ | ✅ | N/A | ❌ | ❌ |
| Pre-built supply chain context | ✅ | ❌ | ❌ | ❌ | ❌ |

---

## 5. Differentiation

### The Three Moats

**1. Industry Ontology Packs (Depth moat)**  
Pre-built, curated, validated supply chain and semiconductor ontologies are not replicable quickly. Building accurate domain ontologies requires domain experts + customer validation + iteration. First-mover gets the feedback loop. The supply chain base ontology + semiconductor extension is 12–18 months ahead of any generic platform that tries to catch up.

**2. Cross-System Source Router (Network moat)**  
The more connectors ContextGraph has (SAP, Oracle, Snowflake, Databricks, PLM, CRM, spreadsheets), the more valuable the routing layer becomes. Each new connector makes existing customers' agents smarter. Classic network-effect moat.

**3. Customer Context Graphs (Switching moat)**  
Once a customer has mapped their entities, customized their ontology, and built their synonym libraries, the cost of switching is high. The context graph becomes institutional memory — tuned to their terminology, their supplier names, their metric definitions.

### The Core Differentiating Message

> **Not a graph database. Not a data catalog. Not generic RAG. A context engine for agents.**

| We are NOT | We ARE |
|---|---|
| Another BI semantic layer | A runtime agent context API |
| A graph database (Neo4j, etc.) | An application layer above the graph |
| An ontology platform (months to deploy) | A lightweight API pack (days to integrate) |
| A data catalog (Collibra, Alation) | A live query-time resolver |
| A RAG pipeline | A structured context layer (no hallucination surface for entities) |
| SAP- or Oracle-locked | Vendor-neutral across any enterprise stack |

---

## 6. Product Architecture

### System Architecture

```
┌─────────────────────────────────────────────────────┐
│                   User Question                      │
│  "How many orders are pending from SK hynix for us?" │
└─────────────────────────┬───────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────┐
│              Agent / Copilot Layer                   │
│    (LangChain, Semantic Kernel, Claude, GPT-4o)     │
└─────────────────────────┬───────────────────────────┘
                          ↓ API call
┌─────────────────────────────────────────────────────┐
│              ContextGraph API Layer                  │
│                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │  Entity     │  │  Ontology    │  │ Relationship│ │
│  │  Resolver   │  │  Engine      │  │ Graph      │ │
│  └─────────────┘  └──────────────┘  └────────────┘ │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │  Metric     │  │  Source      │  │ Lineage &  │ │
│  │  Dictionary │  │  Router      │  │ Explainer  │ │
│  └─────────────┘  └──────────────┘  └────────────┘ │
│                                                      │
│  Industry Ontology Pack: Supply Chain / Semiconductor│
│  Customer Context Layer: Tenant-specific mappings    │
└─────────────────────────┬───────────────────────────┘
                          ↓ routed query
┌─────────────────────────────────────────────────────┐
│              Enterprise Data Systems                 │
│   SAP │ Oracle │ Snowflake │ Databricks │ PLM │ CRM │
└─────────────────────────┬───────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────┐
│           Answer + Lineage + Confidence              │
└─────────────────────────────────────────────────────┘
```

### Core Components

#### 1. Entity Resolver
Resolves natural language entity references to canonical business identifiers.

- **Tenant context**: "us" / "we" / "our company" → tenant's canonical entity ID
- **Supplier aliases**: "Hynix" / "SK Hynix" / "SK-Hynix" → `supplier_id: SKH-001`
- **Status terms**: "pending" → context-dependent (`open_po`, `unconfirmed`, `in_transit` based on domain)
- **Abbreviations**: "HBM" → "High Bandwidth Memory" → component class → related SKUs
- **Fuzzy matching + confidence scoring**: Returns alternatives when ambiguous

#### 2. Ontology Engine
Applies industry ontology rules and customer-specific extensions.

- Loads base ontology pack (supply chain, semiconductor) at startup
- Applies customer extension layer (custom terms, overrides)
- Determines domain classification for the query
- Provides term hierarchy (HBM3E is-a HBM is-a memory component)

#### 3. Relationship Graph
Traverses the supplier-to-product relationship graph.

- **Supply chain graph**: Supplier → Component → BOM → SKU → Product → Customer/Order
- **Organizational graph**: Legal entity → Division → Business unit → Cost center
- **Data system graph**: Entity → owning system → relevant tables/APIs
- Bidirectional traversal: "Which products use this component?" and "What components are in this product?"

#### 4. Metric Dictionary
Governs metric definitions and disambiguation.

- Standard supply chain metrics with precise definitions
- Disambiguation logic: "backlog" vs "open PO" vs "committed supply" vs "pending orders"
- Cross-system metric harmonization (same metric, different field names in SAP vs Oracle)
- Metric lineage: which systems contribute to which metrics

#### 5. Source Router
Determines which enterprise system(s) should answer the query.

- Rules-based routing (procurement data → SAP MM module, product data → PLM)
- Query type inference (read vs. action vs. report)
- Multi-source fan-out for aggregated metrics
- Returns structured query hints: system, module, filters, join keys

#### 6. Lineage & Explainer
Provides full transparency on every resolution decision.

- Human-readable explanation of each resolution step
- Confidence score with basis
- Alternative interpretations flagged
- Audit trail for governance and debugging

### Technology Choices (Recommended)

| Layer | Technology | Rationale |
|---|---|---|
| API runtime | FastAPI (Python) | Fast iteration, async support, OpenAPI docs free |
| Entity graph | Postgres + pgvector | Start simple; vector similarity for fuzzy matching |
| Relationship graph | PostgreSQL graph queries or lightweight Neo4j | Avoid over-engineering early; migrate if needed |
| Ontology storage | YAML + compiled Python objects | Human-readable, version-controllable, fast at runtime |
| Tenant isolation | Schema-per-tenant or row-level security | Simple multi-tenancy |
| Auth | API key + JWT; OAuth for enterprise | Standard enterprise requirements |
| Connectors | Python adapters per system | Start with SAP (pyrfc), Snowflake, REST |
| Caching | Redis | Cache resolved entity lookups per tenant |
| MCP support | MCP server wrapper | Required for Claude, Copilot, Gemini integration |

---

## 7. API Specification

### Base URL
```
https://api.contextgraph.ai/v1
```

### Authentication
```
Authorization: Bearer {api_key}
X-Tenant-ID: {tenant_id}
```

---

#### `POST /resolve`
Resolve entities, terms, and metrics from natural language.

**Request:**
```json
{
  "text": "How many orders are pending from SK hynix for us?",
  "domain_hint": "supply_chain",
  "tenant_id": "nvidia-prod"
}
```

**Response:**
```json
{
  "resolved_entities": {
    "us": {
      "canonical": "NVIDIA Corporation",
      "entity_id": "ENT-001",
      "confidence": 0.98,
      "basis": "tenant_context"
    },
    "SK hynix": {
      "canonical": "SK Hynix Inc.",
      "entity_id": "SUP-0042",
      "confidence": 0.96,
      "aliases_matched": ["Hynix", "SK Hynix", "SKH"]
    }
  },
  "resolved_metrics": {
    "orders pending": {
      "canonical_metric": "open_purchase_orders",
      "definition": "Purchase orders with status IN ('open', 'partially_received') not yet fulfilled",
      "alternative_interpretations": ["backlog", "unconfirmed_orders"],
      "confidence": 0.91
    }
  },
  "domain": "semiconductor_supply_chain",
  "ambiguities": []
}
```

---

#### `POST /context`
Get full business context for a natural language question.

**Request:**
```json
{
  "question": "Which products are impacted if HBM supply from SK hynix is delayed?",
  "tenant_id": "nvidia-prod"
}
```

**Response:**
```json
{
  "question": "Which products are impacted if HBM supply from SK hynix is delayed?",
  "resolved_context": {
    "supplier": "SK Hynix Inc.",
    "component_class": "High Bandwidth Memory (HBM)",
    "impact_direction": "downstream",
    "tenant": "NVIDIA Corporation",
    "domain": "semiconductor_supply_chain"
  },
  "traversal_path": [
    "SK Hynix → HBM3E components → GPU memory stack → H100/H200/B100 GPU boards → NVIDIA product SKUs"
  ],
  "affected_products": {
    "lookup_required": true,
    "route_to": "source_router"
  },
  "confidence": 0.94,
  "explanation": "Identified SK Hynix as HBM supplier. HBM maps to memory stack in GPU BOM. Downstream impact traversal required to enumerate affected SKUs."
}
```

---

#### `GET /relationships`
Traverse the relationship graph between entities.

**Request:**
```
GET /relationships?from=SK+Hynix&to=product&direction=downstream&tenant_id=nvidia-prod
```

**Response:**
```json
{
  "from_entity": "SK Hynix Inc. (SUP-0042)",
  "to_type": "product",
  "direction": "downstream",
  "paths": [
    {
      "path": ["SK Hynix", "HBM3E", "GPU memory module", "H100 SXM5", "NVIDIA DGX H100"],
      "hops": 4,
      "confidence": 0.92,
      "data_sources": ["PLM-BOM", "SAP-MM"]
    },
    {
      "path": ["SK Hynix", "HBM3E", "GPU memory module", "H200 SXM5", "NVIDIA DGX H200"],
      "hops": 4,
      "confidence": 0.92,
      "data_sources": ["PLM-BOM", "SAP-MM"]
    }
  ],
  "total_paths": 8,
  "truncated": true
}
```

---

#### `POST /metrics`
Look up and disambiguate metric definitions.

**Request:**
```json
{
  "terms": ["backlog", "open PO", "committed supply", "at-risk supply"],
  "domain": "supply_chain",
  "tenant_id": "nvidia-prod"
}
```

**Response:**
```json
{
  "metric_definitions": {
    "backlog": {
      "canonical_metric": "demand_backlog",
      "definition": "Unfulfilled customer orders past their committed ship date",
      "do_not_confuse_with": "open_purchase_orders",
      "system_fields": {
        "SAP": "VBAK.AUGRU + delivery status filter",
        "Snowflake": "orders_fact.status = 'backordered'"
      }
    },
    "open PO": {
      "canonical_metric": "open_purchase_orders",
      "definition": "Supplier POs in status open or partially received",
      "system_fields": {
        "SAP": "EKKO/EKPO with ELIKZ != 'X' and EREKZ != 'X'"
      }
    },
    "committed supply": {
      "canonical_metric": "supplier_committed_supply",
      "definition": "Quantity supplier has confirmed for delivery against open POs"
    },
    "at-risk supply": {
      "canonical_metric": "at_risk_supply",
      "definition": "Open PO quantity where supplier has flagged a delay or confidence < threshold"
    }
  }
}
```

---

#### `POST /route`
Determine which system(s) should answer a query.

**Request:**
```json
{
  "question": "What is the committed supply from SK Hynix for Q3?",
  "resolved_context": { ... },
  "tenant_id": "nvidia-prod"
}
```

**Response:**
```json
{
  "primary_system": {
    "system": "SAP S/4HANA",
    "module": "Materials Management (MM)",
    "query_type": "purchase_order_query",
    "suggested_query": {
      "table_hints": ["EKKO", "EKPO", "EKET"],
      "filters": {
        "supplier_id": "SK_HYNIX_LIFNR",
        "delivery_date_range": "2026-07-01 to 2026-09-30",
        "status": ["B", "C"]
      },
      "aggregate": "SUM(MENGE) as committed_quantity"
    }
  },
  "supplementary_systems": [
    {
      "system": "Snowflake data warehouse",
      "purpose": "Cross-reference with supplier scorecard data",
      "confidence": 0.71
    }
  ],
  "confidence": 0.93
}
```

---

#### `GET /explain`
Get human-readable explanation of how a previous resolution was made.

**Request:**
```
GET /explain?resolution_id=res_abc123&tenant_id=nvidia-prod
```

**Response:**
```json
{
  "resolution_id": "res_abc123",
  "original_question": "How many orders are pending from SK hynix for us?",
  "explanation_steps": [
    "1. 'us' resolved to NVIDIA Corporation using tenant context (tenant default entity).",
    "2. 'SK hynix' matched to SK Hynix Inc. (supplier ID SUP-0042) with 96% confidence via alias table.",
    "3. 'pending orders' interpreted as open_purchase_orders (not demand backlog) because the entity is a supplier, not a customer.",
    "4. Domain classified as semiconductor_supply_chain based on supplier entity type.",
    "5. Routed to SAP MM module as primary system for supplier PO data."
  ],
  "confidence": 0.93,
  "alternative_interpretations": [
    "'pending' could also mean orders in approval workflow — use /resolve with domain_hint='procurement_ops' to try alternate interpretation"
  ]
}
```

---

## 8. Industry Ontology Packs

### Pack 1: Supply Chain Base (v1.0)
Covers the foundational concepts of enterprise supply chain operations.

**Entities covered:**
- Supplier (tier-1, tier-2, tier-3 designations)
- Component / Part / Material
- Bill of Materials (BOM)
- SKU / Finished Good
- Purchase Order, Sales Order, Work Order
- Inventory (on-hand, in-transit, safety stock)
- Warehouse / DC / Plant
- Customer / End Customer

**Relationships:**
- Supplier `SUPPLIES` Component
- Component `USED_IN` BOM
- BOM `BUILDS` SKU
- SKU `SOLD_TO` Customer
- PO `SOURCED_FROM` Supplier
- Order `ALLOCATED_TO` Demand

**Metrics (30+ pre-defined):**
- Open PO qty / value
- Demand backlog
- Inventory turn
- Lead time (quoted, actual, planned)
- On-time delivery rate
- Supplier scorecard metrics
- At-risk supply
- Coverage ratio

### Pack 2: Semiconductor Extension (v1.0)
Extends the base with semiconductor-specific concepts.

**Additional entities:**
- Fab / Foundry (TSMC, Samsung Foundry, Intel Foundry)
- Process node (3nm, 5nm, 7nm, etc.)
- Die / Wafer / Package
- Memory type hierarchy (DRAM → HBM → HBM3E)
- GPU architecture family (Blackwell, Hopper, Ada Lovelace)
- End-of-life / Product qualification status
- Allocation / Committed volume agreements

**Additional relationships:**
- Die `MANUFACTURED_AT` Fab
- Memory `STACKED_ON` Logic die
- Component `QUALIFIES_FOR` Product family
- Product `REQUIRES` Allocation agreement

**Semiconductor-specific metrics:**
- Wafer starts committed
- Die yield (projected vs. actual)
- NRFE/qualification risk
- Allocation coverage
- TAT (turn-around time) by fab
- Long-lead-time component flag

### Pack 3: Customer-Specific Mapping Layer (per tenant)
- Custom entity aliases (company-specific supplier names, internal part numbering)
- Metric overrides (redefine "backlog" to match internal BI definition)
- Additional relationship edges
- Source system configuration (which SAP instance, which Snowflake database)
- Access control rules per entity type

---

## 9. MVP Scope & Roadmap

### MVP Definition (Months 1–4)

**Goal:** Prove the core value proposition with one design partner on one killer workflow: supply chain natural language Q&A with governed, explainable answers.

**Included in MVP:**

| Component | MVP scope |
|---|---|
| Entity Resolver | Supplier name disambiguation, tenant entity resolution, metric term disambiguation |
| Ontology Engine | Supply chain base pack + semiconductor extension |
| Relationship Graph | Supplier → Component → SKU → Product path (read-only) |
| Metric Dictionary | 15 core supply chain metrics |
| Source Router | SAP S/4HANA and Snowflake |
| Explainer | Text explanation + confidence score for every resolution |
| API endpoints | `/resolve`, `/context`, `/route`, `/explain` |
| MCP support | Basic MCP server for Claude and Copilot integration |

**Excluded from MVP:**
- `/relationships` graph traversal API (Phase 2)
- `/metrics` full dictionary API (Phase 2)
- Oracle, Databricks connectors (Phase 2)
- PLM/CRM connectors (Phase 3)
- Self-service ontology editor UI (Phase 3)

**MVP Success Criteria:**
- Design partner agents answer supply chain questions correctly ≥85% of the time (vs. baseline ~40%)
- Entity resolution latency <100ms p95
- Full context API response <200ms p95
- At least 3 "killer question" types working end-to-end

### MVP Killer Workflows

1. **Open order visibility**: "How many open POs do we have with supplier X?" → Resolved supplier, metric, SAP route
2. **Supply risk mapping**: "Which products are impacted if HBM supply is delayed?" → Component → BOM → Product traversal
3. **Customer exposure**: "Which customers are exposed to supplier X's components?" → Bidirectional traversal
4. **Component sourcing**: "Who supplies component Y and what's our committed qty?" → Supplier + PO data routing
5. **Metric disambiguation**: "What's the difference between our backlog and open PO?" → Metric dictionary + definitions

### Roadmap

**Phase 1 — Foundation (Months 1–4)**
- MVP above
- One design partner (semiconductor/supply chain)
- Manual ontology curation

**Phase 2 — Expand (Months 5–9)**
- Full relationship traversal API
- Additional connectors: Oracle, Databricks, PLM
- Customer-facing ontology mapping tool
- Second and third design partners
- Telemetry and confidence improvement loop

**Phase 3 — Scale (Months 10–18)**
- Automotive supply chain ontology pack
- Pharma extension
- Self-service tenant admin UI
- ISV partner SDK
- SOC2 Type II certification
- Additional ERP connectors (Infor, Blue Yonder)

**Phase 4 — Platform (Months 19–24)**
- Ontology marketplace (community-contributed packs)
- Agents-as-a-Service layer on top of ContextGraph
- Real-time event hooks (supply disruption → context update → agent alert)

---

## 10. Business Value & Outcomes

### Value Drivers by Persona

**For AI/ML platform teams:**
- Reduce agent development time: eliminates 60–80% of custom disambiguation and routing code
- Reduce hallucinations: structured context resolution vs. LLM guessing at business terms
- Reusable across all agents: build once, every agent in the enterprise uses the same context layer
- **Metric**: Agent project delivery time reduced from 6 months to 6 weeks

**For data and analytics leaders:**
- Governance at inference time: business definitions enforced before agent acts, not after
- Audit trail: every agent decision traceable to a context resolution step
- No shadow AI: agents use governed metric definitions instead of inventing their own
- **Metric**: Zero "rogue metrics" incidents from agents; full resolution audit log

**For supply chain operations:**
- Self-service supply chain Q&A: analysts spend time on decisions, not data retrieval
- Faster supplier risk analysis: hours to minutes
- Component-to-product traceability in one query
- **Metric**: Analyst time on data retrieval reduced by 70%; mean-time-to-answer for supply risk questions: 4 hours → 15 minutes

**For ISVs:**
- Ship agent features faster: don't rebuild context logic for every customer
- Handle enterprise customer terminology out of the box
- Differentiate on agent quality, not plumbing
- **Metric**: Customer implementation time for agent features: 3 months → 2 weeks

### ROI Quantification (Enterprise Supply Chain Customer)

| Metric | Before ContextGraph | After ContextGraph | Value |
|---|---|---|---|
| Wrong agent answers (supply planning) | ~40% error rate | <10% error rate | Avoids $X0M planning errors |
| Analyst hours on data disambiguation | 15 hrs/week per analyst | 4 hrs/week | 74% reduction |
| Time to identify supplier risk exposure | 4–8 hours | <30 minutes | Speed for critical decisions |
| Agent development cycles (new workflow) | 4–6 months | 4–6 weeks | Faster ROI on AI investment |
| Supplier incident response time | 2–3 days for impact analysis | Same day | Resilience value |

---

## 11. Go-to-Market Strategy

### GTM Motion: Land via AI Platform Teams, Expand via Business Value

```
Land: AI/ML platform team adopts ContextGraph API
  ↓
First agent goes live with correct answers
  ↓
Supply chain ops team sees results → wants more workflows
  ↓
CDO / VP Data formalizes as governed context layer
  ↓
ISVs embed ContextGraph in their products
```

### Target Accounts (MVP Phase)

**Tier 1: Semiconductor OEMs**
- NVIDIA, AMD, Intel, Qualcomm, Marvell, Broadcom
- Profile: Complex HBM/memory supply chains; heavy SAP/Oracle + Snowflake stack; active AI agent investments
- Entry point: AI Platform team building supply chain copilot

**Tier 2: EMS / ODM / Contract Manufacturers**
- Foxconn, Flex, Jabil, Celestica
- Profile: Multi-customer, multi-supplier complexity; NL-to-ERP is extremely high value

**Tier 3: ISV Partners**
- AI companies building supply chain agents (e.g., Aera Technology, o9 Solutions, Kinaxis AI extensions)
- Distribution multiplier: each ISV brings 10–50 enterprise customers

### Distribution Channels

1. **Direct enterprise sales** (Months 1–12): Founder-led, design partner relationships
2. **AI agent ecosystem** (Months 6–18): MCP marketplace listing, LangChain/LlamaIndex plugin, Claude partner directory
3. **ISV/OEM licensing** (Months 12–24): White-label API embedded in ISV products
4. **Systems integrator channel** (Months 18+): Accenture, Deloitte, Infosys supply chain practices

### Positioning by Audience

| Audience | Positioning |
|---|---|
| AI Engineers | "Stop writing disambiguation code. Call one API." |
| Data Leaders | "Governance at inference time, not after the hallucination." |
| Supply Chain Ops | "Ask any supply chain question and get a governed answer in seconds." |
| ISVs | "Your agents work on day one at every customer, not month six." |
| Executives | "Reduce agent failure rate from 40% to 10%. Protect your AI investment." |

---

## 12. Pricing & Packaging

### Pricing Model: API call-based + ontology pack licensing

**Tier 1: Developer / Startup**
- $0/month
- 10,000 API calls/month free
- Public ontology packs only
- Community support
- No SLA

**Tier 2: Team**
- $2,000/month
- 500,000 API calls/month
- Supply chain base ontology pack included
- Email support, 99.5% uptime SLA
- Up to 3 source connectors

**Tier 3: Enterprise**
- $10,000–$50,000/month (based on call volume + connectors)
- Unlimited API calls (fair use)
- All ontology packs (supply chain + semiconductor + future)
- Custom mapping layer
- Dedicated support + CSM
- 99.9% uptime SLA
- SOC2 compliance
- On-prem/VPC deployment option

**Tier 4: ISV/OEM**
- Custom licensing
- White-label API
- Revenue share or per-seat model
- Co-marketing and partner program

### Unit Economics Target

- Average enterprise ACV: $200,000–$600,000
- Average ISV ACV: $100,000–$300,000 + per-seat royalty
- Gross margin target: 75%+ (SaaS API model)
- Payback period target: <12 months

---

## 13. Risks & Mitigations

### Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Ontology accuracy issues in production | Medium | High | Curate with domain experts; confidence scoring flags low-confidence responses; human-in-the-loop for low-confidence resolutions |
| Latency under load (complex graph traversal) | Medium | High | Redis caching for common entity lookups; pre-compute relationship paths; async for deep traversal |
| ERP connector brittleness (SAP version changes) | High | Medium | Abstract connector layer; version-specific adapters; design partner SLA includes connector support |
| Tenant data isolation security breach | Low | Critical | Schema isolation; penetration testing; SOC2 from day one |

### Market Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| SAP/Oracle build similar capability natively | High (2–3 yr) | High | Multi-ERP value proposition is structurally neutral; be the "Switzerland" of context APIs; first-mover customer lock-in |
| Semantic layer vendors (dbt, Cube) expand scope | Medium | Medium | Their core motion is SQL/BI; agent context is orthogonal; partnership > competition |
| OSI standard makes ontologies a commodity | Low-medium | Medium | Standard covers metric metadata YAML, not entity resolution, routing, or industry-specific relationships |
| Customers build in-house | Medium | Medium | Switching cost moat (customer context graph); ongoing ontology maintenance burden |

### Go-to-Market Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Long enterprise sales cycles | High | Medium | Start with AI/ML platform teams (faster buyer); pilot-first pricing |
| "Build it ourselves" objection | High | Medium | Quantify cost of building: 2–4 engineers × 12 months + ongoing maintenance |
| Design partner demands custom features | Medium | Medium | Design partner agreement includes scope; cap customization at 30% of engineering |

---

## Appendix A: Example Full API Response

```json
{
  "question": "How many orders are pending from SK hynix for us?",
  "resolution_id": "res_abc123",
  "resolved_context": {
    "us": "NVIDIA Corporation",
    "supplier": "SK Hynix Inc.",
    "domain": "semiconductor_supply_chain",
    "metric": "open_purchase_orders",
    "metric_definition": "Purchase orders with delivery_status IN ('open', 'partially_received') not yet goods-receipted"
  },
  "relationships": [
    "SK Hynix Inc. supplies HBM3E memory components to NVIDIA Corporation",
    "HBM3E components map to internal GPU board assembly SKUs",
    "Open purchase orders for suppliers are stored in SAP Materials Management"
  ],
  "recommended_action": {
    "system": "SAP S/4HANA",
    "module": "Materials Management (MM)",
    "query_type": "open_purchase_orders",
    "filters": {
      "supplier_lifnr": "SK_HYNIX_0042",
      "buyer_entity": "NVIDIA_CORP_1000",
      "delivery_status": ["A", "B"],
      "deletion_flag": false
    },
    "tables": ["EKKO", "EKPO", "EKET"],
    "aggregate": "SUM(EKPO.MENGE - EKPO.WEMNG) as open_qty, SUM((EKPO.MENGE - EKPO.WEMNG) * EKPO.NETPR) as open_value"
  },
  "confidence": 0.93,
  "explainability": {
    "entity_resolutions": [
      "'us' → NVIDIA Corporation: resolved from tenant default entity (confidence: 0.98)",
      "'SK hynix' → SK Hynix Inc. (SUP-0042): matched via supplier alias table (confidence: 0.96)"
    ],
    "metric_resolution": "'pending orders' interpreted as open_purchase_orders (not demand backlog) because subject is a supplier entity, not a customer entity (confidence: 0.91)",
    "routing_decision": "SAP MM selected as primary system: NVIDIA's procurement data resides in SAP S/4HANA instance PRD-100 per system registry"
  },
  "alternative_interpretations": [
    "If 'pending' means 'awaiting supplier acknowledgment', use metric: unconfirmed_pos — call /metrics?terms=pending+orders&context=supplier_acknowledgment"
  ]
}
```

---

## Appendix B: Ontology Pack — Metric Dictionary (Excerpt)

| Metric Name | Canonical ID | Definition | Do Not Confuse With |
|---|---|---|---|
| Open Purchase Orders | `open_po` | POs issued to supplier, not yet fully goods-receipted | Demand backlog |
| Demand Backlog | `demand_backlog` | Customer orders past committed ship date, unfulfilled | Open PO |
| Committed Supply | `committed_supply` | Supplier-confirmed qty against open POs | At-risk supply |
| At-Risk Supply | `at_risk_supply` | Open PO qty with supplier delay flag or confidence < 80% | Committed supply |
| Allocated Demand | `allocated_demand` | Customer orders with confirmed supply allocation | Unallocated demand |
| Safety Stock | `safety_stock` | Minimum inventory buffer per planning policy | Cycle stock |
| Lead Time (Quoted) | `lead_time_quoted` | Supplier-stated lead time at PO creation | Lead time (actual) |
| Lead Time (Actual) | `lead_time_actual` | Actual days from PO to goods receipt, trailing 12 months | Lead time (quoted) |
| Coverage Ratio | `coverage_ratio` | Committed supply / open demand for a given horizon | Fill rate |

---

*Document owner: Founding team | Distribution: Internal + authorized investors | Version control: Git*
