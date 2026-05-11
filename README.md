# The Cosmos DB Developer Handbook
**A Paradigm Shift from Relational and Traditional NoSQL Databases**

When developers transition from SQL (PostgreSQL, MySQL) or traditional document stores (MongoDB) to Azure Cosmos DB, the most common mistake is applying relational concepts to a horizontally scalable distributed database. This document serves as a comprehensive guide to understanding, modeling, querying, and managing data in Cosmos DB the *right* way.

---

## 1. Out-of-the-Box Benefits of Cosmos DB

Cosmos DB is Microsoft's globally distributed, multi-model database service. It is designed for elastically scalable applications.

*   **Turnkey Global Distribution:** Replicate data across any number of Azure regions with a single click.
*   **Guaranteed Single-Digit Millisecond Latency:** SLAs guarantee <10ms latency for reads and writes at the 99th percentile anywhere in the world.
*   **Elastic Scalability:** Scale storage and throughput (Request Units - RUs) instantly without downtime.
*   **Five 9s Availability:** Enterprise-grade 99.999% availability SLAs.
*   **Automatic Indexing:** By default, every property of every item is indexed automatically without requiring schema or index management.
*   **Change Feed:** A persistent, order-guaranteed log of all changes to a container, making event-driven architectures and materialized views natively supported.

---

## 2. The Golden Rules of Data Modeling in Cosmos DB

In SQL, you normalize data to avoid duplication. In Cosmos DB, **you optimize for your read/write patterns.** 

### Rule #1: Containers are NOT Tables (Single-Container Architecture)
A container in Cosmos DB is a **billing and physical scaling boundary**, not a schema boundary. 
*   **Anti-Pattern:** Creating a container for `Users`, a container for `Posts`, and a container for `Comments`. (This multiplies your minimum RU/s costs and prevents cross-entity transactions).
*   **Best Practice:** Put multiple entities into a single container if they share a partition key and are queried together. Differentiate them using a `type` or `entityType` discriminator field.

### Rule #2: Embed vs. Reference
*   **Embed (Denormalize):** If data is updated together, queried together, and has a 1-to-few relationship (e.g., an Order and its OrderItems).
*   **Reference (Normalize):** If the related data is large, changes frequently independently, or has a 1-to-many/many-to-many relationship (e.g., a User and their 10,000 Transfers).

### Rule #3: Optimize for the Critical Path
Design your model around the most frequent or most expensive queries. If your dashboard requires aggregating 10,000 records, you do not calculate that on the fly. You calculate it at write-time and store a pre-aggregated document.

---

## 3. Containers and Naming Conventions

*   **Naming:** Name containers based on the **aggregate root** or **functional boundary**, not the specific entity. 
    *   *Bad:* `ProviderProfiles`, `CustomReminderRequests`
    *   *Good:* `TenantWorkspace`, `GlobalDirectory`, `SystemLogs`
*   **When to create a new container:**
    1.  They require different Partition Keys.
    2.  They have vastly different throughput requirements (e.g., highly active telemetry logs vs. static reference data).
    3.  Different access control requirements.

---

## 4. Partitioning (The Most Critical Concept)

Cosmos DB scales horizontally by dividing your data into physical partitions under the hood. It maps your data to these physical servers using a **Partition Key**.

### Logical vs. Physical Partitions
*   **Logical Partition:** All documents with the same Partition Key value. (e.g., all docs where `tenantId = "tenant_A"`). **Max size is 20 GB.**
*   **Physical Partition:** Actual servers managed by Azure. One physical partition can hold many logical partitions.

### Choosing a Partition Key
*   **High Cardinality:** The key should have many distinct values (hundreds or thousands). `/tenantId` is good. `/isActive` (boolean) is terrible.
*   **Even Distribution:** Avoid "Hot Partitions" where one value receives 90% of the read/write traffic.
*   **Align with Queries:** Your most frequent queries **must** include the partition key in the `WHERE` clause.

### Synthetic Partition Keys
If a single property doesn't provide good distribution, you create a synthetic key.
1.  **Concatenation:** `partitionKey = "{tenantId}_{date}"` (e.g., `tenant_A_2026-05`). This prevents a massive tenant from hitting the 20GB limit by chunking their data by month.
2.  **Hashing / Random Suffix:** If you have write-heavy telemetry, you might use `partitionKey = "{deviceId}_{random(1-10)}"`.

---

## 5. Data Access and Query Patterns

The cost of operations in Cosmos DB is measured in **Request Units (RUs)**.

### Pattern 1: Point Reads (The Holy Grail)
A Point Read looks up a single document by its `id` and `partitionKey`. It bypasses the query engine completely.
*   **Cost:** ~1 RU for a 1KB document.
*   **Code:** `container.item(id, partitionKey).read()`
*   **Rule:** ALWAYS use this if you know both values. Never use `SELECT * FROM c WHERE c.id = @id`.

### Pattern 2: Single-Partition Query (Good)
A query where the `partitionKey` is provided in the `WHERE` clause. Cosmos DB routes the query to exactly one physical server.
*   **Code:** `SELECT * FROM c WHERE c.tenantId = 'tenant_A' AND c.entityType = 'Transfer'`

### Pattern 3: Cross-Partition Scan / Fan-out (The Enemy)
A query that does *not* include the partition key. Cosmos DB must send this query to **every physical server** in the cluster, wait for them all to respond, and merge the results.
*   **Impact:** Destroys performance, consumes massive RUs, and does not scale as data grows.
*   **Solution:** Use the **Materialized View Pattern** (Dual-Write or Change Feed) to project data into a new container partitioned by the attribute you need to search by.

---

## 6. Indexing Strategy

Cosmos DB uses a unique **Inverted Index** tree structure. By default, it indexes *every property* of *every document*. 

### Why customize indexing?
While automatic indexing is great for read-heavy workloads, maintaining the index costs RUs on every write. If you have large JSON payloads (like patient medical history) that you never query with a `WHERE` clause, you are wasting money indexing them.

### Best Practices
1.  **Include / Exclude Paths:** Explicitly exclude large text fields, raw payloads, or deeply nested arrays.
    ```json
    {
      "indexingMode": "consistent",
      "includedPaths": [{"path": "/*"}],
      "excludedPaths": [
        {"path": "/referralData/message/?"},
        {"path": "/documents/files/*"}
      ]
    }
    ```
2.  **Composite Indexes:** If your queries frequently have `ORDER BY propertyA ASC, propertyB DESC` or use multiple equality filters, create a composite index. Otherwise, the query might fail or consume excessive RUs.

---

## 7. ACID Transactions and Multi-Document Operations

Does Cosmos DB support ACID transactions? **Yes, but with a strict scope constraint.**

### The Scope: Single Logical Partition
Transactions are **strictly scoped to a single logical partition within a single container**. You cannot perform an ACID transaction across different partition keys, or across different containers.

### Method 1: TransactionalBatch (SDK Level)
The modern Node.js SDK allows you to bundle multiple operations (Create, Replace, Delete) into a single RPC call. If one fails, the entire batch rolls back.
```javascript
// Example: Updating a transfer and a user profile simultaneously
const batch = container.items.batch(partitionKey); // MUST share the same PK
batch.replace(transfer, { id: 'tx_123' });
batch.create(auditLog);
const response = await batch.execute();
```

### Method 2: Stored Procedures (Server-Side)
Written in JavaScript and executed on the database server. They run within the context of a single partition key and guarantee ACID compliance. Often used for complex bulk operations or ensuring concurrency control (Optimistic Concurrency Control using ETags is also natively supported).

---

## 8. Advanced Patterns: Materialized Views & Change Feed

When a system requires data to be queried in multiple ways (e.g., finding a transfer by `senderTenantId` AND by `recipientTenantId`), you use the **Change Feed**.

1.  **The Change Feed** is an API that listens to a container for any inserts or updates.
2.  You write an Azure Function (or background worker) that listens to the `Transfers` container (partitioned by Sender).
3.  When a transfer is created, the Change Feed processor catches it, and writes a *copy* of that transfer to a `ReceivedTransfers` container (partitioned by Recipient).
4.  This creates a **Materialized View**. Writes are eventually consistent, but reads are blazing fast point-reads or single-partition queries.

---

## References & Required Reading
*   [Data modeling in Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/modeling-data)
*   [Partitioning and horizontal scaling](https://learn.microsoft.com/en-us/azure/cosmos-db/partitioning-overview)
*   [Request Units (RUs) explained](https://learn.microsoft.com/en-us/azure/cosmos-db/request-units)
*   [Transactional Batch operations](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/transactional-batch)
*   [Materialized View pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view)
