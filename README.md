# Azure APIM Distributed Tracing with OpenTelemetry

Production-ready APIM policy for distributed tracing using W3C Trace Context and OTLP.

## What This Is

An Azure API Management policy that enables complete distributed tracing across your API gateway and downstream services. Traces are sent to any OTLP-compatible collector (Grafana Alloy, Jaeger, Tempo, etc.) using the OpenTelemetry Protocol.

**Key capabilities:**
- ✅ W3C Trace Context propagation via `traceparent` header
- ✅ OTLP span generation with Liquid templates
- ✅ Service Graph integration (shows APIM → downstream service connections)
- ✅ Application Observability with RED metrics
- ✅ Vendor-neutral (works with any OTLP collector)

## Repository Contents

### Policy Files

**`apim-policy.xml`** - Complete policy for single API testing/deployment
- Customize 3 lines: collector endpoint URL (line 125), downstream service name (line 176), environment (line 147)
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
| Downstream service name | Line 176 (policy.xml) | Must match your service's `service.name` attribute |
| Environment | Line 147 (policy.xml) | `production`, `staging`, `dev` |

## How It Works

```
Client Request
    ↓
[APIM Policy] (this repository)
    ├─ Parse/generate traceparent header
    ├─ Build OTLP span with Liquid template
    ├─ Send to collector (fire-and-forget)
    └─ Forward traceparent to downstream services
            ↓
    [Your Downstream Services]
        ├─ Receive traceparent
        ├─ Create child spans (via OpenTelemetry SDK)
        └─ Send to collector
                ↓
        [OTLP Collector]
            └─ Forward to observability platform
                    ↓
            [Tempo/Jaeger/etc]
                └─ Service Graph + Traces
```

## Requirements

**For this APIM Policy:**
- Azure API Management (Developer SKU or higher)
- Outbound policy configuration access

**For Your Downstream Services:**
- OpenTelemetry SDK implementation (any language)
- Must set resource attributes: `service.name`, `deployment.environment`
- Must export traces to same collector endpoint as APIM

**OTLP Collector:**
- OTLP HTTP support (port 4318)
- Can be Grafana Alloy, Jaeger, Tempo, or any OTLP-compatible collector

## Critical Configuration

**Service Graph:** For the service graph to connect APIM → your services, set `peer.service` (line 176) to match your service's `service.name` attribute. Case-sensitive.

```
peer.service = "my-api"  →  service.name = "my-api"
```

## License

MIT License - Free to use and modify

## Resources

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [W3C Trace Context Specification](https://www.w3.org/TR/trace-context/)
- [Azure APIM Policy Reference](https://learn.microsoft.com/azure/api-management/api-management-policies)
- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/)
