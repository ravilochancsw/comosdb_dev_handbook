# Cosmos DB Best Practices: Models vs. Containers & Data Access

To answer your fundamental questions coming from a Node.js/MongoDB background: **No, in Cosmos DB, you do NOT want a 1:1 mapping between a Domain Model and a Container.**

### The MongoDB/SQL Mindset (Anti-Pattern in Cosmos)
In MongoDB or SQL, the physical storage boundary (Collection/Table) maps exactly to your logical entity (Model). 
*   `Users` Collection -> `UserModel`
*   `Transfers` Collection -> `TransferModel`
*   `ProviderProfiles` Collection -> `ProviderProfileModel`

### The Cosmos DB Mindset
In Cosmos DB, a **Container** is a *billing and performance boundary*, not a schema boundary. Cosmos DB is schema-agnostic. 
The best practice is **Single-Container Multi-Entity Architecture**. You should group different entity types into the same container if they share the same **Partition Key** and are accessed together.

#### Why group entities?
1. **Cost:** You pay minimum Request Units (RUs) per container (typically 400 RU/s). If you have 10 containers, you pay for 4,000 RU/s minimum, even if 9 of them are mostly idle. Consolidating into 2 containers means you pay for 800 RU/s and the RUs are shared across all your entities.
2. **Transactions:** Cosmos DB only supports ACID transactions (Stored Procedures/Transactional Batch) *within a single logical partition in a single container*. If you need to update a `ProviderProfile` and a `Transfer` atomically, they MUST be in the same container and share the same partition key.

### Suggested Code Structure: The Repository Pattern
Because the physical storage (Container) is decoupled from the logical entity (Model), you need a layer in your code to translate between them.

Do not put Cosmos DB logic directly in your Services. Instead, use a **Data Access Layer (Repository Pattern)**.

*   **Models:** Plain JavaScript objects/classes representing business data (no DB logic).
*   **Repositories:** Classes that handle writing/reading Models to/from Cosmos DB.
*   **Services:** Business logic that calls Repositories.

**Example Flow:**
`TransferService` -> calls `TransferRepository.save(transfer)`
Inside `TransferRepository`:
```javascript
// The repository knows about the Cosmos DB dual-write strategy, the service doesn't.
async function save(transfer) {
    // 1. Write the Sender's view (Partition: SenderTenantId)
    await container.items.create({ ...transfer, partitionKey: transfer.senderTenantId, viewType: 'sender' });
    
    // 2. Write the Recipient's view (Partition: RecipientTenantId)
    await container.items.create({ ...transfer, partitionKey: transfer.recipientTenantId, viewType: 'recipient' });
}
```

---

## Redesigning the Containers for Aperio

Based on your access patterns, we need to eliminate cross-partition queries and pre-aggregate data. We will reduce your scattered containers down to **Four Highly Optimized Containers**:

### 1. `TenantWorkspace` Container
*   **Partition Key:** `/tenantId`
*   **Purpose:** Holds all tenant-specific configuration and profiles.
*   **Entities Stored:** `ProviderProfile`, `ConnectorInstance`, `HandoffConfig`, `TierConfig`.
*   **Why:** These are strictly queried by `tenantId`. Grouping them saves immense costs and allows you to fetch a tenant's entire workspace state in a single query.

### 2. `Transfers` Container
*   **Partition Key:** `/partitionKey` (Notice it's generic, not hardcoded to tenantId)
*   **Purpose:** Holds the actual referral data.
*   **Strategy: Dual-Write / Materialized View.** 
    *   When Tenant A sends to Tenant B, the repository writes **TWO** documents to this container.
    *   Doc 1: `partitionKey = "TenantA"`, `viewType = "sender"`
    *   Doc 2: `partitionKey = "TenantB"`, `viewType = "recipient"`
*   **Why:** This completely eliminates the fatal fan-out query. When Tenant B opens their dashboard, they do a single-partition query on their own `TenantB` partition. It's lightning fast and cheap.

### 3. `ConnectorLogs` Container
*   **Partition Key:** `/connectorId`
*   **Purpose:** Holds `ConnectorRun` records.
*   **Why:** Logs are high-volume. If we put them in `TenantWorkspace`, they would cause "Hot Partitions" (exceeding the 20GB limit per partition key). Partitioning by `connectorId` isolates this heavy write traffic. Set a **Time-To-Live (TTL)** on this container (e.g., 30 days) so Cosmos auto-deletes old logs for free.

### 4. `Aggregates` Container
*   **Partition Key:** `/partitionKey`
*   **Purpose:** Holds pre-calculated metrics for dashboards.
*   **Why:** Calculating metrics on the fly by scanning thousands of transfers will crash your Node server. Instead, when a transfer status changes, update a single `Metrics` document here. The dashboard does 1 Point Read to get all chart data.

*Check the other JSON/MD files in this directory for the exact schema shapes of these new containers.*
