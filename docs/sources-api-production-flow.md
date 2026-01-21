# Sources API Provider Creation Flow

This document describes provider creation using the Sources API in the on-prem deployment.

## Architecture Overview

```
┌─────────────────┐
│   User/Script   │
└────────┬────────┘
         │ HTTP POST
         ▼
┌─────────────────────┐
│  Koku API           │ ← /api/cost-management/v1/sources
│  (Sources API)      │
└──────────┬──────────┘
         │ AdminSourcesSerializer.create()
         │ ProviderBuilder.create_provider_from_source()
         ▼
┌─────────────────────┐
│  Tenant Provisioning│
│  - Create schema    │
│  - Run migrations   │
│  - Create provider  │
└─────────────────────┘
```

In the on-prem deployment, source and provider creation happens synchronously in a single API call.

## Components

| Component | Template | Purpose |
|-----------|----------|---------|
| Koku API | `cost-onprem/templates/cost-management/api/deployment.yaml` | HTTP endpoints for sources and cost management |
| Envoy Ingress | `cost-onprem/templates/ingress/` | JWT authentication and routing |

## API Endpoint

Sources API is served by Koku API under `/api/cost-management/v1/`:

```bash
# Get the route URL
COST_API_URL=$(oc get route cost-management-api -n cost-onprem -o jsonpath='{.spec.host}')
echo "Sources API: https://$COST_API_URL/api/cost-management/v1/sources"
```

## Flow Details

1. **User creates source via HTTP POST**
   ```
   POST /api/cost-management/v1/sources
   {
     "name": "OCP Test Provider",
     "source_type": "OCP",
     "authentication": {"credentials": {"cluster_id": "test-cluster-123"}},
     "billing_source": {"data_source": {"bucket": "koku-bucket"}}
   }
   ```

2. **Koku API processes request synchronously**
   - `AdminSourcesSerializer.validate()` - validates source data
   - `Sources.objects.create()` - creates Source record
   - `ProviderBuilder.create_provider_from_source()` - creates Provider
   - Links source to provider

3. **Provider available immediately**
   - No Kafka messaging required
   - No separate listener component needed

## Source Deletion

When a source is deleted:

1. **DELETE /api/cost-management/v1/sources/{id}**
2. **Koku deletes Provider and Source**
3. **Koku publishes `Application.destroy` to Kafka** (for ROS cleanup)
4. **ROS Housekeeper cleans up ROS data**

## Testing

```bash
# Run E2E test
./scripts/cost-mgmt-ocp-dataflow.sh --namespace cost-onprem
```

## Comparison: On-Prem vs SaaS

| Aspect | On-Prem (Koku API) | SaaS (sources-api-go) |
|--------|--------------------|-----------------------|
| Source Creation | Synchronous HTTP | Async via Kafka |
| Provider Creation | Same request | Sources Listener |
| Kafka Required | Only for ROS cleanup | Full event pipeline |
| Separate Services | Koku only | sources-api-go + sources-listener |
