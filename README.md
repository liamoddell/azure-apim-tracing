# Azure APIM Distributed Tracing with OpenTelemetry

Production-ready APIM policy for distributed tracing using W3C Trace Context and OTLP.

## What This Is

An Azure API Management policy that enables complete distributed tracing across your API gateway and backend services. Traces are sent to any OTLP-compatible collector (Grafana Alloy, Jaeger, Tempo, etc.) using the OpenTelemetry Protocol.

**Key capabilities:**
- ✅ W3C Trace Context propagation via `traceparent` header
- ✅ OTLP span generation with Liquid templates
- ✅ Service Graph integration (shows APIM → Backend connections)
- ✅ Application Observability with RED metrics
- ✅ Vendor-neutral (works with any OTLP collector)

## Why This Exists

Without distributed tracing, APIM and backend services appear as disconnected systems. When requests are slow or failing, you can't see the complete journey through your architecture.

This policy provides end-to-end visibility by:
- Creating properly linked parent-child span relationships
- Automatically generating service topology graphs
- Enabling root cause analysis from gateway to backend services
- Supporting standards-based observability (OpenTelemetry, W3C)

**vs. Alternatives:**
- **EventHub/Log Analytics**: No trace relationships, delayed visibility, not real-time
- **Application Insights**: Azure-only, expensive at scale, limited W3C Trace Context support

## Repository Contents

### Policy Files

**`apim-policy.xml`** - Complete policy for single API testing/deployment
- Customize 3 lines: endpoint URL (line 125), backend service name (line 176), environment (line 147)
- Uses Liquid templates to build OTLP JSON with dynamic context variables
- Apply via Azure Portal: API → Policies → Outbound → Paste

**`apim-policy-fragment.xml`** - Production-ready fragment for multi-API deployments
- Upload as policy fragment with ID `otlp-tracing`
- Reference APIM Named Value for collector endpoint
- Include in API policies: `<include-fragment fragment-id="otlp-tracing" />`

**`apim-policy-simple.xml`** - Minimal variant for quick prototyping

### Key Configuration Points

All three policy files require these values:

| What | Where | Example |
|------|-------|---------|
| Collector endpoint | Line 125 (policy.xml) | `https://alloy.example.com/v1/traces` |
| Backend service name | Line 176 (policy.xml) | Must match backend's `service.name` |
| Environment | Line 147 (policy.xml) | `production`, `staging`, `dev` |

## Quick Start

1. **Deploy an OTLP collector** (Grafana Alloy, Jaeger, etc.)
2. **Download `apim-policy.xml`** and customize the 3 values above
3. **Apply to your API** via Azure Portal or CLI
4. **Configure your backend** to:
   - Propagate `traceparent` header (automatic with OpenTelemetry SDKs)
   - Set `service.name` resource attribute (must match line 176 in policy)
   - Set `deployment.environment` resource attribute (required for metrics)
   - Export traces to same collector

**Result:** Complete distributed traces showing APIM → Backend with service graph visualization.

## How It Works

```
Client Request
    ↓
[APIM Policy]
    ├─ Parse/generate traceparent header
    ├─ Build OTLP span with Liquid template
    ├─ Send to collector (fire-and-forget)
    └─ Forward traceparent to backend
            ↓
    [Backend Service]
        ├─ Receive traceparent
        ├─ Create child spans
        └─ Send to collector
                ↓
        [OTLP Collector]
            └─ Forward to observability platform
                    ↓
            [Tempo/Jaeger/etc]
                └─ Service Graph + Traces
```

## Requirements

**APIM:**
- Developer SKU or higher
- Outbound policy access

**Backend:**
- OpenTelemetry SDK (any language)
- Resource attributes: `service.name`, `deployment.environment`
- Export to same collector endpoint

**Collector:**
- OTLP HTTP support (port 4318)
- Can be Grafana Alloy, Jaeger, Tempo, or any OTLP-compatible collector

## Critical Configuration

**Service Graph Connection:**
The `peer.service` attribute in the APIM policy (line 176) **must exactly match** the backend's `service.name` resource attribute (case-sensitive). This is how the service graph connects APIM → Backend.

```
APIM Policy:                          Backend Config:
peer.service = "my-api"          →    service.name = "my-api"
                                 ✅ MUST MATCH
```

**Metrics Generation:**
The `deployment.environment` resource attribute (line 147) is required by most observability platforms to generate RED metrics (Rate, Errors, Duration).

## Liquid Template Syntax

The policy uses Azure APIM's Liquid template engine to dynamically build OTLP JSON payloads at runtime:

```json
{
  "traceId": "{{context.Variables.trace_id}}",
  "attributes": [
    { "key": "http.method", "value": { "stringValue": "{{context.Request.Method}}" } },
    { "key": "http.status_code", "value": { "intValue": {{context.Response.StatusCode}} } }
  ]
}
```

**Available context variables:**
- Request: `{{context.Request.Method}}`, `{{context.Request.Url}}`, `{{context.Request.IpAddress}}`
- Response: `{{context.Response.StatusCode}}`
- API: `{{context.Api.Name}}`, `{{context.Product.Name}}`
- Deployment: `{{context.Deployment.Region}}`
- Custom: `{{context.Variables.your_variable}}`

## Performance & Cost

**Performance:**
- Policy execution: ~5ms per request
- Non-blocking trace emission (fire-and-forget)
- <1% latency impact

**Cost (example):**
- OTLP Collector (Azure Container Apps, 0.5 vCPU): ~$35/month
- Grafana Cloud free tier: 50GB traces/month (~10M+ requests)
- Total: $35-40/month for complete distributed tracing

## License

MIT License - Free to use and modify

## Resources

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [W3C Trace Context Specification](https://www.w3.org/TR/trace-context/)
- [Azure APIM Policy Reference](https://learn.microsoft.com/azure/api-management/api-management-policies)
- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/)
