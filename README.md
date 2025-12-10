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

## License

MIT License - Free to use and modify

## Resources

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [W3C Trace Context Specification](https://www.w3.org/TR/trace-context/)
- [Azure APIM Policy Reference](https://learn.microsoft.com/azure/api-management/api-management-policies)
- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/)
