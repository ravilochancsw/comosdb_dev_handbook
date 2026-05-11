# Implementing the New Cosmos Architecture

This document provides the exact code configurations and architectural mappings required to migrate the Aperio backend from a MongoDB-style collection structure to a highly optimized Cosmos DB architecture. 

It demonstrates how to consolidate the existing 8+ containers into 4 containers, apply strict indexing policies to reduce RU costs, and use the Repository Pattern to implement a Dual-Write strategy that eliminates expensive cross-partition "fan-out" queries.

---

## 1. The Core Concept: Models vs. Containers
**In Cosmos DB, models are NOT equated to containers.**

*   **Domain Model:** A JavaScript object that represents your business logic (e.g., `createTransfer(data)` in `transfer.model.js`). *These stay exactly the same. You don't need to rewrite your models.*
*   **Container:** The physical storage bucket in Cosmos DB where data lives. It is a billing and scaling boundary, not a schema boundary.
*   **Data Access Layer (Repository):** The bridge between the two. The Repository takes your Domain Model and decides *how* to physically store it in the Container to make queries fast.

---

## 2. Architecture Mapping Cheat Sheet
We are consolidating scattered containers into 4 optimized containers. Use this map to understand where data belongs.

| Old Container (MongoDB mindset) | New Container (Cosmos DB mindset) | EntityType (Discriminator) | Partition Key |
| :--- | :--- | :--- | :--- |
| `providerProfiles` | `TenantWorkspace` | `ProviderProfile` | `/tenantId` |
| `connectorInstances` | `TenantWorkspace` | `ConnectorInstance` | `/tenantId` |
| `handoffConfig` | `TenantWorkspace` | `HandoffConfig` | `/tenantId` |
| `tierConfig` | `TenantWorkspace` | `TierConfig` | `/tenantId` |
| `customReminderRequests` | `TenantWorkspace` | `CustomReminderRequest`| `/tenantId` |
| `transfers` | `Transfers` | `Transfer` | `/partitionKey` (Holds sender OR recipient tenant ID) |
| `connectorRuns` | `ConnectorLogs` | `ConnectorRun` | `/connectorId` |
| *(None - calculated on fly)* | `Aggregates` | `TenantMetrics` | `/partitionKey` |

---

## 3. The New Containers Configuration
This replaces your existing `backend/src/config/containers.js`. Notice the heavy use of `indexingPolicy`—this is how you save massive amounts of money on write operations.

```javascript
// backend/src/config/containers.js (REPLACEMENT)

export const NEW_APERIO_CONTAINERS = Object.freeze({
    tenantWorkspace: "prod-aperio-workspace",
    transfers:       "prod-aperio-transfers",
    connectorLogs:   "prod-aperio-connectorLogs",
    aggregates:      "prod-aperio-aggregates"
});

/**
 * Container Definitions with specific Partition Keys and Indexing Policies
 */
export const CONTAINER_DEFINITIONS = [
    {
        // 1. TENANT WORKSPACE
        // Consolidates: providerProfiles, connectorInstances, handoffConfig, customReminderRequests, tierConfig
        id: NEW_APERIO_CONTAINERS.tenantWorkspace,
        partitionKey: { paths: ["/tenantId"] },
        indexingPolicy: {
            indexingMode: "consistent",
            automatic: true,
            includedPaths: [{ path: "/*" }],
            excludedPaths: [
                // Exclude large config objects that we never use in a WHERE clause
                { path: "/config/credentials/?" },
                { path: "/runtime/cursor/?" }
            ]
        }
    },
    {
        // 2. TRANSFERS
        // Replaces: transfers. 
        // Uses the Dual-Write Materialized View pattern.
        id: NEW_APERIO_CONTAINERS.transfers,
        // Notice we do NOT use /tenantId. We use a generic /partitionKey
        // so it can hold either the sender's tenant ID or the recipient's tenant ID.
        partitionKey: { paths: ["/partitionKey"] },
        indexingPolicy: {
            indexingMode: "consistent",
            automatic: true,
            includedPaths: [
                { path: "/transferId/?" },
                { path: "/viewType/?" },
                { path: "/workflow/status/?" },
                { path: "/workflow/timestamps/?" },
                { path: "/systemMeta/search/?" }
            ],
            excludedPaths: [
                // MASSIVE RU SAVINGS: Do not index the actual payload!
                // We never query "WHERE message = 'Hello'", so don't pay to index it.
                { path: "/referralData/message/?" },
                { path: "/referralData/patientDetails/*" },
                { path: "/referralData/patientHistory/*" },
                { path: "/documents/files/*" }
            ],
            // Composite Index to make ORDER BY created date fast and cheap
            compositeIndexes: [
                [
                    { path: "/viewType", order: "ascending" },
                    { path: "/workflow/status", order: "ascending" },
                    { path: "/workflow/timestamps/createdAt", order: "descending" }
                ]
            ]
        }
    },
    {
        // 3. CONNECTOR LOGS
        // Replaces: connectorRuns
        id: NEW_APERIO_CONTAINERS.connectorLogs,
        partitionKey: { paths: ["/connectorId"] },
        // Automatically delete old logs after 30 days (2,592,000 seconds)
        defaultTtl: 2592000, 
        indexingPolicy: {
            indexingMode: "consistent",
            automatic: true,
            includedPaths: [
                { path: "/status/?" },
                { path: "/timestamps/createdAt/?" }
            ],
            excludedPaths: [
                { path: "/*" } // Exclude everything else by default for logs
            ]
        }
    },
    {
        // 4. AGGREGATES
        // New container for pre-calculated metrics
        id: NEW_APERIO_CONTAINERS.aggregates,
        partitionKey: { paths: ["/partitionKey"] },
        indexingPolicy: {
            indexingMode: "consistent",
            includedPaths: [{ path: "/*" }],
            excludedPaths: []
        }
    }
];

export async function createContainers(database) {
    for (const def of CONTAINER_DEFINITIONS) {
        await database.containers.createIfNotExists(def);
        console.log(`Ensured container: ${def.id}`);
    }
}
```

---

## 4. The Repository Pattern in Action (Solving the Fan-Out)
This demonstrates how to implement the **Dual-Write Materialized View** pattern. By writing a copy of the transfer to the recipient's partition, the Recipient Dashboard loads instantly without scanning the whole database.

```javascript
// backend/src/repositories/TransferRepository.js

import { getAperioContainer } from '../config/cosmosClient.js';
import { NEW_APERIO_CONTAINERS } from '../config/containers.js';

export class TransferRepository {
    static get container() {
        return getAperioContainer(NEW_APERIO_CONTAINERS.transfers);
    }

    /**
     * CREATE / SAVE A TRANSFER (The Dual-Write Pattern)
     */
    static async saveTransfer(domainModel) {
        // In Cosmos, transactions are scoped to a single logical partition.
        // We write the Sender document, then write the Recipient document.
        
        const senderTenantId = domainModel.tenantId; // The authoritative owner
        const recipientTenantId = domainModel.actors.recipient.recipientTenantId;

        // 1. Create the Sender's View (Authoritative)
        const senderDoc = {
            ...domainModel,
            id: `${domainModel.id}_sender`, // Physical ID in Cosmos
            transferId: domainModel.id,     // Logical business ID
            partitionKey: senderTenantId,   // Partitioned under the Sender!
            viewType: 'sender',
            isAuthoritative: true
        };

        // Upsert the sender document
        await this.container.items.upsert(senderDoc);

        // 2. Create the Recipient's View (Materialized View)
        // If the recipient is registered in our system, write a copy to THEIR partition!
        if (recipientTenantId) {
            const recipientDoc = {
                ...domainModel,
                id: `${domainModel.id}_recipient`,
                transferId: domainModel.id,
                partitionKey: recipientTenantId, // Partitioned under the Recipient!
                viewType: 'recipient',
                isAuthoritative: false
            };
            
            await this.container.items.upsert(recipientDoc);
        }

        return domainModel;
    }

    /**
     * POINT READ (The Holy Grail of Cosmos Performance - ~1 RU cost)
     */
    static async getTransferById(transferId, userTenantId, role = 'sender') {
        // Because we know the user's tenant ID, we do a Point Read!
        // No more SELECT * FROM c WHERE id = @id
        const physicalId = `${transferId}_${role}`;
        
        try {
            // This is lightning fast and uses almost no Request Units.
            const { resource } = await this.container.item(physicalId, userTenantId).read();
            return resource;
        } catch (err) {
            if (err.code === 404) return null;
            throw err;
        }
    }

    /**
     * LIST TRANSFERS (Single-Partition Queries! No more Fan-out)
     */
    static async listTransfers(userTenantId, role) {
        // If role is 'recipient', it STILL only hits ONE physical partition 
        // because of our dual-write!
        const querySpec = {
            query: `
                SELECT * FROM c 
                WHERE c.partitionKey = @tenantId 
                  AND c.viewType = @role 
                  AND c.entityType = 'transfer'
                ORDER BY c.workflow.timestamps.createdAt DESC
            `,
            parameters: [
                { name: '@tenantId', value: userTenantId },
                { name: '@role', value: role } // 'sender' or 'recipient'
            ]
        };

        const { resources } = await this.container.items.query(querySpec).fetchAll();
        return resources;
    }
}
```

---

## 5. The Workflow Change for Developers
1. **Stop querying directly in Services:** Developers should no longer call `container.items.query(...)` directly inside `transfer.service.js` or `provider.service.js`.
2. **Implement Repositories:** Create a `TransferRepository`, `WorkspaceRepository`, and `LogsRepository`.
3. **Services call Repositories:** The Services continue to use the existing Domain Models (from `src/models/`), but pass them into the Repositories to be saved.
4. **Repositories handle Cosmos specifics:** The Repositories format the JSON to include `entityType` and the correct `partitionKey`, apply the dual-write logic if necessary, and then execute the native Cosmos SDK calls (like `upsert` or `.item().read()`).