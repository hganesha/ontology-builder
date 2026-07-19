# Lattice — An AI Ontology Builder

*Vision, use cases, and product specification*

> Working name. The product takes a description of a business, researches its
> industry, drafts a domain ontology, validates it with the user, renders it,
> and keeps it alive as the semantic layer that grounds enterprise AI.

The uploaded **Meridian-SC** ontology is the reference target for what Lattice
produces. Note three things baked into that document: every node can be
`validated`, `inferred`, or `declared`; every node and edge carries a
`confidence_score`; and §5 is a stack of reusable industry templates. The
format already assumes a machine drafted it and a human signed off. Lattice is
the system that does the drafting and runs the sign-off.

---

## 1. The thesis: an ontology is the context layer for enterprise AI

For thirty years, building an ontology was a specialist craft — knowledge
engineers in Protégé, OWL/RDF, months of elicitation, brittle results that
rarely survived contact with real data. The artifact was beautiful and the
process was unaffordable, so most enterprises never built one. They shipped
fragmented schemas instead: SAP calls it a *Vendor*, the data warehouse calls
it a *Counterparty*, procurement calls it a *Supplier*, and nobody can join
them without a six-week project.

Two things changed that make this the right moment.

**LLMs collapse the cost of the expensive part.** Entity discovery,
relationship inference, property elicitation, and the grinding work of reading
an industry's regulations and standards — the things that used to take a team a
quarter — a model can now draft in minutes. The bottleneck moves from
*authoring* to *validating*, which is exactly where human expertise should sit.

**Enterprise AI is starving for exactly this artifact.** RAG pipelines
hallucinate because retrieval is ungrounded. Text-to-SQL guesses joins because
it has no model of what a "supplier" is. Agents can't reason about second-order
exposure because they have no graph to traverse. Every one of these failures is
a missing semantic layer. The ontology *is* that layer — the typed, governed
model of the business that AI reasons over instead of guessing at.

So Lattice isn't an ontology tool with AI features. It's a way to make an
enterprise **legible to AI**, where the ontology is the durable context asset
and AI is both the thing that builds it and the primary thing that consumes it.

A useful mental model: Lattice is a **compiler**. Messy business reality —
documents, data, tribal expertise — is the source. The ontology is the
intermediate representation. And it compiles down to many backends: a knowledge
graph, a semantic layer for SQL, agent context, a compliance engine. The "AI"
is the front-end parser and the optimizer; the human is the type-checker.

---

## 2. What the app does — the loop

Lattice runs a six-stage loop. The first pass takes a user from a blank page to
a rendered, validated ontology in a single session. Every later pass is the
same loop in maintenance mode.

```
   DESCRIBE  ──►  DRAFT  ──►  VALIDATE  ──►  RENDER  ──►  OPERATIONALIZE
      ▲                          │                              │
      │                          ▼                              ▼
      └──────────────────────  EVOLVE  ◄──────────────  (usage + new data)
```

**1. Describe.** A conversational intake learns the business: what it makes or
sells, where it operates, its tier structure, and — critically — *what
questions it needs the ontology to answer* (risk exposure? compliance? cost?
provenance?). The intended use shapes which entities and properties matter. In
parallel, the user connects or uploads what they have: ERP exports, a data
warehouse schema, supplier lists, BOMs, policy PDFs, a spreadsheet of plants.

**2. Draft.** Three engines (§3) merge an industry template, live research, and
reasoning over the user's own data into a *candidate* ontology — entity types,
relationship types, property schemas, constraints. Every element is tagged with
provenance and a confidence score. Nothing is presented as fact yet.

**3. Validate.** This is the product. The AI walks the domain expert through the
draft — highest-leverage, lowest-confidence items first — with evidence
attached to each claim and one-tap accept / edit / reject / merge. Consistency
linting and data-grounding checks run continuously. The output of this stage is
an ontology where `validation_status` has been driven from `inferred` toward
`validated` wherever it matters.

**4. Render.** The validated model becomes an interactive graph explorer, an
entity catalog / data dictionary, and a set of exports (the Meridian-SC
markdown is one of them; OWL, JSON-LD, GraphQL, LinkML, dbt, Neo4j are others).

**5. Operationalize.** The ontology is mapped to physical data sources and
published as a live context layer that downstream systems consume — a knowledge
graph, an agent's world model, a text-to-SQL semantic layer, a compliance
engine. This is where the artifact starts paying rent.

**6. Evolve.** Usage and new data feed back. A new regulation drops; a source
system adds a table; agents keep asking a question the ontology can't answer.
Lattice surfaces these as a reviewable changeset — a diff, not a rebuild — and
the loop runs again.

---

## 3. How it builds — three engines and a provenance ledger

The draft stage layers three sources of knowledge, each with a different trust
profile. Keeping them distinct is what makes validation tractable: the user can
trust the engines differently and review accordingly.

**Templates — the industry starter pack.** §5 of your Meridian-SC doc *is* a
template library: per-industry entity subtypes, characteristic relationships,
key materials, and the standards/regulations that apply. Lattice ships these as
versioned, composable templates (Electronics, Automotive/EV, Aerospace &
Defense, Medical Devices, Apparel, Food & Ag, Energy, Mining — and the same
pattern generalizes far beyond supply chain). A template gives the draft a
correct spine in seconds and anchors it in domain reality instead of generic
guesses.

**Research — live specialization.** A research agent reads the current
regulatory and standards landscape for the business's specific products,
geographies, and obligations, and specializes the template. This is how the
cross-industry overlay in §6 (UFLPA, CSRD, CSDDD, EUDR, conflict-minerals rules)
gets attached to the *right* entities, and how the model stays current as
regulations change. Research is also the engine most in need of human
checking — so its outputs are flagged accordingly.

**Intelligence — reasoning over the actual business.** The template says a
semiconductor supply chain *usually* has OSAT facilities; the intelligence
engine looks at *your* data and BOMs and proposes the entities, relationships,
and properties that are actually present — including the ones no template would
predict. This is where `single_source`, `revenue_exposure_usd`, or a bespoke
relationship gets surfaced because the user said exposure was their top
question and the data supports it.

**The provenance ledger.** Every entity, relationship, and property records
where it came from and how confident the system is:

| Provenance | Maps to | Default trust | Review priority |
|---|---|---|---|
| `declared` | The user stated it directly | High | Low |
| `from_template` | Industry starter pack | Medium-high | Low |
| `from_research` | Regulatory / standards research | Medium | **High** |
| `from_data` | Inferred from connected sources | Medium | Medium |
| `inferred` | Model reasoning, no direct evidence | Low | **High** |

This ledger is not decoration — it *is* the validation queue. It maps directly
onto the `validation_status` and `confidence_score` fields already in your
schema, and it lets Lattice triage: auto-accept the high-confidence spine,
escalate the low-confidence inferences, and never make a domain expert review
something they already declared.

---

## 4. Use cases

I've grouped these into the two lenses you called out — *mapping to enterprise
data* and *AI intelligence / context* — plus the domain applications the
ontology unlocks and a note on how far this generalizes past supply chain.

### 4.1 Mapping to enterprise data

This is where the ontology earns its keep first, because it solves a problem
every large enterprise already has: a fragmented data estate with no shared
vocabulary.

**Semantic layer over a fragmented estate.** The ontology is the canonical
model; mapping resolves every system's local vocabulary to it. *Vendor*
(Oracle), *Counterparty* (treasury), *BP* (SAP), *Account* (Salesforce) all map
to one `Supplier` entity type. Queries, metrics, and AI agents speak the
ontology's language and stop caring which system the data lives in.

**AI-bootstrapped master data and entity resolution.** Lattice ingests a
warehouse schema and *proposes* the mapping — column → property, table → entity
type — using names, types, and sampled values. It then proposes entity
resolution: the same real-world supplier appearing under three IDs in three
systems, collapsed to one node. Both are presented as reviewable suggestions,
not silent merges.

**Data contracts and schema-aware exports.** Because properties carry types
(your §3 schema: `hs_code` string, `risk_score` 0–100 float, `validation_status`
enum), the ontology can emit data contracts and target schemas — a GraphQL API,
a dbt semantic model, a JSON Schema — that downstream teams build against. The
ontology becomes the source of truth the warehouse conforms *to*.

**A data catalog people actually read.** The rendered entity catalog is a living
data dictionary grounded in the business, not a stale wiki. "What is a
`Facility`, what properties does it have, which source systems populate them,
and what's the data quality?" is answerable in one place.

**Coverage and quality diagnostics.** Once mapped, Lattice can flag the
revealing gaps: properties the ontology defines that *no* source system
populates (you're tracking `esg_score` in the model but nothing feeds it),
entity types that appear in templates but never in your data, relationships with
suspiciously low fill rates. The ontology becomes a lens on the data estate's
blind spots.

### 4.2 AI intelligence & context

This is the larger, newer prize: the ontology as the structured context that
makes enterprise AI reliable.

**Grounding agents with a world model.** An agent answering *"what's my exposure
if a Taiwan earthquake takes out advanced packaging?"* needs to traverse
`Product → CONTAINS → Component → MADE_OF → Material → DERIVED_FROM → Mineral`
and `Facility(country=TW)`. Dump raw tables into a context window and it
guesses; give it the typed ontology graph and the traversal is meaningful and
auditable. The ontology is the agent's world model.

**Retrieval that respects structure.** RAG scoped by entity type and
relationship traversal beats blind vector similarity. "Find the audit findings
for tier-2 cobalt smelters in the DRC" becomes a typed graph query plus
retrieval, not a fuzzy nearest-neighbor lookup that returns plausible-looking
garbage.

**Text-to-SQL and natural-language analytics that don't guess joins.** The
ontology tells the model that `Supplier OPERATES Facility` is a 1:N join on a
specific key. Cardinalities (your §7 summary), relationship directions, and
property types turn "show me single-source suppliers in Europe by spend" into
correct SQL instead of a confident hallucination.

**Guardrails and hallucination reduction.** The ontology is a constraint system.
An agent can't assert a relationship type that doesn't exist or a property a
node can't have. The model of what's *valid* bounds what the AI is allowed to
claim — a schema-level safety rail under the probabilistic layer.

**Reusable, governed context engineering.** Instead of every team hand-crafting
prompts that smuggle in business definitions, the ontology is the shared,
versioned context block injected into agents. Change the definition of
"critical mineral" once, and every downstream agent inherits it.

**Simulation and counterfactual reasoning.** Because the graph encodes
dependencies, an agent can run the scenarios your §8 feature map already
implies — disruption propagation, what-if a supplier fails, second-order
exposure — by traversing and perturbing the graph rather than reasoning from
nothing.

### 4.3 Domain applications the ontology unlocks

These are the Meridian-SC-style outcomes the rendered ontology makes possible —
the reason a business invests in building it at all:

- **Multi-tier risk scoring.** Composite `risk_score` propagated along
  `SUPPLIES_TO` edges to surface concentration and single-source exposure deep
  in the network.
- **Compliance mapping.** `Policy GOVERNS` / `REQUIRES Certificate` chains turn
  UFLPA, CSRD, EUDR, and conflict-minerals obligations into traceable
  requirements on specific entities, with evidence and gaps.
- **Disruption response.** When a facility goes down, traverse the graph to the
  impacted products, customers, and revenue in seconds.
- **Provenance and traceability.** `DERIVED_FROM` lineage from finished product
  back to mine or farm — the spine of EUDR geolocation and conflict-minerals
  due diligence.

### 4.4 How far this generalizes

Supply chain is the worked example, but the meta-model (§5.1) is
domain-agnostic. The same loop builds:

- **Financial services** — counterparties, instruments, exposures, regulatory
  obligations (Basel, MiFID), KYC graphs.
- **Healthcare / life sciences** — patients, encounters, providers, claims,
  drugs, trials, adverse-event lineage.
- **Manufacturing / PLM** — products, BOMs, plants, equipment, maintenance.
- **Customer / commercial** — accounts, contacts, products, contracts,
  entitlements (the CRM/CDP semantic layer).
- **Internal knowledge** — teams, systems, services, ownership — a "company
  graph" for internal agents.

Anywhere there's a fragmented data estate and an ambition to put AI on top of
it, the ontology is the missing piece — and Lattice is the same product.

---

## 5. Product specification

### 5.1 The meta-model (the ontology of ontologies)

Lattice needs its own internal model of what an ontology *is*. Reverse-engineered
from your Meridian-SC document, the primitives are:

| Primitive | Definition | Evidence in Meridian-SC |
|---|---|---|
| **EntityType** | A node class, optionally extending a parent | Root types → 18 base types → industry subtypes |
| **RelationshipType** | A directed, typed edge with `from`/`to` and cardinality | §2 table; §7 cardinality summary |
| **Property** | A typed attribute on an entity or relationship | §3 schemas; `type` + `notes` columns |
| **Constraint** | Cardinality, enum domains, required inverses, validity rules | §7; enum fields throughout |
| **Provenance** | Source + confidence + validation status | `validation_status`, `confidence_score`, `inferred` |
| **Mapping** | Binding from a property to a physical data source | implied by "mock data generation, graph node typing" |
| **Template** | A reusable, composable industry extension | all of §5 |
| **Overlay** | A cross-cutting layer applied across templates | §6 regulatory overlay |

Everything the user creates is an instance of this meta-model, which is what
makes generic tooling — validation, linting, export, diffing — possible across
any domain.

### 5.2 Personas

- **The domain expert** (procurement lead, compliance officer, supply-chain
  risk analyst). Owns truth. Lives in the validation flow. Rarely technical.
- **The data engineer / architect.** Owns the mapping to physical sources and
  the exports. Cares about cardinality, keys, and contracts.
- **The AI / platform team.** Consumes the ontology as context for agents and
  RAG. Cares about the API, versioning, and grounding quality.
- **The executive sponsor.** Wants the outcomes in §4.3 and a governance story.

### 5.3 Core flows and screens

**Intake (conversational + connectors).** A guided chat establishes industry,
products, geography, tier depth, and — emphasized — the *questions the ontology
must answer*. A connectors panel ingests warehouse schemas, ERP exports,
spreadsheets, and document sets. Intended use is a first-class input, because it
decides which properties are worth modeling.

**Draft review (the candidate map).** The first render of the candidate
ontology, visually distinguishing `validated` / `inferred` / `declared` nodes
(color + badge, mirroring your node semantics in §4.1). Confidence shown per
element. The user can explore before committing to a single decision.

**Validation workspace (the core screen).** A triaged queue, not a wall of
forms. For each proposed element:
- the claim ("`Facility` should have property `capacity_unit`"),
- its provenance and confidence,
- an **evidence drawer** — the regulation text, the data sample, the template
  source, the doc snippet it came from,
- actions: accept / edit / reject / merge / "explain why this is here",
- a **grounding test** where applicable ("3 rows in your `plants` table match
  this entity type — confirm the mapping?").

Running alongside: a **consistency linter** flagging missing inverse
relationships, cardinality contradictions, orphan entity types, enum domains
that don't match sampled data, and properties no source can populate.

**Graph explorer & entity catalog.** Interactive, filterable by entity type,
tier, provenance, and risk. The catalog is the human-readable data dictionary
view of the same model.

**Mapping workspace.** Side-by-side ontology properties and physical columns,
with AI-proposed bindings the data engineer confirms or overrides, plus the
coverage diagnostics from §4.1.

**Export & publish.** One model, many targets (§5.5). Publishing makes the
ontology available as a live, versioned context endpoint.

**Evolution / changeset review.** When research finds a new regulation or a
source adds a table, the change arrives as a reviewable diff against the current
version — additions, deprecations, property changes — each with provenance,
re-running the validation flow only on what changed.

### 5.4 Validation & trust mechanics (the differentiator)

Generating a plausible ontology is now cheap; the defensible value is making
validation fast and trustworthy. The mechanics:

- **Confidence-triaged review.** Auto-accept the high-confidence spine; escalate
  low-confidence inferences. Never ask the expert to review what they declared.
- **Evidence-on-everything.** No claim without a source the user can inspect.
  This is what converts `inferred` to `validated` honestly.
- **Grounding against real data.** Wherever a proposed type or property can be
  checked against connected sources, check it and show the match rate.
- **Continuous linting.** Catch structural defects (your §7 cardinality rules
  are machine-checkable) before they reach production.
- **Reversible, versioned decisions.** Every accept/reject is logged and
  diffable; the ontology has git-like history and stewardship per domain.

### 5.5 Export & integration surface

| Target | Form | Consumer |
|---|---|---|
| Markdown spec | The Meridian-SC format | Humans, docs, governance |
| OWL / RDF / SHACL | Formal semantic web + constraints | Reasoners, semantic tooling |
| JSON-LD | Linked-data instances | Interop, web context |
| LinkML | Schema-as-data definition | Data modeling, codegen |
| GraphQL schema | Typed API | Application teams |
| Property graph (Neo4j / cypher) | Labeled graph | Knowledge graph, traversal |
| dbt / semantic layer | Metrics + join model | Analytics, text-to-SQL |
| JSON Schema / data contracts | Validation schema | Pipelines, ingestion |
| Live context API | Versioned, queryable | Agents, RAG, internal AI |

The same validated model compiles to all of them; the user never re-authors per
backend.

### 5.6 System architecture

```
┌──────────────────────────────────────────────────────────────┐
│  INTAKE         conversational agent + connector ingestion     │
├──────────────────────────────────────────────────────────────┤
│  DRAFT ENGINES  template library · research agent · inference  │
│                 over data  ──►  provenance ledger              │
├──────────────────────────────────────────────────────────────┤
│  META-MODEL     EntityType · RelationshipType · Property ·     │
│  STORE          Constraint · Provenance · Mapping (versioned)  │
├──────────────────────────────────────────────────────────────┤
│  VALIDATION     triage queue · evidence drawer · linter ·      │
│                 grounding tests                                │
├──────────────────────────────────────────────────────────────┤
│  RENDER         graph explorer · entity catalog · diff viewer  │
├──────────────────────────────────────────────────────────────┤
│  COMPILE        export adapters (OWL · GraphQL · dbt · Neo4j…) │
├──────────────────────────────────────────────────────────────┤
│  PUBLISH        live context API · mapping to physical sources │
└──────────────────────────────────────────────────────────────┘
```

**Implementation notes.** LLM orchestration drives the intake, draft, and
research agents. The meta-model store is versioned (git-like history; the
ontology is treated as code). Embeddings power data-to-ontology mapping and
entity resolution. Connectors reach ERP, warehouses, and document stores.
Export adapters target the formats above. The validation linter is mostly
deterministic — cardinality and constraint checks don't need a model. Don't
reinvent the formalism: lean on **LinkML / OWL / SHACL** for the schema and
constraint layer so the output interoperates with existing semantic tooling.

### 5.7 Honest hard parts

- **Validation fatigue.** A 200-entity ontology has thousands of properties and
  edges. If review feels like data entry, the product dies. Triage, batching,
  and "trust this whole template section" controls are existential, not nice-to-
  have.
- **Keeping it alive.** Ontologies rot. The evolution loop and changeset review
  are the antidote, but they have to be low-friction or the model drifts from
  reality within a quarter.
- **Ontology bloat.** AI will happily propose every conceivable entity. The
  *intended-use* signal from intake must aggressively prune — model only what
  answers the user's questions. A smaller correct ontology beats a sprawling
  speculative one.
- **Entity resolution is genuinely hard.** Proposing merges is easy; being right
  about them is not. Keep humans in the loop and never merge silently.
- **Research can be wrong or stale.** Regulatory research must cite sources and
  be re-runnable, and its outputs must stay flagged as the highest-review-
  priority tier.
- **Prior art is real.** Palantir's Foundry/AIP ontology proved the
  ontology-as-operational-layer thesis at the high end; Protégé, TopBraid, and
  Stardog own the formal-semantics world. Lattice's wedge is the combination
  nobody packages together: **AI-bootstrapped authoring + industry templates +
  self-serve validation + compile-to-anything operationalization**, aimed at
  teams who would never staff a knowledge-engineering practice.

### 5.8 Phasing

- **MVP** — intake → template + inference draft → validation workspace → graph
  render → markdown + JSON-LD export. One or two industry templates (start where
  your sample is strongest: electronics/semi). Prove that a domain expert can go
  from blank page to a trustworthy, rendered ontology in one session.
- **V2** — connectors and data-grounded validation, the mapping workspace,
  entity resolution, coverage diagnostics, more templates and the regulatory
  overlay.
- **V3** — the live context API and agent/RAG grounding, the evolution loop and
  changeset review, governance and stewardship, the full export matrix and
  domain applications (risk, compliance, disruption).

---

## 6. The one-sentence version

Lattice turns a description of a business into a validated, living ontology —
using industry templates, live research, and reasoning over the company's own
data — so that an enterprise's reality becomes the typed, governed context layer
its AI can finally reason over instead of guess at.
