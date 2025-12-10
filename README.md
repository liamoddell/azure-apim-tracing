# Azure APIM Distributed Tracing with Grafana Cloud

End-to-end distributed tracing for Azure API Management using OpenTelemetry and Grafana Alloy.

## Overview

This APIM policy enables complete distributed traces showing requests flowing through your API gateway to backend services, with automatic integration into Grafana Cloud's Application Observability and Service Graph.

## What This Achieves

**The Problem:**
Without distributed tracing, you see APIM and backend as separate systems. When requests are slow or failing, you can't see the complete journey through your architecture.

**This Solution:**
- ✅ **True distributed tracing** - Parent-child span relationships between APIM and backends
- ✅ **W3C Trace Context** - Standards-based propagation via `traceparent` header
- ✅ **Service Graph** - Automatic topology visualization showing service dependencies
- ✅ **Application Observability** - Services appear automatically with RED metrics
- ✅ **Vendor neutral** - Works with any OTLP collector (Alloy, Jaeger, Tempo, etc.)
- ✅ **Real-time** - See traces immediately, not hours later
- ✅ **Cost effective** - ~$40-50/month for 10M requests (Grafana Cloud free tier covers most use cases)

**vs. Alternatives:**
- **EventHub logging**: Just logs, no trace relationships, higher cost, delayed visibility
- **Application Insights**: Azure-only, expensive at scale, limited W3C support
- **Log Analytics**: Not real-time, no service graph, designed for audit not tracing

**Use this if:**
- You have microservices with OpenTelemetry instrumentation
- You want true distributed tracing with service graphs
- You use Grafana Cloud, Tempo, or Jaeger
- You need vendor neutrality

**Verified with:** Azure APIM Developer SKU, Azure Functions, Grafana Cloud Tempo

## Implementation Approaches

### Quick Start (Testing/Single API)

1. **Download:** `apim-policy.xml`
2. **Customise:**
   - Line 109: OTLP endpoint URL
   - Line 160: Backend service name (must match backend's `service.name`)
   - Line 131: Environment (production/staging/dev)
3. **Apply:** APIM → APIs → Your API → Policies → Paste → Save

### Production (Policy Fragments)

For multi-API deployments, use fragments for centralised management:

1. **Create Named Value:** `alloy-otlp-endpoint` = `https://your-alloy.azurecontainerapps.io`
2. **Create Fragment:** Upload `apim-policy-fragment.xml` with ID `otlp-tracing`
3. **Apply to APIs:**
   ```xml
   <outbound>
       <include-fragment fragment-id="otlp-tracing" />
       <base />
   </outbound>
   ```

**Benefits:** Update once, applies everywhere. No per-API configuration.

## Full Setup (New Users)

### Prerequisites

- Azure APIM (Developer SKU minimum)
- Grafana Cloud account ([free tier](https://grafana.com/auth/sign-up/create-user))
- Backend service with OpenTelemetry instrumentation

### Step 1: Get Grafana Cloud Credentials

1. Login to Grafana Cloud → Connections → Add new connection → Hosted OTLP
2. Copy these values:
   ```bash
   OTLP Endpoint: https://otlp-gateway-prod-xx-x.grafana.net/otlp
   Instance ID: your-instance-id
   API Key: glc_...
   ```

### Step 2: Deploy Grafana Alloy

Using Azure Container Apps:

```bash
# Set variables
export RG="your-resource-group"
export LOCATION="uksouth"

# Create environment
az containerapp env create --name alloy-env --resource-group $RG --location $LOCATION

# Deploy Alloy
az containerapp create \
  --name grafana-alloy --resource-group $RG --environment alloy-env \
  --image grafana/alloy:latest --target-port 4318 --ingress external \
  --cpu 0.5 --memory 1Gi --min-replicas 1 --max-replicas 3 \
  --secrets \
    grafana-cloud-otlp-endpoint="https://otlp-gateway-prod-xx-x.grafana.net/otlp" \
    grafana-cloud-instance-id="your-instance-id" \
    grafana-cloud-api-key="glc_..." \
  --env-vars \
    GRAFANA_CLOUD_OTLP_ENDPOINT=secretref:grafana-cloud-otlp-endpoint \
    GRAFANA_CLOUD_INSTANCE_ID=secretref:grafana-cloud-instance-id \
    GRAFANA_CLOUD_API_KEY=secretref:grafana-cloud-api-key

# Get Alloy URL
az containerapp show --name grafana-alloy --resource-group $RG \
  --query "properties.configuration.ingress.fqdn" -o tsv
```

See `alloy-config.alloy` for the configuration (automatically embedded in Container App).

### Step 3: Configure APIM Policy

1. Download `apim-policy.xml`
2. Customize these 3 values:

   **Line 80** - Alloy endpoint:
   ```xml
   <set-url>https://your-alloy.azurecontainerapps.io/v1/traces</set-url>
   ```

   **Line 160** - Backend service name:
   ```csharp
   new JProperty("value", new JObject(new JProperty("stringValue", "your-backend-service")))
   ```
   ⚠️ Must exactly match your backend's `service.name` resource attribute

   **Line 217** - Environment:
   ```csharp
   new JProperty("value", new JObject(new JProperty("stringValue", "production")))
   ```

3. Apply via Azure Portal or CLI:
   ```bash
   az rest --method put \
     --url "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG/providers/Microsoft.ApiManagement/service/YOUR-APIM/apis/YOUR-API/policies/policy?api-version=2024-05-01" \
     --body "$(jq -Rs '{properties: {format: "rawxml", value: .}}' apim-policy.xml)" \
     --headers "Content-Type=application/json"
   ```

### Step 4: Configure Backend

Ensure your backend:

1. Exports traces to same Alloy endpoint
2. Propagates the `traceparent` header (automatic with most OTEL SDKs)
3. Sets these resource attributes:
   ```javascript
   {
     'service.name': 'your-backend-service',      // Must match APIM policy!
     'deployment.environment': 'production',       // Required
     'service.version': '1.0.0',
     'service.namespace': 'your-namespace'
   }
   ```

### Step 5: Test

The example deployment includes a realistic multi-service architecture demonstrating distributed tracing:

**Services:**
- **Orders Service:** Creates orders, orchestrates inventory and payment checks
- **Inventory Service:** Checks product availability
- **Payment Service:** Processes payments (90% success rate for testing failures)
- **Echo Service:** Simple test endpoint

**Test distributed tracing:**
```bash
# Create an order (generates 4-service trace)
curl -X POST https://your-apim.azure-api.net/api/orders \
  -H 'Content-Type: application/json' \
  -d '{"productId":"LAPTOP-001","quantity":2}'

# Check inventory
curl https://your-apim.azure-api.net/api/inventory/LAPTOP-001

# Use the test script
./test-tracing.sh https://your-apim.azure-api.net
```

**Verify in Grafana Cloud:**
- **Tempo → Service Graph:** `apim-gateway` → `backend-service` with service dependencies
- **Application Observability:** Both services with RED metrics
- **Trace view:** Complete trace showing:
  ```
  apim-gateway (POST /orders)
    └─> backend-service (CreateOrder)
        ├─> backend-service (CheckInventory)
        └─> backend-service (ProcessPayment)
  ```

## Architecture

```
Client → [APIM Gateway] → [Backend Service]
              ↓                    ↓
         Emit OTLP Span       Emit OTLP Span
              ↓                    ↓
              └──────→ [Alloy] ←───┘
                          ↓
                  [Grafana Cloud Tempo]
```

**Trace flow:**
1. APIM generates/propagates `traceparent` header (W3C Trace Context)
2. APIM emits span to Alloy with OTLP
3. Backend receives `traceparent`, continues trace
4. Backend emits span to Alloy
5. Alloy forwards to Grafana Cloud
6. Tempo creates complete trace with parent-child relationships

## Configuration Reference

### Critical Resource Attributes

The policy sets these resource-level attributes (required for Application Observability):

| Attribute | Purpose | Value |
|-----------|---------|-------|
| `service.name` | Service identifier | `apim-gateway` |
| `deployment.environment` | **Required for metrics** | Your environment name |
| `cloud.region` | Regional dimension | `context.Deployment.Region` |
| `service.version` | Service version | `1.0.0` |
| `service.namespace` | Logical grouping | `azure-apim` |

### Service Graph Requirements

For services to connect in the Service Graph:

```
APIM Policy (line 160):           Backend Config:
peer.service = "backend-api"   →  service.name = "backend-api"
                                  ✅ MUST MATCH EXACTLY
```

The `peer.service` span attribute tells Tempo which service APIM is calling. This must match the backend's `service.name` resource attribute (case-sensitive).

## Troubleshooting

### Traces Not Appearing

Check Alloy is receiving traces:
```bash
az containerapp logs show --name grafana-alloy --resource-group $RG --tail 50
```

Look for errors or connection issues to Grafana Cloud.

### Service Not in Application Observability

**Cause:** Missing `deployment.environment` resource attribute

**Fix:** Verify line 217 in the policy sets this value. This attribute is required by Tempo's metrics generator to create metrics that feed Application Observability.

### Service Graph Not Connecting Services

**Causes:**
1. `peer.service` (line 160) doesn't match backend's `service.name`
2. Backend not setting `service.name` resource attribute
3. `traceparent` header not being propagated

**Fix:**
1. Check backend's OpenTelemetry configuration for `service.name`
2. Update APIM policy line 160 to match exactly
3. Verify in trace JSON: APIM `spanId` should equal backend `parentSpanId`

### Only Some Traces Appearing

**Cause:** Grafana Cloud tail sampling enabled

**Fix:** Temporarily disable tail sampling in Grafana Cloud Tempo settings for testing.

## Advanced

### Custom Span Attributes

Add custom attributes in the Liquid template (line 148+):

```json
{ "key": "custom.attribute", "value": { "stringValue": "{{context.Variables.my_value}}" } }
```

### Environment-Based Configuration

Use APIM Named Values (recommended for production):

1. Create Named Value: `alloy-endpoint`
2. Reference in fragment: `<set-url>{{alloy-endpoint}}/v1/traces</set-url>`

### Multi-Region Deployment

For multi-region APIM:
1. Deploy Alloy in each region
2. Use region-specific Named Values
3. `cloud.region` automatically captures deployment region from `context.Deployment.Region`

### Backend Service Requirements

For complete distributed tracing, ensure backends:

1. **Propagate traceparent header** (automatic with most OTEL SDKs)
2. **Set resource attributes:**
   ```csharp
   .ConfigureResource(resource => resource
       .AddService("backend-service", "1.0.0")
       .AddAttributes(new Dictionary<string, object>
       {
           ["deployment.environment"] = "production"  // Required
       }))
   ```
3. **Export to same Alloy endpoint**

The included C# Function App demonstrates this with:
- OpenTelemetry SDK with OTLP exporter
- .NET ActivitySource for custom spans
- HTTP client instrumentation for service-to-service calls

## Performance & Cost

**Performance Impact:**
- Policy execution: ~5ms per request (Liquid templates)
- Trace emission: Fire-and-forget, non-blocking
- API latency: < 1% increase
- CPU overhead: < 3%

**Azure Costs (UK South):**
- Container Apps (0.5 vCPU, 1Gi): ~£35/month
- APIM Developer SKU: ~£40/month
- Function App Consumption: Pay-per-use
- Egress: ~£5-10/month

**Grafana Cloud:**
- Free tier: 50GB traces/month
- ~200k-500k requests per GB depending on span size
- 10M requests/month ≈ 20-50GB ≈ Within free tier ✅

## Technical Details

### Why deployment.environment is Required

Grafana Cloud's Tempo metrics generator creates RED metrics from traces using configured dimensions. The `deployment.environment` resource attribute is typically configured as a required dimension. Without it, metrics aren't generated, and services don't appear in Application Observability.

### Resource vs Span Attributes

- **Resource attributes:** Apply to ALL spans from a service (e.g., service.name, deployment.environment)
- **Span attributes:** Specific to individual operations (e.g., http.method, peer.service)

The Tempo metrics generator requires `deployment.environment` at the **resource level**, not span level.

### OTLP Span Format

The policy generates OTLP JSON spans following the OpenTelemetry specification:

```json
{
  "resourceSpans": [{
    "resource": {
      "attributes": [
        {"key": "service.name", "value": {"stringValue": "apim-gateway"}},
        {"key": "deployment.environment", "value": {"stringValue": "production"}}
      ]
    },
    "scopeSpans": [{
      "spans": [{
        "traceId": "...",
        "spanId": "...",
        "parentSpanId": "...",
        "name": "apim: GET /endpoint",
        "kind": 2,
        "attributes": [...]
      }]
    }]
  }]
}
```

## Files

- `README.md` - This documentation
- `apim-policy.xml` - APIM policy with Liquid templates (testing/single API)
- `apim-policy-fragment.xml` - Policy fragment for production (multi-API)
- `alloy-config.alloy` - Alloy collector configuration
- `main.bicep` - Complete Azure infrastructure (APIM, Alloy, Functions)
- `deploy.sh` - Automated deployment script
- `test-tracing.sh` - Test script for distributed tracing scenarios
- `function-app-csharp/` - C# Function App with OpenTelemetry
  - `OrderService.cs` - Order creation with multi-service calls
  - `InventoryService.cs` - Product availability checks
  - `PaymentService.cs` - Payment processing
  - `EchoFunction.cs` - Simple test endpoint

## Resources

- [OpenTelemetry Docs](https://opentelemetry.io/docs/)
- [Grafana Alloy Docs](https://grafana.com/docs/alloy/)
- [Azure APIM Policies](https://learn.microsoft.com/azure/api-management/api-management-policies)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)

## License

MIT License - Free to use and modify

## Contributing

Issues and pull requests welcome!

---

**Example working trace:** `33391e2c4e6b4428a5eb8844b2171ba8` (available in Grafana Labs demo environment)
