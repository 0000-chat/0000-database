# 0000-database product vision

`0000-database` is the persistent-data boundary for 0000 services. Its core
proposition is that an agent can provision its own logical database, design the
schema and tables it needs, store and query structured data, and evolve that
schema after the first run. No human needs to preconfigure the application's
tables. The owner and policy still need to be explicit.

This document is an explanation and design guide for maintainers and coding
agents. It turns the repository README and the 0000 service map into a product
concept that can guide implementation. It does not finalize an API, MCP
operation set, remote procedure call, event format, database schema, package,
or programming language.

## Status vocabulary

This guide uses three labels.

- **Decided constraint** means that the repository metadata, `AGENTS.md`, or
  the authoritative 0000 service map already fixes the point.
- **Design direction** means a proposed shape worth testing in an
  implementation.
- **Open question** means that maintainers must decide the point before it
  becomes a contract.

The repository is currently a scaffold. It contains documentation and checks,
not an application or a database. Treat examples in this guide as product
examples, not as finalized interfaces or schemas.

## Core proposition

Agents need state that survives a single run. A research agent may need a
source record, a claim, or a pending task tomorrow. A service may need a
configuration, a cursor, or a record of work completed last week. Each 0000
service should not have to invent its own persistence rules for these cases.

`0000-database` gives the agent's logical database a durable home. The agent
can create tables and indexes, write records, query them, alter the schema as
the work changes, and retire the database when it is no longer needed.

The primary product resource is a **logical database**. A **state scope** is
the authorization and lifecycle boundary used to identify who may operate on
that database. The initial design may associate one logical database with one
state scope. Whether a scope can contain several logical databases, and which
term should name each boundary, remains an open product question.

"Agent-owned" describes an operational relationship, not a legal claim. An
agent or service is the primary actor for a named logical database. Its state
scope supplies the authorization and lifecycle boundary. The deploying
operator, and the hosted platform when it is involved, remain the policy
authority. The system must make the owner and every grant explicit rather than
trusting a caller merely because it has an agent name.

## The problem and the user journey

Without a shared persistence boundary, an agent loses useful state on restart,
or every service stores it differently. That creates unclear ownership,
inconsistent retention, weak deletion behavior, and hard-to-audit access. It
also makes a later hosted composition harder because the platform has to learn
several unrelated storage conventions.

A useful journey has these steps:

1. An operator starts `0000-database` by itself, or the hosted platform
   includes it in a selected product configuration.
2. A service or agent provisions a logical database inside a state scope. The
   database has an owner, a lifecycle policy, and an explicit grant for the
   caller.
3. The agent designs the tables it needs and stores structured facts and work
   state. It can query that state on the next run without asking the language
   model to reconstruct it from a conversation.
4. The control plane checks each request against the caller, scope, operation,
   and resource limits. It records the operation in an audit trail suitable
   for the deployment's policy.
5. A later run reads the state, changes only the records it is allowed to
   change, and can leave a provenance record for the user or another service.
6. The owner or operator can inspect, export, suspend, archive, or delete the
   logical database. The lifecycle result must be observable and testable.

The user should be able to understand what an agent stored, which agent wrote
it, and what will happen when the database is archived or deleted. The
database does not need to provide a product-specific user interface to make
those facts available. It needs inspectable operations and clear
administrative controls.

## Database state is not brain memory

`0000-brain` owns reasoning and decision capabilities. If it provides a memory
capability, that capability can decide what context to recall, summarize,
rank, interpret, or use for a decision. `0000-database` owns persistence and
integrity. It stores the structured records and returns them according to
explicit queries and permissions.

The distinction matters in both directions:

| Concern | `0000-database` | `0000-brain` or its memory capability |
| --- | --- | --- |
| Primary job | Store and retrieve structured state | Reason over context and make decisions |
| Truth behavior | Preserve the recorded value and its provenance | Assess, summarize, compare, or use the value |
| Recall behavior | Execute an explicit query or read | Choose what to recall and how to frame it |
| Lifecycle | Scope, grant, migrate, export, archive, and delete records | Decide what context should remain useful, subject to storage policy |
| Relationship | May persist brain inputs or outputs | May optionally use the database |

The database must not silently become a reasoning engine, a relevance ranker,
or a semantic-memory policy. No hard dependency from `0000-database` to
`0000-brain` follows from this optional use.

## Three layers

The proposed architecture has three layers with separate responsibilities.

```text
agent or 0000 service
          |
          v
MCP agent interface
          |
          v
control plane
          |
          v
isolated database
          |
          v
Durable Object + SQLite (proposed runtime direction)
```

`0000-gateway` may sit in front of the agent interface as an optional entry or
adaptation boundary. `0000-platform` may supply hosted account, identity,
tenancy, and policy context. Neither is required for the standalone service.

### Agent interface via MCP

The **design direction** is an MCP-facing interface for agents and compatible
services. It should let an agent provision and manage a logical database,
define its schema, operate on its data, and request lifecycle actions. The
interface should return enough information for an agent to reason about
success, conflicts, denied access, and retry safety.

An illustrative capability list is intentionally non-contractual:

- create, list, describe, and delete a logical database;
- inspect the database schema;
- create or alter a table;
- create an index;
- insert, update, delete, and query data; and
- snapshot and restore a logical database.

The names and argument shapes are open. A query means a bounded structured
request whose form still needs a product decision. This list does not promise
direct arbitrary SQL execution.

MCP is an intended integration shape, not a finalized protocol contract for
this repository. Maintainers must not infer tool names, resource names,
argument shapes, error codes, or schemas from this guide. The interface may
also need a service-to-service adapter; that choice is open.

### Control plane

The **design direction** is a control plane between the agent interface and
storage. It should:

- identify the caller, requested logical database, and enclosing state scope;
- authorize the requested operation and data range;
- apply quotas, timeouts, and request-size limits;
- manage provisioning, schema versions, migrations, suspension, and deletion;
- produce audit records without exposing stored values unnecessarily; and
- route a request to the correct isolated database.

The control plane is a product responsibility, not a promise that it must be a
separate deployable process. A first implementation may place these duties in
one Worker and split them later if the boundary remains explicit.

### Isolated database

The **design direction** is a logically isolated database for each chosen
logical database resource. The database should let its agent define tables and
indexes, then provide structured records, transactions, schema versioning,
and a defined result for concurrent writes. A state scope supplies the
authorization and lifecycle context around the database. Isolation must cover
both data access and administrative actions. A caller with access to one
database must not discover or modify another database through queries, errors,
or resource identifiers.

The exact relationship between logical databases and state scopes is an **open
question**. The initial hypothesis is one logical database per state scope, but
a scope might later contain several databases. The implementation must choose
one model for the MVP and record that decision before publishing an external
contract.

## Standalone operation

Standalone use is a product requirement, not a development convenience.

**Decided constraints:**

- The service is a public, independently usable component.
- The current product metadata declares no dependencies.
- Standalone use does not require `0000-platform` authentication. A deployment
  may add authentication at its own public boundary.
- Cloudflare is the normal public ingress and runtime class.
- `0000-cloud` is not required for an independent operator. It owns hosted
  deployment and operations for the hosted configuration.

A standalone operator should be able to choose the public-boundary adapter,
identity provider, and local policy without importing hosted account or
tenancy assumptions. Those choices remain outside the database's confirmed
dependency list. A public endpoint with no authentication would still need a
deliberate threat model and rate limits; "not required" does not mean
"appropriate for every deployment."

## 0000 ecosystem relationships

The following relationship terms come from the 0000 vision. A hard dependency
means the dependent service cannot operate in a supported configuration without
the other service. Optional use means a configuration may choose the service
for more capability. A workspace relationship helps local development and is
not a runtime dependency.

| Service | Relationship to `0000-database` | Confirmed ownership or status |
| --- | --- | --- |
| `0000-database` | Can operate alone and provide durable state to other services | No current dependencies |
| `0000-streams` | Requires the database | The only confirmed hard dependency is `0000-streams` -> `0000-database` |
| `0000-brain` | May use the database when a product configuration needs durable state | Owns reasoning and decisions; no hard dependency |
| `0000-gateway` | May provide an entry or adaptation boundary | Optional; no hard dependency |
| `0000-platform` | May host and compose the database with accounts, authentication, tenancy, and policy | Owns the hosted product context; no database dependency is confirmed here |
| `0000-cloud` | Deploys and operates hosted services | Owns hosted deployment and operations; not required for independent operation |
| `0000-communicator` | A hosted configuration may combine it with the database | Owns communication channels and bridges; no hard dependency |
| `0000-full` | Coordinates several repositories for development | Workspace relationship only, not a runtime dependency |

The main `0000` product may select a set of services later. Its final service
set is not fixed. These relationships do not authorize a new dependency. A
future hard dependency needs a product decision and an update to the
authoritative workspace metadata.

## How to maintain this guide

`0000-full/docs/GRAND_VISION.md` and `workspace.json` remain authoritative for
ecosystem roles, relationship terms, and confirmed dependencies. This document
owns the database-specific product direction. Promote an open question to a
decision only after implementation, security, or operational evidence supports
it, and then record the decision in the appropriate contract or repository
metadata. Keep `README.md` and this documentation index aligned with the
guide's status.

## Cloudflare Durable Object and SQLite direction

The preferred **design direction** is to run the service on a Cloudflare
Worker and map one logical database to one Durable Object with one SQLite
database. This one-database-to-one-object mapping is the initial hypothesis,
not a final topology. The fit is worth testing because the normal 0000 runtime
is Cloudflare, Durable Objects can give a database a stable coordination point,
and SQLite can represent structured state with transactions and queries.

The likely logical path is:

```text
logical database identity -> selected Durable Object -> that object's SQLite database
```

Maintainers still need evidence for the mapping and topology, hot-database
behavior, storage limits, migration handling, backup and restore, export speed,
cost, and failure recovery. A different Cloudflare storage arrangement remains
possible if it meets the product criteria.

### What is fixed and what is not

| Category | Current statement |
| --- | --- |
| Decided constraint | The repository is scaffold-only, has no application code or schema, and has no current dependency. |
| Decided constraint | Cloudflare is the normal public ingress and runtime class. |
| Decided constraint | Standalone platform authentication is not required; a deployment may authenticate its own public boundary. |
| Design direction | Agents reach the service through an MCP-facing interface, with a control plane enforcing scope, permissions, limits, and lifecycle. |
| Design direction | One logical database maps to one isolated Durable Object plus SQLite database for the initial hypothesis. |
| Open question | The exact MCP contract, service adapter, object topology, identity model, scope granularity, schema, and migration format. |
| Open question | Backup, restore, retention, encryption, replication, observability, quotas, and cost targets. |

Do not present the design direction as a dependency or as an API commitment.
The use of Cloudflare implementation features does not create a dependency on
`0000-cloud` or on any other 0000 repository.

## Product principles

1. **Make ownership explicit.** Every logical database has a named owner and
   a policy authority. Its state scope supplies the authorization context.
   Records should carry enough provenance to explain who wrote them and under
   what grant.
2. **Keep persistence separate from reasoning.** Store exact structured state;
   leave interpretation, ranking, and decisions to the service that owns those
   capabilities.
3. **Use least authority.** A caller receives the smallest scope and set of
   operations needed for its task. Read access does not imply write or
   administrative access.
4. **Make lifecycle operations ordinary operations.** Provisioning,
   suspension, export, archival, and deletion need clear results, audit
   records, and tests.
5. **Protect integrity before adding convenience.** Define transaction,
   conflict, migration, and retry behavior before adding broad query features.
6. **Keep the standalone path complete.** A hosted composition can add
   identity and policy, but the core service must remain usable without it.
7. **Prefer inspectable state.** Users and operators should be able to find
   what an agent stored, inspect provenance, and export it in a documented
   form.
8. **Treat contracts as deliberate.** Do not freeze protocol, endpoint, or
   schema details until the isolation, permission, and lifecycle assumptions
   have evidence.

## Capabilities and lifecycle

The product should eventually cover these capabilities. The list describes
behavior, not a committed API.

- **Provision a logical database.** Create a database with an owner, state
  scope, policy, and storage status.
- **Design a schema.** Let the agent create or alter tables and indexes without
  requiring a human to predefine the application's data model.
- **Store structured records.** Write typed or otherwise validated values with
  provenance and timestamps where the product needs them.
- **Read and query.** Retrieve records by explicit database and query criteria,
  with bounded work and predictable errors.
- **Change state safely.** Use transactions, idempotency, and conflict rules
  that an agent can understand and retry.
- **Evolve state.** Track schema versions and run migrations with a recovery
  path. The actual schema format is open.
- **Inspect and audit.** Show database metadata, schema, grants, usage,
  lifecycle status, and operation history without leaking unrelated data.
- **Export and restore.** Give an owner an explicit, documented data format and
  a tested way to recover from a backup or snapshot.
- **Retire state.** Suspend writes, archive data, apply retention, and delete
  the logical database with a result that the owner can verify.

The lifecycle should be explicit:

```text
provision -> use -> evolve or inspect -> suspend or archive -> export or delete
```

The arrows describe a product sequence, not a finalized state machine. A
logical database may be restored or reactivated only under a documented
policy. Deletion must state whether backups, audit records, and derived
indexes are also removed, and on what schedule.

## Permissions, safety, and governance

The database will hold state that an agent can use to act later. A permissive
tool that accepts arbitrary scope identifiers would turn one compromised agent
into a cross-project data reader. Safety therefore belongs in the core design,
not in an integration's prompt.

### Access control

- Authenticate the public-boundary caller when the deployment requires it.
- Bind every request to an explicit principal, state scope, and operation.
- Separate data reads and writes from grants, schema changes, exports, and
  deletion.
- Deny access by default and avoid treating an agent-provided label as proof
  of identity.
- Define how hosted platform identity maps to database principals without
  making platform identity mandatory for standalone deployments.
- Make grant creation, delegation, rotation, and revocation auditable.

### Data safety

- Limit query cost, result size, write size, and request duration.
- Use parameterized queries and validate values before storage.
- Make retries safe where possible and return a distinct result for a
  committed operation whose response was lost.
- Test cross-scope access, malformed requests, concurrent updates, migration
  failure, partial export, and deletion behavior.
- Decide encryption at rest and in transit, key ownership, secret handling,
  and redaction rules before production use.

### Governance and operations

- Record actor, scope, operation, time, outcome, and a non-sensitive request
  reference in the audit trail.
- Give the operator a way to inspect health, capacity, quotas, failed
  migrations, and pending deletion.
- Define retention for records, backups, audit entries, and derived data.
- Provide an export before destructive lifecycle actions when policy allows
  it, and make irreversible deletion explicit.
- Set recovery objectives and test restore rather than assuming a durable
  storage primitive is a complete backup plan.

These are product safeguards and validation requirements. They do not select a
particular identity provider, policy language, encryption vendor, or hosted
operator.

## MVP and phased evolution

The first release should prove that one independent agent can keep and recover
structured state safely. Keep the release small enough to test the ownership
and lifecycle model before adding memory features or broad hosted composition.

### MVP scope

- One standalone Cloudflare deployment using a Durable Object plus SQLite, if
  the runtime tests pass.
- One documented logical database model with one owner and explicit grants.
- A small MCP operation set for logical-database selection, bounded reads and
  writes, queries, metadata inspection, and lifecycle actions. Names and
  argument shapes remain to be decided.
- Control-plane checks for identity, scope isolation, resource limits, and
  audit records.
- A tested path for restart recovery, schema change, export, and deletion.
- A concrete research-agent scenario as the first acceptance test.

The MVP does not require `0000-platform`, `0000-gateway`, `0000-brain`,
`0000-cloud`, or `0000-streams` to run. A later integration may use those
services through explicit contracts.

### Phased evolution

1. **Product contract and runtime test.** Decide the MVP scope owner, threat
   model, durability and latency targets, and failure semantics. Measure the
   Durable Object plus SQLite direction against those targets.
2. **Standalone state service.** Implement the narrow interface, control-plane
   checks, isolation tests, migrations, audit records, and lifecycle actions.
3. **Service integration.** Integrate `0000-streams` as its confirmed
   dependent. Add optional adapters for `0000-brain`, `0000-gateway`, or
   `0000-communicator` only when a product configuration needs them.
4. **Hosted composition.** Let `0000-platform` provide hosted accounts,
   authentication, tenancy, policy, and composition. Let `0000-cloud` provide
   the hosted deployment and operations. Keep the standalone mode intact.
5. **Operational hardening.** Add tested backup and restore, capacity and cost
   controls, migration tooling, incident procedures, and stronger observability
   based on measured use.

Each phase needs evidence before the next contract is fixed. New integrations
must not be recorded as hard dependencies merely because they are convenient
in one hosted profile.

## Concrete example: a Southeast Asia fintech research agent

Suppose a user asks an agent to build a research set for Southeast Asian
fintech. No human designs an application schema in advance. The agent
provisions a logical database, inspects its empty schema, and creates the
tables it needs for companies, founders, funding rounds, investors, products,
countries, and sources.

1. The agent provisions the database inside a state scope and records its
   owner and policy. The control plane grants the agent access to that database
   without requiring a human to create tables first.
2. As it researches, the agent adds companies and founders, links funding
   rounds to investors, associates products with countries, and records each
   source with a canonical reference, retrieval time, content hash, and source
   type. It can mark a source as primary when it comes from a company filing,
   regulator, investor announcement, or other authoritative record.
3. The agent evolves the schema when the research requires regulatory data. It
   adds a regulatory-licenses table, links licenses to companies and countries,
   and records the issuing regulator and primary source. The database accepts
   the schema change under migration and permission rules; no human has to
   redesign the application before the agent can continue.
4. On later runs, the agent queries missing sources, changed funding rounds,
   and companies whose license status needs review. It writes updates with
   provenance and can retry a lost response without silently duplicating a
   record. `0000-brain` may assess the evidence and compose a report, while
   the database preserves the recorded facts.
5. The user then asks, "Which Philippine fintech companies have a BSP license,
   and which of them have raised funding?" The agent queries the company,
   country, regulatory-license, funding-round, investor, and source records.
   It can distinguish a primary BSP or company source from a secondary article
   before presenting the answer.

The example tests the product's central promise: the agent creates and evolves
the logical database as its work changes. If the product later uses an event
stream for source changes, `0000-streams` may use the database because the
service map confirms that dependency. The database itself does not need to
depend on streams to support this workflow.

## Non-goals

`0000-database` is not intended to:

- reason, plan, rank relevance, summarize, or make decisions;
- define the memory policy of `0000-brain`;
- carry conversations or connect communication channels;
- provide an event-stream product owned by `0000-streams`;
- own hosted accounts, authentication, tenancy, policy, or service
  composition;
- own hosted deployment and operations;
- require `0000-gateway` as an entry point;
- provide an arbitrary blob store, warehouse, or analytics platform in the MVP;
- freeze a public API, MCP contract, RPC contract, event format, or schema
  before validation; or
- grant an agent access outside its explicit scope.

## Decision and validation criteria

Maintainers can move a design choice into a public contract when the evidence
answers these questions:

| Criterion | Evidence to require |
| --- | --- |
| Durability | A restart, redeploy, and lost-response test recovers committed state without silent loss or duplication. |
| Isolation | Automated tests prove that a caller cannot read, write, inspect, export, or delete another logical database or scope. |
| Permissions | Each operation has a principal, scope, grant, and denial result that an operator can audit. |
| Agent usability | A research-agent run can write state, resume later, detect a conflict, and handle a retry without hidden database knowledge. |
| Integrity | Transactions, migration failures, concurrent writes, and malformed inputs have documented results and passing tests. |
| Lifecycle | Provision, suspend, archive, export, restore, and deletion behavior is observable and policy-controlled. |
| Standalone operation | A deployment works without hosted platform authentication and can add its own public-boundary authentication. |
| Ecosystem fit | `0000-streams` can depend on the service without a reverse dependency, while optional integrations remain optional. |
| Operations | Capacity, latency, storage cost, backup, restore, and incident behavior meet measured targets. |
| Contract clarity | The documented API, protocol, and schema match tested behavior and list compatibility and migration rules. |

Before implementation, maintainers should record the decisions for scope
granularity, principal identity, grant model, object mapping, query limits,
backup and restore, retention, and hosted-platform integration. Until then,
keep those choices in the design direction or open-question category.
