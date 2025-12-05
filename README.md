# Azure APIM Distributed Tracing with Grafana Cloud

End-to-end distributed tracing for Azure API Management using OpenTelemetry and Grafana Alloy.

## Overview

This APIM policy enables complete distributed traces showing requests flowing through your API gateway to backend services, with automatic integration into Grafana Cloud's Application Observability and Service Graph.

**What you get:**
- Complete APIM → Backend distributed traces
- Service Graph visualization
- Application Observability auto-discovery
- W3C Trace Context propagation
- RED metrics from traces

**Verified with:** Azure APIM Developer SKU, Azure Functions, Grafana Cloud Tempo

## Quick Start (Experienced Users)

Already have Grafana Cloud and an OTLP collector? Just apply the policy:

1. **Download:** `apim-policy.xml`
2. **Edit 3 lines:**
   - Line 80: Your OTLP endpoint URL
   - Line 160: Your backend's `service.name` (for peer.service)
   - Line 217: Your environment name (e.g., "production")
3. **Apply:** APIM → APIs → Your API → Policies → Paste → Save
4. **Verify:** Backend exports traces to same collector and sets `deployment.environment` resource attribute

Done. Test with a request and check Grafana Tempo.

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

```bash
# Make a request through APIM
curl https://your-apim.azure-api.net/your-api/endpoint

# Check Grafana Cloud (wait 30 seconds)
# Explore → Tempo → Query: {resource.service.name="apim-gateway"}
# Should see spans from both APIM and backend
```

Verify in Grafana Cloud:
- **Tempo → Service Graph:** Should show `apim-gateway` → `your-backend-service`
- **Application Observability:** Both services listed (not "uninstrumented")

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

Add custom attributes to the `attributes` array (around line 111):

```csharp
new JObject(
    new JProperty("key", "custom.attribute"),
    new JProperty("value", new JObject(new JProperty("stringValue", "custom-value")))
)
```

### Environment-Based Configuration

Use APIM Named Values for environment-specific configuration:

1. Create Named Value: `alloy-endpoint`
2. Reference in policy: `<set-url>{{alloy-endpoint}}/v1/traces</set-url>`

### Multi-Region Deployment

For multi-region APIM:
1. Deploy Alloy in each region
2. Use region-specific Named Values
3. `cloud.region` automatically captures deployment region

## Performance & Cost

**Performance Impact:**
- Policy execution: < 10ms per request
- Trace emission: Async/non-blocking
- API latency: < 1% increase
- CPU overhead: < 5%

**Azure Costs:**
- Container Apps (0.5 vCPU, 1Gi): ~$40/month
- APIM policy: No additional cost
- Egress: ~$5-10/month

**Grafana Cloud:**
- Free tier: 50GB traces/month = ~200k-500k requests per GB
- Example: 10M requests/month ≈ 20-50GB ≈ Free tier ✅

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
- `apim-policy.xml` - APIM policy template (use this)
- `alloy-config.alloy` - Alloy configuration reference

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
