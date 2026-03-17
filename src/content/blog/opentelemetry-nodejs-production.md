---
title: "OpenTelemetry in Node.js: A Practical Guide"
date: 2026-03-12
description: "A practical guide to distributed tracing, metrics, and structured logs in Node.js using OpenTelemetry — based on real production instrumentation."
tags: ["node.js", "observability", "opentelemetry", "devops", "kubernetes"]
draft: false
---

# OpenTelemetry in a Node.js Production App: What Actually Works

I've set up observability from scratch more than once. The first time I did it properly — with OpenTelemetry — was during infrastructure work for a platform running on Kubernetes with a handful of Node.js microservices. Before that, we had logs scattered across CloudWatch, no tracing at all, and debugging a slow request meant guessing which service was the culprit.

OTEL fixed that. But the path from "installed the package" to "actually useful in production" has some gaps that most tutorials skip. This is what I wish I'd had.

## What OpenTelemetry Actually Is

OpenTelemetry is a CNCF project that standardizes how you collect **traces**, **metrics**, and **logs** from your application. The key word is *standardizes* — you instrument once and send to whatever backend you want: Grafana Tempo, Jaeger, Datadog, New Relic, whatever.

The three signals:

- **Traces** — the path a request takes across your services. A trace is made of spans; each span is one unit of work (an HTTP call, a DB query, a job execution).
- **Metrics** — counters, gauges, histograms. How many requests per second, p95 latency, queue depth.
- **Logs** — structured log records, ideally correlated with a trace ID so you can jump from a log line to the trace that caused it.

In practice, traces give you the most value first. Start there.

## Project Setup

I'll use a NestJS service as the example because that's what most of my production Node.js work runs on, but the OTEL SDK is framework-agnostic. Express works the same way.

```bash
npm install @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/exporter-metrics-otlp-http \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions
```

`auto-instrumentations-node` is the important one — it automatically patches `http`, `express`, `nestjs-core`, `pg`, `redis`, `ioredis`, and a bunch of other common libraries. You get spans for all your HTTP calls, DB queries, and cache operations without touching application code.

## The Tracing Setup File

This file must be loaded **before anything else** — before NestJS bootstraps, before any imports. That means it needs to be a separate entry point.

```typescript
// src/instrumentation.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-http';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { Resource } from '@opentelemetry/resources';
import { SEMRESATTRS_SERVICE_NAME, SEMRESATTRS_SERVICE_VERSION } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: new Resource({
    [SEMRESATTRS_SERVICE_NAME]: process.env.SERVICE_NAME ?? 'my-service',
    [SEMRESATTRS_SERVICE_VERSION]: process.env.npm_package_version ?? '0.0.0',
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_TRACES_ENDPOINT ?? 'http://localhost:4318/v1/traces',
  }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: process.env.OTEL_EXPORTER_OTLP_METRICS_ENDPOINT ?? 'http://localhost:4318/v1/metrics',
    }),
    exportIntervalMillis: 15000,
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': { enabled: false }, // too noisy
    }),
  ],
});

sdk.start();

process.on('SIGTERM', () => {
  sdk.shutdown().finally(() => process.exit(0));
});
```

Then update your entry point:

```typescript
// src/main.ts
import './instrumentation'; // must be first
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

Or if you use `--require` in your start command:

```bash
node --require ./dist/instrumentation.js dist/main.js
```

I prefer the `--require` approach for production because it makes it easy to toggle OTEL off without changing application code.

## The Collector

You don't send traces directly to Jaeger or Grafana from your app. You send to an **OpenTelemetry Collector**, which handles batching, retries, and fan-out to multiple backends.

For Kubernetes, the collector runs as a DaemonSet (one per node) or a Deployment. Here's a minimal collector config:

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 5s
    send_batch_size: 512
  memory_limiter:
    limit_mib: 256
    check_interval: 1s

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/tempo]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

The `memory_limiter` processor is non-negotiable in production. Without it, if your app suddenly generates a lot of telemetry (say, an incident causing errors to spike), the collector will eat all available memory and crash. I learned this the hard way.

## Using Elastic Agent as a Backend

If you're on the Elastic stack, you can skip the OpenTelemetry Collector entirely and send OTLP data straight to **Elastic Agent**. Since version 8.6, Elastic Agent ships with a built-in OTLP receiver — it accepts traces, metrics, and logs over the standard OTLP protocol and forwards everything to Elasticsearch automatically.

The main advantage over running a separate OTel Collector is operational simplicity: one less component to deploy and maintain, and your telemetry lands directly in the Elastic APM UI with service maps, transaction details, and error grouping out of the box.

### Deploying Elastic Agent in Kubernetes

The recommended setup is a DaemonSet so every node runs one agent instance. Elastic provides a Helm chart:

```bash
helm repo add elastic https://helm.elastic.co
helm repo update

helm install elastic-agent elastic/elastic-agent \
  --namespace monitoring --create-namespace \
  --set agent.fleet.enabled=true \
  --set agent.fleet.url=https://your-fleet-server:8220 \
  --set agent.fleet.token=${FLEET_ENROLLMENT_TOKEN}
```

If you're not using Fleet (fully managed mode), you can run Elastic Agent in standalone mode with a config file instead.

### Enabling the OTLP Input

In Fleet, add an **APM** integration to your agent policy and enable the OTLP endpoint. In standalone mode, configure it directly in `elastic-agent.yml`:

```yaml
inputs:
  - type: apm
    id: apm-otlp
    data_stream.namespace: default
    apm-server:
      host: "0.0.0.0:8200"
      # OTLP is served on a separate port
      otlp:
        grpc:
          host: "0.0.0.0:4317"
        http:
          host: "0.0.0.0:4318"
      output.elasticsearch:
        hosts: ["https://your-es-host:9200"]
        api_key: "${ES_API_KEY}"
```

Elastic Agent exposes OTLP on ports `4317` (gRPC) and `4318` (HTTP) — the same ports the OTel Collector uses. This means you can switch from the OTel Collector to Elastic Agent without changing a single line of application code.

### Pointing Your Node.js App at Elastic Agent

Update the environment variables in your Kubernetes deployment to point at the Elastic Agent pod or service:

```yaml
env:
  - name: SERVICE_NAME
    value: "ratings-api"
  - name: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
    value: "http://elastic-agent.monitoring:4318/v1/traces"
  - name: OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
    value: "http://elastic-agent.monitoring:4318/v1/metrics"
  - name: OTEL_EXPORTER_OTLP_LOGS_ENDPOINT
    value: "http://elastic-agent.monitoring:4318/v1/logs"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "deployment.environment=production"
```

That's it on the application side. The `instrumentation.ts` file doesn't change at all.

### What You Get in Kibana

Once data is flowing, the APM section in Kibana gives you:

- **Service map** — automatic topology showing which services call which
- **Transactions** — p50/p95/p99 latency per endpoint, throughput, error rate
- **Traces** — waterfall view of individual requests with every span
- **Errors** — grouped by exception type with stack traces and occurrence counts
- **Correlations** — Kibana can automatically find which attributes (e.g. a specific user ID, a region, a version) correlate with slow or failing requests

The Elastic APM UI is particularly good at the **correlations** feature — it's not something you get out of the box with Grafana Tempo. If you have an incident and want to know "what's different about the slow requests vs the fast ones", it surfaces that automatically.

### Credentials

Use an Elasticsearch API key rather than username/password for the agent output. Create one in Kibana under Stack Management → API Keys, scope it to the indices Elastic Agent writes to, and inject it as a Kubernetes secret:

```yaml
env:
  - name: ES_API_KEY
    valueFrom:
      secretKeyRef:
        name: elastic-agent-credentials
        key: api-key
```

API keys are easier to rotate than passwords and give you fine-grained access control per agent.

## Adding Custom Spans

Auto-instrumentation gets you a long way, but you'll want custom spans for your business logic. The API is simple:

```typescript
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('my-service');

async function processInvoiceFile(fileKey: string) {
  return tracer.startActiveSpan('process-invoice-file', async (span) => {
    span.setAttribute('file.key', fileKey);
    span.setAttribute('file.type', 'invoice');

    try {
      const file = await downloadFromS3(fileKey);
      span.setAttribute('file.size_bytes', file.length);

      const result = await parseInvoice(file);
      span.setAttribute('invoice.line_count', result.lines.length);

      return result;
    } catch (err) {
      span.recordException(err as Error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: (err as Error).message });
      throw err;
    } finally {
      span.end();
    }
  });
}
```

A few rules I follow for spans:
- **Always call `span.end()`** — use `finally` to guarantee it. A span that never ends leaks memory in the SDK.
- **Set attributes before the work that might fail** — you want them on the span even if an exception is thrown.
- **Don't over-instrument** — one span per meaningful unit of work. Not one per function call.

## Correlating Logs with Traces

This is where things get genuinely useful. If every log line has a trace ID, you can click a slow trace in Tempo and jump straight to the log lines that happened inside it.

With `pino`:

```typescript
import { trace, context } from '@opentelemetry/api';
import pino from 'pino';

const logger = pino({
  mixin() {
    const span = trace.getActiveSpan();
    if (!span) return {};
    const { traceId, spanId, traceFlags } = span.spanContext();
    return { traceId, spanId, traceFlags };
  },
});
```

Now every log line automatically includes the current `traceId`. In Grafana you can set up a derived field that turns the trace ID into a link directly into Tempo. Once that's working, debugging a production issue goes from "grep through CloudWatch and pray" to "find the request in Tempo, follow the spans, jump to the logs."

## Kubernetes Environment Variables

In production, configure OTEL via environment variables in your deployment manifest. This keeps instrumentation code environment-agnostic:

```yaml
env:
  - name: SERVICE_NAME
    value: "ratings-api"
  - name: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
    value: "http://otel-collector.monitoring:4318/v1/traces"
  - name: OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
    value: "http://otel-collector.monitoring:4318/v1/metrics"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "deployment.environment=production,k8s.namespace.name=api"
```

The `OTEL_RESOURCE_ATTRIBUTES` variable lets you tag all telemetry with environment and namespace without changing any code. Very useful when you're running the same service in staging and production and need to filter.

## What I'd Do Differently

**Start with just traces.** The first time I set this up I tried to get traces, metrics, and logs all working at the same time. It was too many moving pieces. Traces alone are immediately valuable — get those working end-to-end first, then add metrics, then log correlation.

**Use the gRPC exporter in production, not HTTP.** I used `exporter-trace-otlp-http` because the HTTP endpoint is easier to debug, but `exporter-trace-otlp-grpc` has lower overhead and better compression. For a low-traffic service it doesn't matter. For a service handling thousands of requests per second, it does.

**Set sampling early.** By default OTEL samples 100% of traces. Fine in development, expensive in production. Add a `TraceIdRatioBased` sampler at setup time:

```typescript
import { TraceIdRatioBased, ParentBasedSampler } from '@opentelemetry/sdk-trace-base';

const sdk = new NodeSDK({
  sampler: new ParentBasedSampler({
    root: new TraceIdRatioBased(0.1), // 10% sampling
  }),
  // ...
});
```

Use `ParentBasedSampler` so that if an upstream service decided to sample a trace, you continue sampling it — you don't want gaps in a distributed trace because one service decided to drop it.

**Add `@opentelemetry/instrumentation-winston` or `-pino` from the start.** Log correlation is almost free to set up but incredibly useful. I didn't add it until later and had to backfill the logging setup across multiple services.

## The Payoff

The first time you catch a slow request, pull it up in Tempo, and see exactly which span inside which service is taking 800ms — and it turns out to be an N+1 query you didn't know existed — the setup effort pays for itself immediately. Before OTEL, that kind of bug could hide for months. After, it's usually visible within a day of new code hitting production.

The OpenTelemetry ecosystem is still maturing. Some instrumentations are rough around the edges, the collector config has a learning curve, and the semantic conventions keep changing. But the core is solid and the vendor neutrality is real — I've pointed the same setup at Grafana Tempo, Jaeger, and Honeycomb without changing a line of application code, just the collector config.

That flexibility alone is worth it.
