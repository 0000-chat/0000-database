# Standalone pilot baseline

Research for [issue #2](https://github.com/0000-chat/0000-database/issues/2),
which is a child of [the strategy map](https://github.com/0000-chat/0000-database/issues/1).
Captured on 2026-09-12 from the local source and decision documents.

## Result

The existing worktree provides a local Hono REST and OpenAPI contract over a
deterministic in-memory mock. It supports database, table, column, and record
CRUD, typed cells, schema validation, filtering, sorting, pagination, scoped
ancestor checks, and deterministic errors. It cannot support the accepted
temporary-database pilot as a runtime because each HTTP request starts with a
new store. The current top-level database listing and public resource deletes
also need an explicit product decision before this route shape can be exposed
for the pilot.

The REST shape is useful as a contract probe and as a possible HTTP adapter for
the first persistence slice. It does not establish a deployed API, a durable
database, or an MCP contract.

## Evidence state

The findings branch is `research/standalone-pilot-baseline`, created from the
canonical repository's `main` at `ef6aa3d95f6f7df8a1a306c4c1d76192a52476e1`
(`chore: configure engineering skills`). This branch contains this findings
file only; it does not copy or modify the implementation worktree.

The implementation evidence is in
`/home/ubuntu/.codex/worktrees/0000-database/hono-health-openapi/0000-database`,
branch `codex/hono-health-openapi`, HEAD
`4383f6a4b95b1c94c9ab743da5d46c5c99f11971` (`docs: document database Worker
scaffold`). That worktree was dirty when inspected: six tracked files were
modified and the new `src/api/`, `src/mock/`, `test/`, `docs/specs/`, and
`docs/superpowers/` files were untracked. It was left unchanged.

Paths beginning with `src/`, `test/`, or `docs/` below are relative to that
implementation worktree. Its `wrangler.jsonc:1-11` has no storage binding, and
the generated `src/worker-configuration.d.ts:4-12` has an empty binding
interface.

`/home/ubuntu/0000-full/agent-artifacts/0000-database-rest-implementation-20260912/ACCEPTANCE.md`
is historical evidence. It reports an independent PASS, `./scripts/check`
exit 0, and 84 tests across 9 files on the implementation branch. It also
states that no commit, push, merge, or deployment was performed. This report
does not present those results as new verification. The source facts below
come from the inspected worktree; the only check run for this artifact is the
scaffold repository check on this findings branch.

## What the local implementation supports

The source registers 20 CRUD operations for databases, tables, columns, and
records, plus health and generated OpenAPI. Paths are nested by resource ID.
Creates return `201` and a relative `Location`; reads and patches return
`200`; deletes return `204`; collections return a page and `nextCursor`.
Record lists support typed equality filters, typed sorting, and cursor
pagination. See
`/home/ubuntu/.codex/worktrees/0000-database/hono-health-openapi/0000-database/docs/specs/rest-api-endpoints.md:20-77,126-230`
and the route registrations in
`/home/ubuntu/.codex/worktrees/0000-database/hono-health-openapi/0000-database/src/api/databases.ts:75-241`,
`src/api/tables.ts:111-289`, `src/api/columns.ts:90-263`, and
`src/api/records.ts:122-299`.

The mock has fixed `db_demo` and `db_other` fixtures. `tbl_people` has three
typed columns and four records, while `tbl_other` is empty. Create and patch
values use column IDs. Values are checked against the column type and
nullability; sibling names are case-insensitively unique; nullable columns
backfill existing records; incompatible schema changes fail atomically;
column deletes remove their cells; and table or database deletes cascade in
the mock. These behaviors are documented at
`/home/ubuntu/.codex/worktrees/0000-database/hono-health-openapi/0000-database/docs/specs/rest-api-endpoints.md:95-168,225-242`
and implemented in
`/home/ubuntu/.codex/worktrees/0000-database/hono-health-openapi/0000-database/src/mock/store.ts:168-227,323-521`.

The default app selects `createMockStore()` as its store factory and the mock
middleware creates one store for each known mock request. The store is built
from a fresh copy of the fixtures. A write is visible in its response, but a
later request sees the original fixtures. Tests may inject one shared store,
which proves the domain methods can perform a sequence but does not change the
default Worker behavior. See
`/home/ubuntu/.codex/worktrees/0000-database/hono-health-openapi/0000-database/src/app.ts:76-155`,
`src/mock/store.ts:252-288`,
`docs/specs/rest-api-endpoints.md:79-93`, and
`test/databases.spec.ts:35-68,189-201`.

The local safety checks cover input and query shape. Mutation bodies are
limited to 65,536 bytes, strings to 10,000 characters, descriptions to 1,000
characters, collection pages to 1 through 100 items, decoded filter text to
256 characters, and cursors to 2,048 characters. The implementation has no
storage counter, daily request counter, burst limiter, response-size limiter,
or returned-data counter. See
`/home/ubuntu/.codex/worktrees/0000-database/hono-health-openapi/0000-database/src/app.ts:89-97`,
`src/api/common.ts:38-75`, and
`docs/specs/rest-api-endpoints.md:165-181,225-259`.

## Comparison with the accepted temporary-database decision

| Concern | Observed source behavior | Result against the accepted direction |
| --- | --- | --- |
| Request-to-request persistence | The default Worker resets all state for every request. There is no storage binding, Durable Object, restart recovery path, or persistent schema. Sources: `src/app.ts:76-155`, `src/mock/store.ts:252-288`, `wrangler.jsonc:1-11`, `docs/specs/rest-api-endpoints.md:79-93`. | Blocking gap. The pilot needs a shared temporary database whose writes can be read by later requests. The parent service role names durable storage as the database boundary. Sources: `/home/ubuntu/0000-full/docs/0000-database/strategy/adr/0001-temporary-database-and-private-copy.md:7-24`, `/home/ubuntu/0000-full/docs/GRAND_VISION.md:59-62`. |
| Slug access and discovery | `Database` contains `id`, `name`, and `description`, with no slug. IDs use a generic 64-character pattern and new IDs come from `crypto.randomUUID()`. The public route inventory includes `GET /v1/databases`, and the fixed fixtures expose two database IDs. The generated OpenAPI document has no security scheme. Sources: `src/api/databases.ts:15-24`, `src/api/common.ts:38-41`, `src/mock/store.ts:246-254`, `src/mock/fixtures.ts:13-18`, `docs/specs/rest-api-endpoints.md:39-47`, `test/openapi.spec.ts:283-290`. | The accountless part is compatible with possession-based access, but slug issuance, lookup, entropy, warning text, and leak behavior are missing. A public database list conflicts with the ADR's unlisted and non-directory-indexed requirement unless it is restricted or removed. The absence of hosted platform authentication is allowed by the architecture, but it does not define the standalone threat model or rate limits. Sources: `/home/ubuntu/0000-full/docs/0000-database/strategy/adr/0001-temporary-database-and-private-copy.md:7-11`, `/home/ubuntu/0000-full/docs/ARCHITECTURE_HANDOFF.md:46-64`, `docs/PRODUCT_VISION.md:199-218`. |
| Resource deletion versus row deletion | The mock publishes `DELETE` for databases, tables, columns, and records. Database and table deletion cascade to children; record deletion removes a row; column deletion removes its cells. Sources: `src/api/databases.ts:194-211`, `src/api/tables.ts:234-251`, `src/api/columns.ts:213-230`, `src/api/records.ts:243-260`, `src/mock/store.ts:355-361,396-401,466-478,517-521`. | Record deletion and other destructive data or schema operations may fit the anonymous write window. Public database resource deletion conflicts with the ADR, which assigns removal to an operator or expiry policy. The treatment of nested table and column resource deletes needs an explicit reading of that rule. Source: `/home/ubuntu/0000-full/docs/0000-database/strategy/adr/0001-temporary-database-and-private-copy.md:8-18`. |
| Lifecycle | Database metadata has no creation time, inactivity time, write cutoff, status, or cleanup marker. Every local mutation remains writable, and no request updates lifecycle state. Sources: `src/api/databases.ts:15-24`, `src/mock/store.ts:323-521`. | Blocking gap. The ADR requires a 30-day write window, read-only behavior afterward, seven-day inactivity expiry, capacity-suspension handling, and no age-only deletion. Source: `/home/ubuntu/0000-full/docs/0000-database/strategy/adr/0001-temporary-database-and-private-copy.md:13-24,36-40`. |
| Limits | The code enforces request and value validation, including a 65,536-byte mutation body and page bounds. It does not enforce storage, request, mutating-request, burst, ordinary-response, or returned-data budgets. Sources: `src/app.ts:89-97`, `src/api/common.ts:53-75`, `docs/specs/rest-api-endpoints.md:165-181,225-259`. | The body limit matches the ADR's 64 KiB value. The accepted 10 MiB storage, 1,000 daily data requests, 100 daily mutating requests within that total, 60-per-minute burst, 256 KiB ordinary response, and 100 MiB daily returned-data quotas are absent. Row-work, schema/index counts, and execution limits remain open in the ADR, so those still need measurement rather than invention. Source: `/home/ubuntu/0000-full/docs/0000-database/strategy/adr/0001-temporary-database-and-private-copy.md:20-40`. |
| Export | No export operation or route appears in the 20-operation local contract. The local docs list exports, backup, and restore as outside the slice. Sources: `docs/specs/rest-api-endpoints.md:20-77,261-272`, `test/openapi.spec.ts:52-120`. | Blocking gap for the pilot lifecycle. Bounded exports must remain available after the write cutoff, with a reserved export allowance. Format, authorization, size, and accounting still need decisions. Source: `/home/ubuntu/0000-full/docs/0000-database/strategy/adr/0001-temporary-database-and-private-copy.md:13-24`. |
| Interface suitability | The nested REST paths, typed request schemas, error envelope, pagination, and generated OpenAPI document give clients a testable HTTP contract. The route shape is described as compatible with a future Durable Object routing model. Sources: `docs/specs/rest-api-endpoints.md:11-24,33-77,225-242`, `test/openapi.spec.ts:159-281`. | Suitable for local contract tests and a possible adapter seam. It is not yet a standalone pilot interface because it has no durable state, lifecycle or quota enforcement, export, actor or provenance fields, or production deployment. The product guide calls MCP an intended but non-final interface, so the REST routes cannot be treated as an MCP contract. Sources: `src/api/records.ts:16-26`, `docs/PRODUCT_VISION.md:140-180,182-197,290-311`, `/home/ubuntu/0000-full/docs/ARCHITECTURE_HANDOFF.md:69-75`. |

The accepted ADR is product direction only. It says that the behaviors it names
do not exist today, so its requirements must not be reported as implemented.
The source and local endpoint document also state that production currently
exposes only health and OpenAPI. See
`/home/ubuntu/0000-full/docs/0000-database/strategy/adr/0001-temporary-database-and-private-copy.md:7-52`
and
`/home/ubuntu/.codex/worktrees/0000-database/hono-health-openapi/0000-database/docs/specs/rest-api-endpoints.md:1-9,261-276`.

## Smallest next standalone build increment

The smallest useful increment is a persistence vertical slice behind the
existing app factory. Select one Cloudflare storage arrangement, add a durable
database identity and lookup path, and make one create-table-column-record
sequence survive separate HTTP requests and a Worker restart. Keep the current
REST route shape as the test transport while the boundary is measured. Add
tests for cross-request reads, updates, schema changes, and isolation between
database resources. Do not expose top-level database deletion until the
accepted removal rule is represented in the contract. This increment proves
the missing state boundary; it does not claim pilot readiness before lifecycle,
quotas, discovery, and export are implemented.

The parent product guide identifies Durable Object plus SQLite as an initial
hypothesis, not a selected technology. It asks for a runtime test of that
direction against durability, latency, hot-database, storage, export, cost,
and recovery targets. A different Cloudflare arrangement remains possible.
See
`/home/ubuntu/.codex/worktrees/0000-database/hono-health-openapi/0000-database/docs/PRODUCT_VISION.md:260-286,403-426`.

## Decisions required before that increment becomes a pilot

1. Decide whether the database path ID is also the public random slug or
   whether the API needs a separate slug. Decide how `GET /v1/databases` is
   prevented from directory-indexing unlisted databases, and define the warning
   shown to anyone who receives a slug.
2. Select the storage topology and its mapping from a logical database to
   durable state. Define transaction, concurrency, restart, migration, and
   failure behavior before publishing the persistent contract.
3. Define lifecycle fields and enforcement. The implementation needs an
   authoritative clock, the 30-day write cutoff, seven-day inactivity renewal
   rules, capacity-suspension behavior, and operator or expiry cleanup.
4. Define enforcement for the accepted provisional quotas, including counter
   scope and UTC reset behavior. Measure the still-open row-work,
   schema/index, and execution limits.
5. Resolve deletion and export together. Decide which nested schema deletes
   remain public, keep logical-database resource deletion operator or policy
   controlled, and choose an export format, size bound, authorization rule,
   and post-cutoff availability.
6. Decide whether REST is the standalone public contract, an internal adapter,
   or a transport for a later MCP adapter. Define public-boundary threat
   controls, audit/provenance fields, and deployment verification without
   adding a dependency on hosted platform authentication.

Private copies, paid plans, hosted integrations, and commercial limits are
later work in the map. They do not need to block the first accountless
temporary-database persistence slice.

## Primary sources

- Accepted decision: `/home/ubuntu/0000-full/docs/0000-database/strategy/adr/0001-temporary-database-and-private-copy.md`
- Strategy glossary: `/home/ubuntu/0000-full/docs/0000-database/strategy/CONTEXT.md:1-17`
- Parent architecture: `/home/ubuntu/0000-full/docs/ARCHITECTURE_HANDOFF.md:27-130` and `/home/ubuntu/0000-full/docs/GRAND_VISION.md:41-90,117-142`
- Workspace metadata: `/home/ubuntu/0000-full/workspace.json:1-84`
- Implementation worktree state: `/home/ubuntu/.codex/worktrees/0000-database/hono-health-openapi/0000-database`, branch `codex/hono-health-openapi`, HEAD `4383f6a4b95b1c94c9ab743da5d46c5c99f11971`, dirty and preserved
- Local REST contract: `/home/ubuntu/.codex/worktrees/0000-database/hono-health-openapi/0000-database/docs/specs/rest-api-endpoints.md`
- Historical acceptance: `/home/ubuntu/0000-full/agent-artifacts/0000-database-rest-implementation-20260912/ACCEPTANCE.md`
