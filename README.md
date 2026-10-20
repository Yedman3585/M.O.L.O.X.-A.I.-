# M.O.L.O.X. — Technical Architecture Regulation


## 1. Purpose and Scope

This document defines the mandatory architectural rules of M.O.L.O.X.: every team member checks their code and PR against it before merge.

- **Audience:** developers, reviewers, and those preparing the demo and pitch.
- **Mandatory:** sections 2, 3, 5, 10, and 14. Violation blocks merge.
- **Recommended:** the remaining sections. Deviation from them must be documented as an ADR (section 12).
- **How to change the regulation:** only via a PR to `docs/adr/` with approval from the technical lead and one additional reviewer.

## 2. Invariant Principles

The system is a controlled local loop, not a swarm of chatbots. Seven rules are not up for debate within a task or PR.

1. **Localness.** All services run in a single isolated network. Outbound internet is allowed only for the `collectors` module via an allowlist (section 11); all others are forbidden.
2. **Provenance and audit by default.** Every fact stores source, time, confidence, method, and author (service or operator). A fact without these fields is not recorded.
3. **Modular monolith.** One deployment, clear boundaries. Modules communicate only through contracts (section 5) and do not read each other's tables.
4. **Human in the loop.** The system proposes, the operator confirms. Agents do not perform actions with external effects.
5. **LLM is not a source of truth.** The model parses, links, summarizes, and formulates hypotheses. Every statement in a report references a `Claim` with a source.
6. **End-to-end scenario first.** One working path from source to report is more important than twenty services.
7. **Data about infrastructure, not people.** The boundaries are described in section 3.

## 3. System Boundaries

M.O.L.O.X. analyzes network and infrastructure conditions, not individual people. Anything outside the “In scope” table is not added to the project — not in code, not in data, not in the demo.

| In scope | Out of scope |
| --- | --- |
| IP enrichment from our own logs and threat indicators: country, city, ASN, owning organization, signs of proxy, VPN, and hosting (WhereGo, IP2X, MaxMind MMDB) | Determining the location of a specific person, including by linking IP with a phone, nickname, or email |
| Infrastructure and event layers: aviation, satellites, earthquakes, ports, cables, CVE, sanctions lists (Akashic and Osiris adapters) | The phone part of IP-Tracker and the username/email recon mode from Akashic: disabled in adapters |
| Graph of relationships between IPs, ASNs, organizations, objects, and events | `Person` and `Device` entities with personal attributes, profiles on people |
| Reports with sources and confidence | Use of public cameras to identify people |

**Hard rules, verified by tests:**

- `geo-enrichment` accepts only an IP address as input. Phone, email, and nickname fields are absent from the request schema; extra fields are rejected.
- The response always includes `accuracy_radius_m`. The UI and reports do not show coordinates more precisely than the source provides.
- IPs of private individuals are masked by default in the UI and reports to /24 (IPv4) or /48 (IPv6). The full address is visible only to a role with explicit permission and is written to audit.

## 4. Modules and Responsibility Zones

Data flow is one-way: `collectors` → queue → `normalizer` → `knowledge` → `agents` → `llm-gateway` → `api` → `ui`. Each step writes an event to `audit`.

| Module | Responsible for | Writes | Must not |
| --- | --- | --- | --- |
| `collectors` | Data ingestion from sources: Akashic and Osiris adapters, offline feeds, custom scripts | Raw data to MinIO, messages to queue | Write to the knowledge DB, call the LLM |
| `normalizer` | Ontology normalization, deduplication, time zones | Observations to Postgres | Access the internet, call the LLM |
| `geo-enrichment` | IP → country, city, ASN, organization, proxy flag (WhereGo, IP2X, offline MMDB) | Nothing: returns `GeoResult` | Accept data other than IP; store state |
| `knowledge` | Graph (Neo4j), vector index (Qdrant), fact versions | Neo4j, Qdrant, claims to Postgres | Call collectors |
| `agents` | Role orchestration as a state machine (LangGraph) | Draft claims and reports via the `knowledge` API | Write directly to storages |
| `llm-gateway` | Single entry point to Ollama, structured output, guardrails | Only audit records | Access the internet, accept unvalidated raw data |
| `api` | FastAPI, authentication and roles, single entry point | Only audit | Contain business logic |
| `ui` | Map, graph, timeline, chat, audit log | Nothing | Access the DB directly |
| `audit` | Immutable log | Append-only table in Postgres | Edit or delete records |

## 5. Contracts Between Modules

Modules exchange only five structures. Schemas are in `contracts/` as JSON Schema and are validated at the input and output of each module.

```json
// RawRecord: collectors → queue
{ "record_id": "uuid", "source_id": "string", "collected_at": "RFC3339",
  "content_ref": "s3://raw/...", "content_hash": "sha256",
  "adapter_version": "semver", "license": "string" }

// Observation: normalizer → Postgres
{ "obs_id": "uuid", "entity_type": "IP|ASN|Organization|Asset|Event|Location",
  "entity_key": "string", "attrs": {}, "observed_at": "RFC3339",
  "source_id": "string", "raw_ref": "record_id" }

// GeoResult: geo-enrichment → normalizer / agents
{ "ip": "string", "country": "ISO-3166", "city": "string|null", "asn": 0,
  "org": "string|null", "lat": 0.0, "lon": 0.0, "accuracy_radius_m": 0,
  "is_proxy": false, "source_db": "string", "db_version": "string",
  "timestamp": "RFC3339" }

// Claim: knowledge (unit of knowledge)
{ "claim_id": "uuid", "subject": "entity_key", "predicate": "string",
  "object": "entity_key|value", "confidence": 0.0,
  "method": "rule|geo-db|llm|operator", "source_ids": ["..."],
  "created_by": "service|operator_id", "created_at": "RFC3339",
  "version": 1, "supersedes": "claim_id|null" }

// AuditEvent: all modules → audit
{ "event_id": "uuid", "ts": "RFC3339", "actor": "service|operator_id",
  "action": "string", "target": "string", "inputs_ref": ["..."],
  "outcome": "ok|denied|error", "trace_id": "uuid" }
```

**Rules:**

- Schemas are versioned by semver. Breaking changes are possible only via ADR.
- Unknown fields are rejected, not ignored.
- The `confidence` field is in the range 0–1. For `method = llm`, it cannot exceed 0.7 until the fact is confirmed by `VerifierAgent` or an operator.

## 6. Ontology and Knowledge Graph

The graph answers the question “which objects are connected, on what basis, and how confident we are.” The entity set is fixed; a new type is added only via ADR.

| Entity | Key | Main attributes |
| --- | --- | --- |
| `IP` | Address | Version, mask for UI, first and last seen |
| `ASN` | Number | Name, registration country |
| `Organization` | Normalized name | Type, country, link to sanctions list |
| `Asset` | Identifier | Type (port, cable, infrastructure object), coordinates, source |
| `Event` | Identifier | Type, time, geometry, severity |
| `Location` | Geometry (PostGIS) | Precision level, `accuracy_radius_m` |
| `Source` | Identifier | Adapter, version, license, export date |
| `Claim` | `claim_id` | See contract in section 5 |
| `Report` | Identifier | Author, version, list of used `Claim`s |

**Edges** always carry `confidence`, `first_seen`, `last_seen`, `source_id`, and `method`. An edge without these fields is not created.

**Versioning:** any change creates a new version; the old one remains (`supersedes`). Data deletion is possible only under the retention policy (section 7) and is written to audit.

**What the ontology does not contain:** `Person`, `Device`, phones, emails, and nicknames. Operator accounts live only in the authentication module and do not enter the graph.

## 7. Storages and Data Ownership

The rule is simple: each record has one owner, and only the owner writes. Others read via the owner's API.

| Storage | Owner (writes) | What it stores | Reads |
| --- | --- | --- | --- |
| PostgreSQL + PostGIS | `normalizer` (observations), `knowledge` (claims), `audit` (log) | Structured entities, events, claims, policies, audit | `knowledge`, `api` |
| Neo4j | `knowledge` | Graph of relationships and edge versions | `knowledge` |
| Qdrant | `knowledge` | Embeddings for RAG | `knowledge` |
| MinIO | `collectors` | Raw data, immutable after write, object name = sha256 | `normalizer` |
| Redis / NATS | Infrastructure | Queues, cache, sessions. Does not store truth | All modules |

**Forbidden:** storing everything in one Postgres “just in case,” storing data in Redis that cannot be lost, and modifying raw data in MinIO.

**Retention and deletion:** retention periods for raw data and IP addresses are set by policy in `config/retention.yaml`. Deletion under policy is written to audit.

**Open question:** what retention periods do we set for raw data and logs with IPs (proposal: 30 days for raw data with IPs, 1 year for aggregates). The technical lead decides before the MVP starts.

## 8. Agents and State Machine

Agents do not “talk” to each other. Each is a LangGraph node with its own input, output, and list of allowed tools. Transitions between nodes are defined by code, not by the model.

| Agent | Input | Output | Allowed tools |
| --- | --- | --- | --- |
| `CollectorAgent` | Schedule, source list | Launch adapters, status | `collectors` API |
| `NormalizerAgent` | New `RawRecord` | Observations | `normalizer` API |
| `EntityResolutionAgent` | Observations | Duplicate merge proposals | Read `knowledge` |
| `EnrichmentAgent` | Valid IPs from observations | `GeoResult` as claims | `geo-enrichment` API (IP only) |
| `LinkerAgent` | Claims, observations | Graph edge candidates | Cypher read-only, `knowledge` API |
| `VerifierAgent` | Claim candidates | Confirmation from independent sources | Read `knowledge` |
| `RedTeamAgent` | Hypotheses | Counterarguments, contradictions | Read `knowledge` |
| `PolicyAgent` | Any request and action | Permission or denial with reason (veto power) | Policies from `config/policies.yaml` |
| `ReportAgent` | Confirmed claims | Draft report with references | Read `knowledge`, `llm-gateway` |

**Execution rules:**

- State and checkpoints are stored in Postgres; each step is idempotent (key = `trace_id` + step).
- Each node has timeouts and retry limits. The entire run has a step limit.
- Collection, inference, and decision-making are never combined in one agent.
- Operator confirmation points: entity merge, report publication, any action outside the system.
- `PolicyAgent` is called before every tool call. Its denial cannot be bypassed.

## 9. Rules for Working with LLM

The model is a tool for parsing and formulation, not a source of facts.

1. **Single entry.** Only `llm-gateway` calls Ollama. Direct calls from other modules are forbidden.
2. **Structured output.** Each response is validated against JSON Schema. An invalid response is retried no more than twice, then the step fails.
3. **Tools by whitelist.** For each agent role, allowed tools are specified. SQL and Cypher are read-only, via parameterized templates.
4. **Raw data from external sources is considered untrusted.** Text from feeds, posts, and documents goes to the model as data in a separate block, not as instructions. Prompt injection attempts (commands inside data) are logged to audit and discarded.
5. **No self-management of facts.** Model assertions enter the graph only as `Claim` with `method = llm` and a confidence cap of 0.7 until verified.
6. **Models.** Versions are pinned in `models.lock` with sha256. Before loading, the checksum and license are checked. Free downloading of models on the fly is forbidden.
7. **Prompts are versioned** in the repository (`agents/prompts/`) and changed via PR.

## 10. Audit and Provenance

From the audit log, one can reconstruct who, what, when, and on what basis something was done. If a step cannot be reconstructed, it cannot be released.

- **What is logged:** every `api` call, every agent action, every LLM call (reference to prompt and response), every read of a full IP, every deletion under policy, every `PolicyAgent` denial.
- **Format:** `AuditEvent` from section 5. The log is append-only. Each entry stores the hash of the previous one; the chain is verified by a nightly job.
- **Access:** writes only via the `audit` service. No DB role has UPDATE or DELETE rights.
- **Provenance in reports:** every statement references `claim_id`. The report shows source, date, method, and confidence. A statement without a reference does not enter the report.
- **Honest accuracy:** geodata from offline databases is shown with an accuracy radius and a “city-level” note. The words “exact location” are not used in the UI or reports.

## 11. Deployment and Network

**MVP:** Docker Compose on one machine (GPU 24–48 GB VRAM, 64–128 GB RAM, 2–4 TB NVMe). Services: `ollama`, `postgres` (PostGIS), `neo4j`, `qdrant`, `minio`, `redis` or `nats`, `api`, `orchestrator`, `llm-gateway`, `geo-enrichment`, `collectors`, `ui`, `nginx`.

**Network:**

- The `internal` network is created with `internal: true`. All services are included; it has no internet access.
- The `egress` network exists only for `collectors`. Egress goes through a proxy with an allowlist of domains from `config/egress-allowlist.yaml`. Any new domain is added via PR.
- Only `nginx` is exposed externally, bound to localhost or a LAN interface. Other ports are not published.
- If a fully offline mode is needed, `collectors` works only with local dumps and feeds from `data/offline/`.

**Secrets and images:** secrets via Docker secrets, never in the repository. Images are pinned by digest. Geo databases (MMDB, IP2X) are placed in `data/geo/` with version and checksum.

**Path to production:** Kubernetes with namespaces `ingest`, `storage`, `agents`, `llm`, `ui`, `security`, NetworkPolicy default deny, mTLS between services, Vault for secrets, Temporal instead of LangGraph. We do not do this before the MVP.

## 12. Repository Structure and Development Rules

```text
molox/
  contracts/        JSON Schema contracts (section 5)
  services/
    collectors/     source adapters
    normalizer/
    geo_enrichment/
    knowledge/
    agents/         LangGraph graphs, prompts/
    llm_gateway/
    api/
    audit/
  ui/
  config/           policies.yaml, retention.yaml, egress-allowlist.yaml
  data/             offline/, geo/ (not in git, versions in manifest)
  deploy/           docker-compose.yml, nginx/
  docs/adr/         architecture decisions
  tests/            contract/, e2e/, policy/
```

**Rules:**

- **Branches and review.** Work happens in `feature/*`; into `main` only via PR with one reviewer. A PR that changes `contracts/`, `config/policies.yaml`, or module boundaries requires review by the technical lead.
- **Module boundaries** are checked by `import-linter`: a service does not import another service's code, only a client of its API.
- **Tests.** Contract tests for each schema, a policy test that checks that `geo-enrichment` rejects everything except IP, and an e2e test of the end-to-end scenario. CI without green tests does not allow entry into `main`.
- **Quality.** Linter and type checking (ruff, mypy for Python) are mandatory.
- **ADR.** Any deviation from the regulation: a short file in `docs/adr/` with context, decision, and consequences.

## 13. MVP Plan and Definition of Done

The MVP shows one end-to-end scenario on data from logs and public infrastructure sources, without real people.

1. The operator asks in chat, for example: “Which ASNs and organizations are behind the suspicious IPs from this log over the last 24 hours?”
2. `CollectorAgent` and `NormalizerAgent` load the log and public feeds into observations.
3. `EnrichmentAgent` enriches IPs via `geo-enrichment`; `LinkerAgent` builds the graph IP → ASN → organization → event.
4. `VerifierAgent` and `RedTeamAgent` check hypotheses; `PolicyAgent` checks the request.
5. `ReportAgent` forms an answer with references to claims, confidence, and accuracy radius. The operator confirms the report.

| Module | Ready when |
| --- | --- |
| `collectors` | One adapter and mock data write raw data to MinIO and messages to the queue |
| `normalizer` | Observations pass schema validation, duplicates are merged |
| `geo-enrichment` | Returns `GeoResult` with radius, rejects non-IP at the schema level (test green) |
| `knowledge` | Graph and claims with versions, provenance on every edge |
| `agents` | One end-to-end cycle with checkpoints and an operator confirmation point |
| `llm-gateway` | One model in Ollama, structured output, logging to audit |
| `audit` | Write, hash chain verification, viewing in the UI |
| `ui` | Map, graph, and chat work on the same APIs as the tests |

**Not doing before MVP:** Kubernetes, Temporal, ClickHouse, model fine-tuning, more than one source adapter, more than one model.

## 14. Prohibitions and Review Checklist

Any item below blocks merge. The reviewer goes through the list for each PR.

- [ ] No `Person` and `Device` entities and personal attributes (phone, email, nickname).
- [ ] No code that links IP to a person or determines the location of a specific person.
- [ ] The phone part of IP-Tracker and the username/email recon of Akashic are not connected.
- [ ] `geo-enrichment` accepts only IP; `accuracy_radius_m` is present in every response.
- [ ] Every fact is recorded with source, time, confidence, method, and author.
- [ ] No direct access to other tables and storages, only via the owner's API.
- [ ] No calls to Ollama bypassing `llm-gateway`.
- [ ] No outbound connections from services except `collectors` via allowlist.
- [ ] An agent does not combine collection, inference, and decision-making.
- [ ] The LLM does not receive unprocessed external data as instructions.
- [ ] No agent action affects the outside world without operator confirmation.
- [ ] Changes to `contracts/`, policies, and module boundaries are documented by ADR and approved by the technical lead.
