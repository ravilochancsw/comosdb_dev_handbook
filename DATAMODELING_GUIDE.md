# Cosmos DB Architecture And Data Modeling Guide

This guide is for engineers designing backend services on Azure Cosmos DB for NoSQL. It explains how Cosmos DB wants you to model data, choose containers, pick partition keys, write queries, handle indexes, use transactions, and maintain duplicated read models.

It is intentionally written from a Node.js/backend perspective, especially for teams coming from SQL, MongoDB, or Mongoose-style modeling.

## References

Primary Microsoft documentation:

- [Data modeling in Azure Cosmos DB for NoSQL](https://learn.microsoft.com/en-us/azure/cosmos-db/modeling-data)
- [Partitioning and horizontal scaling](https://learn.microsoft.com/en-us/azure/cosmos-db/partitioning)
- [Hierarchical partition keys](https://learn.microsoft.com/en-us/azure/cosmos-db/hierarchical-partition-keys)
- [Request Units in Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/request-units)
- [Optimize request cost](https://learn.microsoft.com/en-au/azure/cosmos-db/optimize-cost-reads-writes)
- [Indexing policy](https://learn.microsoft.com/en-us/azure/cosmos-db/index-policy)
- [Indexing in Cosmos DB](https://learn.microsoft.com/en-us/cosmos-db/indexing)
- [Unique keys](https://learn.microsoft.com/en-us/azure/cosmos-db/unique-keys)
- [Query pagination](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/query/pagination)
- [Query metrics](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/query-metrics)
- [Consistency levels](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels)
- [Transactional batch REST reference](https://learn.microsoft.com/en-us/rest/api/cosmos-db/transactional-batch)
- [Stored procedures, triggers, and UDFs](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-write-stored-procedures-triggers-udfs)
- [Transactional outbox pattern with Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/architecture/best-practices/transactional-outbox-cosmos)
- [Change feed design patterns](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed-design-patterns)
- [Change feed processor](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/change-feed-processor)
- [Time to live](https://learn.microsoft.com/en-us/cosmos-db/time-to-live)
- [Architecture best practices for Cosmos DB for NoSQL](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/cosmos-db)

## 1. Cosmos DB Mental Model

Cosmos DB for NoSQL is not a relational database and should not be treated as MongoDB with a different SDK.

The important primitives are:

- Account: global Cosmos DB account.
- Database: logical namespace for containers.
- Container: scalable set of JSON items sharing a partition key definition, indexing policy, throughput configuration, TTL, and unique key policy.
- Item: JSON document with an `id` and a partition key value.
- Logical partition: all items with the same partition key value.
- Physical partition: internal infrastructure partition used by Cosmos to distribute logical partitions.
- Request Unit (RU): normalized cost of each operation.

The cheapest and most predictable read is a point read:

```text
read item by id + partition key
```

Microsoft documents that reading a 1 KB item by id and partition key costs about 1 RU. Queries cost more depending on item size, query complexity, number of partitions touched, index usage, and result size.

The main design principle:

```text
Design containers around access patterns and partition boundaries, not around traditional entity tables.
```

## 2. SQL, MongoDB, And Cosmos Compared

### SQL mindset

SQL design often starts with:

- Normalize entities.
- Create tables.
- Use joins.
- Add indexes later.
- Query flexibly.
- Use ACID transactions across tables.

This does not map cleanly to Cosmos.

### MongoDB mindset

MongoDB design often starts with:

- Collections per entity.
- Documents with embedded subdocuments.
- Secondary indexes for common filters.
- Aggregation pipelines.
- References where needed.

Cosmos has JSON documents too, but the RU and partition model is stricter. A query that feels natural in MongoDB can become an expensive cross-partition query in Cosmos.

### Cosmos mindset

Cosmos design starts with:

- What are the exact reads?
- What are the exact writes?
- What needs to be strongly consistent?
- What can be eventually consistent?
- What is the partition key for each operation?
- Can the operation be a point read?
- Can related writes live in one logical partition?
- Which read models should be duplicated?
- Which fields should be indexed?

The goal is not "one clean normalized model." The goal is "predictable, partition-aware, low-RU operations."

## 3. Domain Models Vs Container Item Models

In Node.js applications, teams often say "model" to mean a Mongoose model or ORM entity. In Cosmos, use more precise terms.

Recommended terms:

- Domain model: business concept, such as `Transfer`, `Connector`, `ProviderProfile`.
- Write model: canonical item used as source of truth.
- Read model: duplicated/projection item optimized for a screen or query.
- Lookup model: small item used to resolve a key cheaply.
- Work model: queue/scheduler/retry item.
- Event model: immutable event/audit item.

Do not force "one model equals one container."

A container may hold:

- One item type, when the container is high volume or has one clear lifecycle.
- Several related item types, when they share the same partition key and are commonly updated together.

Example:

```text
Container: transferWrite
Partition key: /transferId

Items:
- TransferDocument
- TransferEventItem
- TransferOutboxItem
```

These can live together because they are scoped to one transfer and can be written in one logical partition.

## 4. Container Naming

Container names should describe the purpose of the data, not merely the entity.

Poor names:

```text
transfers
users
runs
records
data
```

Better names:

```text
transferWrite
transferAccess
transferTenantIndex
transferWork
transferMetricsDaily
connectorInstances
connectorRuns
connectorImportDedup
connectorRetryWork
providerProfiles
userEntitlements
```

Naming convention:

```text
<domain><purpose>
```

Purpose examples:

- `Write`: canonical source of truth.
- `Access`: user-specific read projection.
- `Index`: lookup or listing projection.
- `Work`: scheduler or queue items.
- `Metrics`: precomputed aggregates.
- `Dedup`: uniqueness/idempotency records.
- `Config`: static or tenant config.
- `Events`: immutable audit stream.

Avoid environment prefixes in code-level names. Use environment-specific physical names through configuration:

```text
Logical name: transferWrite
Dev physical name: dev-aperio-transferWrite
Prod physical name: prod-aperio-transferWrite
```

## 5. Data Modeling Process

Use this process before creating a container.

### Step 1: List access patterns

For every screen, endpoint, background job, and integration, write:

```text
Operation:
Caller:
Filter:
Sort:
Page size:
Expected volume:
Latency requirement:
Consistency requirement:
Write frequency:
Read frequency:
```

Example:

```text
Operation: list received transfers for user
Caller: recipient user
Filter: recipient user id, optional status
Sort: createdAt descending
Page size: 20
Volume: up to thousands per user
Consistency: can be eventually consistent within seconds
```

This should produce a read model such as `transferAccess` partitioned by `/principalId`.

### Step 2: Identify point reads

Any "get by id" endpoint should be a point read if possible.

Good:

```text
container.item(id, partitionKey).read()
```

Bad:

```sql
SELECT * FROM c WHERE c.id = @id
```

If the application receives only an id but not the partition key, either:

- Make the id also be the partition key.
- Use a small lookup container.
- Include the partition key in URLs/tokens.
- Use hierarchical partition keys if useful.

### Step 3: Decide write consistency boundaries

Ask:

- Which items must be updated atomically?
- Can they live in the same logical partition?
- Can the rest be eventually consistent?

Cosmos transactional batch requires the same logical partition key. If your operation needs atomic updates across unrelated partition keys or containers, Cosmos will not give you a normal SQL-style transaction.

### Step 4: Design read projections

For any expensive listing query, create a projection.

Examples:

- User inbox: `transferAccess`, partition `/principalId`.
- Tenant admin list: `transferTenantIndex`, partition `/tenantId`.
- Dashboard counts: `transferMetricsDaily`, partition `/metricOwnerId`.
- Due reminders: `transferWork`, partition `/bucketShard`.

### Step 5: Design idempotency and deduplication

Use deterministic ids.

Examples:

```text
transfer access item:
id = "recipient:" + transferId
partitionKey = "user:" + recipientUserId

connector dedup item:
id = sha256(tenantId + ":" + connectorId + ":" + externalId)
partitionKey = tenantId + ":" + connectorId

user entitlement:
id = productId
partitionKey = userId
```

### Step 6: Define indexing policy

Do not blindly index all properties forever. Default automatic indexing is convenient, but high-write containers often need excluded paths.

Index only:

- fields used in filters
- fields used in sorts
- fields used in joins within document arrays if unavoidable
- partition key paths used in queries

Exclude:

- raw payloads
- large embedded JSON
- logs that are never filtered
- encrypted blobs or opaque payloads
- large arrays that are only returned by point read

## 6. Partition Keys

The partition key is the most important design decision in Cosmos.

It determines:

- data distribution
- query routing
- point-read addressability
- transactional batch boundary
- hot partition risk
- logical partition storage limits

An item is uniquely identified by:

```text
id + partition key
```

### Good partition key properties

A good partition key usually has:

- High cardinality: many distinct values.
- Even distribution: no single value dominates.
- Query alignment: common reads include the partition key.
- Write distribution: high write volume spreads across partitions.
- Transaction alignment: items that must be written atomically share the key.

### Bad partition key properties

Avoid keys that are:

- Low cardinality: `status`, `type`, `country`, `role`.
- Monotonic: raw timestamp, incrementing integer.
- Hot: one tenant/user receives most writes.
- Not present in queries.
- Too broad for large tenants: `/tenantId` can become hot or too large.

### Partition key examples

Good for transfer point reads:

```text
/transferId
```

Good for user inbox:

```text
/principalId
```

Good for tenant admin listing:

```text
/tenantId
```

Good for retry queues:

```text
/bucketShard
```

Good for connector run history:

```text
hierarchical: /tenantId, /connectorId, /runId
```

### Tenant partitioning warning

`/tenantId` is tempting for every container, but it is not always correct.

It is good when:

- most queries are tenant-scoped
- tenants are similar in size
- per-tenant data will not grow too large
- per-tenant RU pressure is moderate

It is weak when:

- one tenant is much larger than others
- most reads are by transfer id or user id
- background jobs scan across tenants
- tenant partitions become hot

For multi-tenant workloads with large tenants, consider hierarchical partition keys such as:

```text
/tenantId, /userId, /id
/tenantId, /connectorId, /runId
/tenantId, /bucket, /id
```

Microsoft documents hierarchical partition keys as a way to let tenant prefixes scale beyond the traditional single logical partition limits and route prefix queries efficiently.

## 7. Synthetic Keys

A synthetic key is an application-generated value combining multiple fields into one partition key or id.

Use synthetic keys when no natural single field fits the access pattern.

### Common synthetic key patterns

#### Principal key

```text
user:<userId>
email:<sha256(lowercaseEmail)>
tenant:<tenantId>
```

Use for inboxes and access projections.

#### Tenant connector key

```text
tenantConnectorId = tenantId + ":" + connectorId
```

Use for connector dedup and connector-local lookups.

#### Time bucket shard key

```text
bucketShard = yyyy-MM-ddTHH:mmZ + "#" + shardNumber
```

Use for scheduled work, retries, reminder queues, and expiry queues.

Example:

```text
2026-05-11T10:00Z#07
```

This avoids one hot partition for a timestamp bucket by spreading writes across N shards.

#### Search-safe email key

```text
emailHash = sha256(lowercase(trim(email)))
partitionKey = emailHash
```

Use instead of case-insensitive `LOWER(c.email)` queries.

#### Deterministic id

```text
id = sha256(tenantId + ":" + connectorId + ":" + externalId)
```

Use for deduplication, uniqueness, idempotent upserts.

### Synthetic key rules

- Keep them stable.
- Keep them deterministic.
- Normalize inputs before hashing.
- Avoid personally identifiable information in raw partition keys when possible.
- Include enough entropy to avoid hot partitions.
- Make them readable where operational debugging matters.

## 8. Query Patterns

### Best query: point read

Use when you know `id` and partition key.

```js
await container.item(id, partitionKey).read();
```

Use for:

- get transfer by id
- get provider profile by user id
- get entitlement by user id and product id
- get connector by connector id when partition key is known

### Good query: single-partition query

Use when filtering and sorting within one partition.

```sql
SELECT * FROM c
WHERE c.principalId = @principalId
  AND c.status = @status
ORDER BY c.createdAt DESC
```

Use query options with the partition key when the SDK supports it.

### Acceptable query: prefix-routed hierarchical query

With hierarchical partition keys, filtering by the leading key or leading keys can route to fewer partitions.

Example:

```sql
SELECT * FROM c
WHERE c.tenantId = @tenantId
  AND c.connectorId = @connectorId
ORDER BY c.createdAt DESC
```

### Expensive query: cross-partition query

Cross-partition queries are sometimes acceptable for:

- admin tools
- low-frequency reports
- small containers
- migration scripts
- operational diagnostics

They should not power hot user-facing screens.

### Very expensive pattern: fetchAll then slice

Bad:

```js
const { resources } = await container.items.query(query).fetchAll();
return resources.slice(offset, offset + limit);
```

Use continuation tokens and purpose-built partitions instead.

### Avoid these hot-path query patterns

Avoid:

```sql
SELECT * FROM c WHERE c.id = @id
```

Use point read.

Avoid:

```sql
SELECT * FROM c WHERE LOWER(c.email) = LOWER(@email)
```

Store normalized fields or hash keys.

Avoid:

```sql
SELECT * FROM c WHERE CONTAINS(LOWER(c.name), @search)
```

For real search, use Azure AI Search or a dedicated search index. For simple prefix/equality filters, store normalized indexed fields.

Avoid:

```sql
SELECT * FROM c WHERE c.status = 'pending'
```

If status is a low-cardinality cross-partition query, use a queue/work projection partitioned by due bucket or status plus shard.

Avoid:

```sql
SELECT * FROM c ORDER BY c.createdAt DESC
```

This is a global order-by query. Scope it to a partition or projection.

Avoid joining across documents. Cosmos SQL supports joins within a document/array, not relational joins across containers.

## 9. Pagination

Cosmos queries may return multiple pages. Use continuation tokens, not offset pagination.

Good:

```text
request: limit + continuationToken
response: items + continuationToken
```

Bad:

```text
request: page=100&offset=9900
```

Offset pagination forces the database or app to skip over previous rows. It becomes more expensive as the offset grows.

Continuation token rules:

- Treat tokens as opaque.
- Do not parse them in application code.
- Return them to clients as-is.
- Keep query shape stable between pages.
- Prefer single-partition queries for predictable paging.

Microsoft documents that Cosmos query executions are stateless server-side and can resume using continuation tokens.

## 10. Indexing

Cosmos DB indexes item paths. The default policy indexes every property automatically, which is convenient but can be expensive for write-heavy containers or large documents.

Indexing affects:

- query RU cost
- write RU cost
- storage cost
- index transformation time

### Index types

Cosmos supports several index types, including:

- Range indexes: equality, range, string functions, array contains.
- Composite indexes: multi-field `ORDER BY`, and many filter plus sort patterns.
- Spatial indexes: geospatial queries.
- Vector indexes: vector search scenarios.

### Default indexing

Default indexing:

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    { "path": "/*" }
  ],
  "excludedPaths": [
    { "path": "/\"_etag\"/?" }
  ]
}
```

This is fine for prototypes. It is rarely ideal for mature write-heavy systems.

### Excluding large paths

If a transfer has raw payload data or large arrays that are never filtered, exclude them:

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    { "path": "/*" }
  ],
  "excludedPaths": [
    { "path": "/\"_etag\"/?" },
    { "path": "/rawPayload/*" },
    { "path": "/rawGraph/*" },
    { "path": "/contentRefs/files/*" },
    { "path": "/auditBlob/*" }
  ]
}
```

Better: do not store raw payloads in Cosmos at all. Store them in Blob Storage and keep a pointer.

### Minimal indexing for read models

For a `transferAccess` container:

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    { "path": "/principalId/?" },
    { "path": "/role/?" },
    { "path": "/status/?" },
    { "path": "/createdAt/?" },
    { "path": "/updatedAt/?" },
    { "path": "/urgency/?" }
  ],
  "excludedPaths": [
    { "path": "/*" }
  ],
  "compositeIndexes": [
    [
      { "path": "/principalId", "order": "ascending" },
      { "path": "/createdAt", "order": "descending" }
    ],
    [
      { "path": "/principalId", "order": "ascending" },
      { "path": "/status", "order": "ascending" },
      { "path": "/createdAt", "order": "descending" }
    ],
    [
      { "path": "/principalId", "order": "ascending" },
      { "path": "/role", "order": "ascending" },
      { "path": "/createdAt", "order": "descending" }
    ]
  ]
}
```

Note: exact indexing policy syntax must be validated against the SDK/API version used.

### Composite indexes

Use composite indexes for queries like:

```sql
SELECT * FROM c
WHERE c.principalId = @principalId
  AND c.status = @status
ORDER BY c.createdAt DESC
```

Composite index:

```text
/principalId ASC, /status ASC, /createdAt DESC
```

Without the right composite index, Cosmos may still run the query, but RU cost can be higher.

### Unique keys

Unique keys enforce uniqueness within a logical partition. They must be defined when creating the container and cannot be added later to an existing container.

Example:

```text
partition key: /tenantId
unique key: /emailHash
```

This guarantees unique email per tenant, not globally.

For many scenarios, deterministic ids are simpler than unique keys:

```text
id = emailHash
partitionKey = tenantId
```

## 11. Transactions And ACID Semantics

Cosmos supports ACID transactions only within a single logical partition.

Supported options:

- Transactional batch: multiple operations with same partition key.
- Stored procedures: JavaScript executed inside the database engine, scoped to a partition key for partitioned containers.
- Triggers: pre/post JavaScript logic for operations, also constrained by partition behavior.

Not supported like SQL:

- Cross-container ACID transactions.
- Cross-partition ACID transactions.
- Distributed transactions across unrelated records.
- Long-running server-side transactions.

### Transactional batch

Use transactional batch when multiple items share the same partition key.

Example:

```text
Container: transferWrite
Partition key: /transferId

Batch:
- patch TransferDocument status
- create TransferEventItem
- create TransferOutboxItem
```

All succeed or all fail because they share `transferId`.

### Stored procedures

Stored procedures can provide transactional execution in JavaScript. For partitioned containers, the caller must provide a partition key value. Stored procedures are less commonly used in modern Node.js services than transactional batch, but they still exist for server-side logic.

### Cross-container consistency

If you write to `transferWrite`, `transferAccess`, and `transferMetricsDaily`, you cannot make that one ACID transaction across containers.

Use one of these patterns:

#### Pattern A: synchronous dual write

Write canonical item, then write projection items.

Pros:

- Simple.
- Projection updates are immediate if all writes succeed.

Cons:

- Partial failure risk.
- Requires compensating retries.
- Harder to reason about at scale.

Use for early-stage systems or low-risk projections.

#### Pattern B: transactional outbox

Within the canonical partition, write:

- canonical item update
- event item
- outbox item

Then process outbox asynchronously to update other containers.

Pros:

- Canonical write and event are atomic.
- Projection updates can be retried.
- Good auditability.

Cons:

- Eventual consistency.
- More infrastructure.

This is often the best pattern for Cosmos.

#### Pattern C: change feed projection

Use Cosmos DB change feed or Azure Functions Cosmos DB trigger to react to changes and update projections.

Pros:

- Scalable.
- Avoids polling full containers.
- Natural for materialized views.

Cons:

- At-least-once delivery means handlers must be idempotent.
- Eventual consistency.
- The official change feed processor library support varies by SDK; Azure Functions triggers are often the practical route for JavaScript/Node teams.

## 12. Consistency Levels

Cosmos DB offers five consistency levels:

- Strong
- Bounded staleness
- Session
- Consistent prefix
- Eventual

For most web applications, session consistency is a good default because it gives read-your-writes behavior within a client session while preserving better performance than strong consistency.

Important tradeoffs:

- Strong consistency has higher latency/throughput cost and does not pair with multiple write regions in the same way weaker levels do.
- Session consistency is usually developer-friendly.
- Eventual consistency is useful for projections, metrics, search indexes, and derived views.

Separate consistency by use case:

- Canonical transfer read after update: session consistency or point read with session token.
- Inbox projection: eventual consistency acceptable within seconds.
- Metrics: eventual consistency acceptable.
- Audit/event write: transactional with canonical update if in same partition.

## 13. Change Feed And Materialized Views

Change feed records changes to items in a container.

Use it for:

- Maintaining read models.
- Updating metrics.
- Sending notifications.
- Synchronizing to search.
- Archiving to Blob/Data Lake.
- Triggering external workflows.

Common pattern:

```text
transferWrite changes
  -> projection processor
  -> transferAccess
  -> transferTenantIndex
  -> transferMetricsDaily
  -> transferWork
```

Handlers must be idempotent.

Example idempotency:

```text
Projection item id = deterministic value
Use upsert
Store source event id or source version
Ignore older versions
```

## 14. TTL And Lifecycle

TTL automatically expires items after a configured number of seconds.

Use TTL for:

- temporary work items
- retry records after retention period
- session records
- short-lived email token records
- old projection records
- logs with fixed retention

Do not blindly TTL canonical medical/legal audit records unless retention requirements allow it.

Example:

```json
{
  "id": "expiry:transfer_123",
  "bucketShard": "2026-05-18T00:00Z#07",
  "itemType": "transferWork",
  "ttl": 2592000
}
```

TTL deletion runs in the background. In provisioned throughput accounts, deletion uses leftover RUs when available.

## 15. Common Cosmos Data Modeling Patterns

### Aggregate document

Store data together when it is read together and updated together.

Example:

```text
TransferDocument embeds sender, recipient, referral summary, workflow status, content refs.
```

### Event stream in same partition

Store immutable events next to canonical item.

```text
partitionKey = transferId
id = event_<timestamp>_<uuid>
```

### Materialized view

Duplicate summary data for a specific screen.

```text
transferAccess for inbox
transferMetricsDaily for dashboard
```

### Lookup item

Small item to resolve an alternate key.

```text
emailHash -> userId
transferId -> transfer partition
externalId -> imported transfer
```

### Work queue item

Do not scan all items looking for work. Write work items when work is scheduled.

```text
transferWork due at 2026-05-18T00:00Z
connectorRetryWork due at 2026-05-11T00:15Z
```

### Dedup item

Use deterministic ids to make retries safe.

```text
id = sha256(sourceSystem + ":" + externalId)
```

### Bucketed time-series

Group metrics by day/hour/user/tenant instead of scanning raw events.

```text
id = 2026-05-11
partitionKey = user:<id>:sender
```

## 16. Anti-Patterns

Avoid these in production hot paths:

- One container per SQL table without considering access patterns.
- `/tenantId` as partition key for every container.
- `SELECT *` for every query.
- Cross-partition `fetchAll()` for user-facing endpoints.
- Offset pagination.
- Case-insensitive functions on indexed fields instead of normalized fields.
- Large raw payloads inside Cosmos documents.
- Global status queues such as `WHERE status = 'pending'`.
- Dashboard metrics computed by scanning all source records.
- Storing ever-growing arrays in one item.
- Assuming unique `id` is global across the container. It is unique only within a partition key.
- Treating duplicated data as a bug. In Cosmos, purposeful duplication is normal.

## 17. Data Access Layer Design

Controllers should not know Cosmos.

Recommended layering:

```text
controllers/
  transfer.controller.js

services/
  transfer.service.js

data/
  cosmos/
    cosmosClient.js
    containerRegistry.js
    transferWrite.store.js
    transferAccess.store.js
    transferWork.store.js
    transferMetrics.store.js
```

Responsibilities:

### Controller

- HTTP request/response.
- Input extraction.
- Status code mapping.

### Service

- Business workflow.
- Authorization decisions.
- Calls stores.
- Coordinates canonical writes and projections/outbox.

### Store/repository

- Knows container name.
- Knows partition key.
- Knows query text.
- Knows continuation token handling.
- Knows patch vs replace.
- Logs RU charge and diagnostics.

Example store API:

```js
transferWriteStore.getById(transferId)
transferWriteStore.patchStatus(transferId, expectedVersion, patch)
transferAccessStore.listForPrincipal(principalId, filters, page)
transferWorkStore.claimDueWork(bucketShard, workerId)
transferMetricsStore.incrementDaily(ownerId, date, counters)
```

Service code should read like business logic:

```js
async function acknowledgeTransfer({ transferId, actor }) {
  const transfer = await transferWriteStore.getById(transferId);
  assertRecipientCanAcknowledge(transfer, actor);

  const event = buildTransferAcknowledgedEvent(transfer, actor);
  await transferWriteStore.applyEvent(transferId, event);

  return transferReadModel.getMetadata(transferId, actor);
}
```

## 18. Observability And RU Discipline

Every data access method should log:

- operation name
- container
- partition key
- request charge
- item count
- continuation token presence
- elapsed time
- whether cross-partition
- query metrics for slow/high-RU queries

Example log shape:

```json
{
  "operation": "transferAccess.listForPrincipal",
  "container": "transferAccess",
  "partitionKey": "user:abc",
  "requestCharge": 4.8,
  "itemCount": 20,
  "hasContinuation": true,
  "elapsedMs": 18
}
```

Set budgets:

```text
Point read: ~1-5 RU depending on item size
Inbox page: target < 10-20 RU
Dashboard load: target < 20-50 RU
Admin reports: can be higher, but not on hot path
```

Exact RU depends on data size, indexing, consistency, and query shape.

## 19. Practical Container Design Checklist

Before creating a container, answer:

1. What is the primary access pattern?
2. What is the partition key?
3. What queries are single-partition?
4. What queries are cross-partition, and are they acceptable?
5. What is the expected item size?
6. Could any item grow indefinitely?
7. What fields are indexed?
8. What fields are excluded from indexing?
9. Are composite indexes required?
10. Is TTL required?
11. Are unique keys required?
12. What writes must be atomic?
13. Can atomic writes fit in one logical partition?
14. What projections are updated from this item?
15. Are projection writes synchronous, outbox-based, or change-feed-based?
16. What is the retry/idempotency strategy?
17. What is the migration/backfill strategy?
18. What is the RU budget?

## 20. Example: Transfer Modeling

Bad design:

```text
Container: transfers
Partition key: /tenantId

Queries:
- get by id across partitions
- list received by recipient across partitions
- scan active transfers for reminders
- scan all transfers for metrics
```

Better design:

```text
Container: transferWrite
Partition key: /transferId
Purpose: canonical transfer, events, outbox

Container: transferAccess
Partition key: /principalId
Purpose: sent/received inbox

Container: transferTenantIndex
Partition key: /tenantId
Purpose: tenant admin listing

Container: transferWork
Partition key: /bucketShard
Purpose: reminder and expiry work

Container: transferMetricsDaily
Partition key: /metricOwnerId
Purpose: dashboards
```

This is more containers, but each container has a job. It trades write complexity for predictable reads and cost.

## 21. What Cosmos Gives You Out Of The Box

Cosmos DB is valuable when you use its strengths:

- Globally distributed database service.
- Horizontal scaling through partitioning.
- Low-latency point reads and writes when modeled correctly.
- Request Unit model for predictable capacity planning.
- Automatic indexing by default.
- Custom indexing policies.
- Tunable consistency levels.
- Multi-region replication.
- Change feed for event-driven processing.
- TTL for automatic data lifecycle management.
- Serverless, provisioned, and autoscale throughput options.
- Integrated Azure security, networking, monitoring, and backup features.
- Multiple APIs, though this guide focuses on API for NoSQL.

Cosmos is not valuable if the application mostly does ad hoc cross-partition queries, relational joins, global scans, or expects SQL-style transactions.

## 22. Review Rubric

Use this rubric during design review.

### Green

- Hot reads are point reads or single-partition queries.
- Partition keys match access patterns.
- Large payloads are in Blob Storage.
- Read models exist for hot screens.
- Work queues avoid scans.
- Metrics are precomputed.
- Indexing policies are intentional.
- RU charges are logged.
- Cross-container updates are idempotent.
- Transaction boundaries are explicit.

### Yellow

- Some low-frequency cross-partition queries.
- Default indexing still used on moderate-write containers.
- Projection updates are synchronous dual writes.
- Tenant partitioning used before volume is known.

### Red

- `fetchAll()` in hot paths.
- Query by id instead of point read.
- `/tenantId` everywhere without analysis.
- `LOWER`, `CONTAINS`, and global `ORDER BY` on hot endpoints.
- Scheduler scans entire containers.
- Dashboard scans raw data.
- Raw PHI or large integration payloads stored in Cosmos.
- Assumption that cross-container writes are ACID.

## 23. Summary Rules

1. Start from access patterns, not entity diagrams.
2. Make hot `get by id` operations point reads.
3. Choose partition keys before writing code.
4. Duplicate data intentionally for read models.
5. Use deterministic ids for idempotency and deduplication.
6. Keep canonical writes and events in one logical partition when possible.
7. Use outbox/change feed for projections.
8. Do not scan containers for background work.
9. Do not compute dashboards from raw operational documents.
10. Index only what you query.
11. Exclude large payloads from indexing, or better, store them outside Cosmos.
12. Treat cross-partition queries as expensive unless proven otherwise.
13. Log RU charges for every data access method.
14. Keep Cosmos-specific code inside a data access layer.
15. Remember: Cosmos transactions are single-logical-partition transactions.

