# Azure APIM Distributed Tracing with Grafana Cloud

End-to-end distributed tracing for Azure API Management using OpenTelemetry and Grafana Alloy.

## Overview

This APIM policy enables complete distributed traces showing requests flowing through your API gateway to backend services, with automatic integration into Grafana Cloud's Application Observability and Knowledge Graph.

Already have Grafana Cloud and an OTLP collector? Just apply the policy:

1. **Download:** `apim-policy.xml`
2. **Edit 3 lines:**
   - Line 80: Your OTLP endpoint URL
   - Line 160: Your backend's `service.name` (for peer.service)
   - Line 217: Your environment name (e.g., "production")
3. **Apply:** APIM → APIs → Your API → Policies → Paste → Save
4. **Verify:** Backend exports traces to same collector and sets `deployment.environment` resource attribute

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

```

The `peer.service` span attribute tells Tempo which service APIM is calling. This must match the backend's `service.name` resource attribute (case-sensitive).

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
