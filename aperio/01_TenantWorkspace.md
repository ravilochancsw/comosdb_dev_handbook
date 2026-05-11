# 1. TenantWorkspace Container

*   **Container Name:** `TenantWorkspace` (Replaces `providerProfiles`, `connectorInstances`, `handoffConfig`, `tierConfig`)
*   **Partition Key:** `/tenantId`
*   **Indexing Policy:** Default (or exclude specific large config fields)

Because multiple entity types live here, **every document MUST have an `entityType` field** so you can query them specifically (`SELECT * FROM c WHERE c.tenantId = @tid AND c.entityType = 'ProviderProfile'`).

## Schema Examples

### Entity Type: ProviderProfile
```json
{
  "id": "profile_12345",
  "tenantId": "tenant_abc",
  "entityType": "ProviderProfile",
  "userId": "user_789",
  "email": "dr.smith@example.com",
  "npiNumber": "1234567890",
  "firstName": "John",
  "lastName": "Smith",
  "practiceAddress": {
      "street": "123 Medical Way",
      "city": "Austin",
      "state": "TX"
  },
  "hipaaAttestationAccepted": true,
  "createdAt": "2026-05-11T10:00:00Z"
}
```

### Entity Type: ConnectorInstance
```json
{
  "id": "conn_999",
  "tenantId": "tenant_abc",
  "entityType": "ConnectorInstance",
  "connectorType": "epic_fhir",
  "state": "active",
  "config": {
     "clientId": "xxx-yyy",
     "fhirBaseUrl": "https://epic.example.com/api/FHIR/R4"
  },
  "runtime": {
     "triggerMode": "webhook"
  },
  "createdAt": "2026-05-11T10:00:00Z"
}
```

### Entity Type: TierConfig
```json
{
  "id": "tier_config_tenant_abc",
  "tenantId": "tenant_abc",
  "entityType": "TierConfig",
  "tier": "enterprise",
  "features": {
      "customBranding": true,
      "maxTransfers": 10000
  }
}
```