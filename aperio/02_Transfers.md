# 2. Transfers Container (The Dual-Write Pattern)

*   **Container Name:** `Transfers`
*   **Partition Key:** `/partitionKey` (Do NOT name it `/tenantId`, name the physical partition key property `/partitionKey` so it can hold either a sender's tenant ID or a recipient's tenant ID).
*   **Indexing Policy:** Exclude `/referralData/message/?` and array fields from indexing to save write RUs. Only index status, timestamps, and search fields.

## The Dual-Write Strategy
When Provider A (Tenant A) sends a transfer to Provider B (Tenant B), the `TransferRepository` writes **two distinct documents** to this container. Both share the same physical `transferId`, but they have different `id` and `partitionKey` values.

### Document 1: The Sender's View (Authoritative)
```json
{
  "id": "tx_uuid_1234_sender", 
  "partitionKey": "tenant_A",   // Partitioned under the Sender
  "transferId": "tx_uuid_1234",
  "viewType": "sender",
  "entityType": "Transfer",
  "isAuthoritative": true,
  
  "actors": {
    "sender": { "tenantId": "tenant_A", "userId": "user_1" },
    "recipient": { "tenantId": "tenant_B", "email": "dr.b@example.com" }
  },
  "workflow": {
    "status": "sent",
    "timestamps": { "createdAt": "2026-05-11T10:00:00Z" }
  },
  "referralData": { ... },
  "documents": { ... }
}
```

### Document 2: The Recipient's View (Materialized View)
```json
{
  "id": "tx_uuid_1234_recipient", 
  "partitionKey": "tenant_B",   // Partitioned under the Recipient!
  "transferId": "tx_uuid_1234",
  "viewType": "recipient",
  "entityType": "Transfer",
  "isAuthoritative": false,
  
  "actors": {
    "sender": { "tenantId": "tenant_A", "userId": "user_1" },
    "recipient": { "tenantId": "tenant_B", "email": "dr.b@example.com" }
  },
  "workflow": {
    "status": "sent",
    "timestamps": { "createdAt": "2026-05-11T10:00:00Z" }
  },
  "referralData": { ... },
  "documents": { ... }
}
```

### How Queries Work Now:
1. **Sender lists sent transfers:** `SELECT * FROM c WHERE c.partitionKey = 'tenant_A' AND c.viewType = 'sender'` (Single Partition Query! Fast.)
2. **Recipient lists received transfers:** `SELECT * FROM c WHERE c.partitionKey = 'tenant_B' AND c.viewType = 'recipient'` (Single Partition Query! Fast. No more fan-out).
3. **Recipient accepts transfer:** `TransferRepository` updates BOTH documents. It point-reads the recipient view to validate, updates the sender view (authoritative), and overwrites the recipient view.