# 3. ConnectorLogs Container

*   **Container Name:** `ConnectorLogs`
*   **Partition Key:** `/connectorId`
*   **Time-To-Live (TTL):** Enabled (e.g., 2592000 seconds / 30 days)
*   **Indexing Policy:** Exclude all paths except `id`, `connectorId`, `status`, and `timestamps/createdAt`.

## Why a separate container?
The `connectorRuns` data is essentially a time-series log of background jobs. It grows rapidly and is rarely queried after a few days. If you keep this in the `TenantWorkspace` container, it will bloat the tenant's logical partition size (20GB max limit) and consume RUs needed for critical user operations.

By partitioning by `/connectorId`, we isolate the heavy write traffic.

## Schema Example

### Entity Type: ConnectorRun
```json
{
  "id": "run_abc123",
  "connectorId": "conn_999",
  "entityType": "ConnectorRun",
  "tenantId": "tenant_abc",
  "status": "success",
  "externalId": "fhir_referral_888",
  "transferId": "tx_uuid_1234",
  "timestamps": {
    "createdAt": "2026-05-11T10:05:00Z",
    "updatedAt": "2026-05-11T10:05:02Z"
  },
  "metrics": {
    "durationMs": 1500
  },
  "ttl": 2592000  // Auto-deletes after 30 days
}
```