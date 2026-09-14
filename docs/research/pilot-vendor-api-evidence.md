# Pilot vendor API evidence

_Research date: 2026-09-15. Scope: Cloudflare Durable Objects SQLite API, query metrics, alarms, and platform limits. Sources are first-party Cloudflare documentation searched through the configured Cloudflare Docs gateway._

## Verified facts

| Fact | Evidence |
| --- | --- |
| Cloudflare recommends SQLite-backed Durable Objects for new namespaces. The SQL API provides relational queries, indexes, and transactions. | [Rules of Durable Objects](https://developers.cloudflare.com/durable-objects/best-practices/rules-of-durable-objects/) |
| Durable Object storage is private to the object and is documented as transactional, strongly consistent, and serializable. | [Access Durable Objects Storage](https://developers.cloudflare.com/durable-objects/best-practices/access-durable-objects-storage/); [Cloudflare glossary](https://developers.cloudflare.com/glossary/) |
| The SQL API's `exec()` returns a cursor; the official example materializes results with `cursor.toArray()`. | [Rules of Durable Objects](https://developers.cloudflare.com/durable-objects/best-practices/rules-of-durable-objects/) |
| Cloudflare's tracing documentation identifies SQL response attributes `cloudflare.durable_object.response.rows_read` and `cloudflare.durable_object.response.rows_written` for `durable_object_storage_exec`. It separately identifies `cloudflare.durable_object.response.db_size` for `getDatabaseSize`. These are observability attributes, not a published SQL quota. | [Spans and attributes](https://developers.cloudflare.com/workers/observability/traces/spans-and-attributes/) |
| The SQLite test documentation uses `state.storage.sql.databaseSize` and asserts it is greater than zero, confirming the current SQL storage surface exposes `databaseSize`. | [Testing with Durable Objects](https://developers.cloudflare.com/durable-objects/examples/testing-with-durable-objects/) |
| A current SQLite tutorial describes writes as synchronous and using local disks, but this does not establish the behavior or guarantees of `transactionSync()`. | [Build a seat booking app](https://developers.cloudflare.com/durable-objects/tutorials/build-a-seat-booking-app/) |

## Gaps closed by direct official reference reads

The search connector missed detailed references. Direct reads of the official pages established the following facts on 2026-09-15.

`storage.transactionSync(callback)` wraps synchronous work in a transaction and rolls back when the callback throws. The callback must not return a Promise. `sql.exec()` cannot execute transaction-control statements such as `BEGIN` or `SAVEPOINT`; use the storage transaction API. Consume each SQL cursor before an `await`, because a cursor crossing that boundary has no snapshot-isolation guarantee. `sql.databaseSize` reports SQLite bytes; cursor `rowsRead` and `rowsWritten` accumulate as the cursor is consumed. Index updates add billed row writes. PITR restores the whole object; it is not a paginated export reader. The same page describes millisecond alarm scheduling with possible delays of up to a minute during maintenance or failover. [SQLite storage reference](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/).

Published SQL limits include 100 columns per table, 100 bound parameters per query, 2 MB per string/BLOB/row, 100 KB per SQL statement, and 50 bytes for LIKE/GLOB patterns. Row count is constrained by storage rather than a separate table-row ceiling. The default CPU allowance is 30 seconds, configurable to five minutes. Alarm handlers have a 15-minute wall-time limit; HTTP/RPC work has no fixed wall-time maximum while the caller stays connected. A 32 MiB WebSocket limit is not an HTTP response limit. [Durable Objects limits](https://developers.cloudflare.com/durable-objects/platform/limits/).

Each object schedules one alarm. Delivery is at least once, and uncaught failures receive up to six retries with exponential backoff starting at two seconds. Only one alarm handler runs per object at a time, but other requests can interleave. Setting another alarm replaces the scheduled time. Constructor scheduling must preserve an existing alarm. Deleting an alarm does not guarantee cancellation of retries. [Alarm reference](https://developers.cloudflare.com/durable-objects/api/alarms/).

## Implementation consequences and remaining measurements

Put version checks, mutations and replay outcomes in one synchronous transaction. Store individual records as bounded rows; a 10 MiB database cannot be serialized into one SQL row. Materialize exports atomically, then paginate immutable rows. Measure logical bytes separately from physical SQLite growth. Enforce expiry on request admission as well as during idempotent cleanup, because alarms are not precise expiry boundaries.

Remaining experiments concern this implementation: transaction duration at maximum admitted data, snapshot consistency under mutations, crash/replay recovery, physical storage amplification, bounded cleanup, and global reservations under races. Local workerd results establish local behavior, not deployed CPU costs or a guaranteed US$25 invoice. No user product decision is needed to apply the documented API semantics above.
