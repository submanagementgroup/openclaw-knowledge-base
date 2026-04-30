---
domain: gateway
topic: "Gateway Observability: OpenTelemetry Traces and Prometheus Metrics"
type: reference
keywords:
  - OpenTelemetry
  - Prometheus
  - metrics
  - traces
  - observability
  - monitoring
  - telemetry
related:
  - gateway/health-diagnostics-logging
  - gateway/gateway-runbook
source:
  - gateway/opentelemetry.md
  - gateway/prometheus.md
---

OpenClaw supports OpenTelemetry traces and Prometheus metrics for production monitoring.

## OpenTelemetry

```json5
{
  gateway: {
    telemetry: {
      otel: {
        endpoint: "http://localhost:4318/v1/traces",
        serviceName: "openclaw",
      }
    }
  }
}
```

OpenClaw exports diagnostics through the bundled `diagnostics-otel` plugin
using **OTLP/HTTP (protobuf)**. Any collector or backend that accepts OTLP/HTTP
works without code changes. For local file logs and how to read them, see
[Logging](/logging).

## How it fits together

- **Diagnostics events** are structured, in-process records emitted by the
  Gateway and bundled plugins for model runs, message flow, sessions, queues,
  and exec.
- **`diagnostics-otel` plugin** subscribes to those events and exports them as
  OpenTelemetry **metrics**, **traces**, and **logs** over OTLP/HTTP.
- **Provider calls** receive a W3C `traceparent` header from OpenClaw's
  trusted model-call span context when the provider transport accepts custom
  headers. Plugin-emitted trace context is not propagated.
- Exporters only attach when both the diagnostics surface and the plugin are
  enabled, so the in-process cost stays near zero by default.

## Quick start

```json5
{
  plugins: {
    allow: ["diagnostics-otel"],
    entries: {
      "diagnostics-otel": { enabled: true },
    },
  },
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      protocol: "http/protobuf",
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: true,
      sampleRate: 0.2,
      flushIntervalMs: 60000,
    },
  },
}
```

You can also enable the plugin from the CLI:

```bash
openclaw plugins enable diagnostics-otel
```

`protocol` currently supports `http/protobuf` only. `grpc` is ignored.

## Signals exported

| Signal      | What goes in it                                                                                                                            |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **Metrics** | Counters and histograms for token usage, cost, run duration, message flow, queue lanes, session state, exec, and memory pressure.          |
| **Traces**  | Spans for model usage, model calls, harness lifecycle, tool execution, exec, webhook/message processing, context assembly, and tool loops. |
| **Logs**    | Structured `logging.file` records exported over OTLP when `diagnostics.otel.logs` is enabled.                                              |

Toggle `traces`, `metrics`, and `logs` independently. All three default to on
when `diagnostics.otel.enabled` is true.

## Configuration reference

```json5
{
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      tracesEndpoint: "http://otel-collector:4318/v1/traces",
      metricsEndpoint: "http://otel-collector:4318/v1/metrics",
      logsEndpoint: "http://otel-collector:4318/v1/logs",
      protocol: "http/protobuf", // grpc is ignored
      serviceName: "openclaw-gateway",
      headers: { "x-collector-token": "..." },
      traces: true,
      metrics: true,
      logs: true,
      sampleRate: 0.2, // root-span sampler, 0.0..1.0
      flushIntervalMs: 60000, // metric export interval (min 1000ms)
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
      },
    },
  },
}
```

### Environment variables

| Variable                                                                                                          | Purpose                                                                                                                                                                                                                                    |
| ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------

## Prometheus Metrics

```json5
{
  gateway: {
    telemetry: {
      prometheus: {
        enabled: true,
        port: 9090,
      }
    }
  }
}
```

OpenClaw can expose diagnostics metrics through the bundled `diagnostics-prometheus` plugin. It listens to trusted internal diagnostics and renders a Prometheus text endpoint at:

```text
GET /api/diagnostics/prometheus
```

Content type is `text/plain; version=0.0.4; charset=utf-8`, the standard Prometheus exposition format.

The route uses Gateway authentication (operator scope). Do not expose it as a public unauthenticated `/metrics` endpoint. Scrape it through the same auth path you use for other operator APIs.

For traces, logs, OTLP push, and OpenTelemetry GenAI semantic attributes, see [OpenTelemetry export](/gateway/opentelemetry).

## Quick start

        ```json5
        {
          plugins: {
            allow: ["diagnostics-prometheus"],
            entries: {
              "diagnostics-prometheus": { enabled: true },
            },
          },
          diagnostics: {
            enabled: true,
          },
        }
        ```

        ```bash
        openclaw plugins enable diagnostics-prometheus
        ```

    The HTTP route is registered at plugin startup, so reload after enabling.

    Send the same gateway auth your operator clients use:

    ```bash
    curl -H "Authorization: Bearer $OPENCLAW_GATEWAY_TOKEN" \
      http://127.0.0.1:18789/api/diagnostics/prometheus
    ```

    ```yaml
    # prometheus.yml
    scrape_configs:
      - job_name: openclaw
        scrape_interval: 30s
        metrics_path: /api/diagnostics/prometheus
        authorization:
          credentials_file: /etc/prometheus/openclaw-gateway-token
        static_configs:
          - targets: ["openclaw-gateway:18789"]
    ```

`diagnostics.enabled: true` is required. Without it, the plugin still registers the HTTP route but no diagnostic events flow into the exporter, so the response is empty.

## Metrics exported

| Metric                                        | Type      | Labels                                                                                    |
| --------------------------------------------- | --------- | ----------------------------------------------------------------------------------------- |
| `openclaw_run_completed_total`                | counter   | `channel`, `model`, `outcome`, `provider`, `trigger`                                      |
| `openclaw_run_duration_seconds`               | histogram | `channel`, `model`, `outcome`, `provider`, `trigger`                                      |
| `openclaw_model_call_total`                   | counter   | `api`, `error_category`, `model`, `outcome`, `provider`, `transport`                      |
| `openclaw_model_call_duration_seconds`        | histogram | `api`, `error_category`, `model`, `outcome`, `provider`, `transport`                      |
| `openclaw_model_tokens_total`                 | counter   | `agent`, `channel`, `model`, `provider`, `token_type`                                     |
| `openclaw_gen_ai_client_token_usage`          | histogram | `model`, `provider`
