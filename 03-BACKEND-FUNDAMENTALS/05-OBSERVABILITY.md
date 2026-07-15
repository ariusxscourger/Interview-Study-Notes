# OBSERVABILITY - LOGGING, METRICS, TRACING
## Exam-Style Study Notes

---

## TOPIC: Three Pillars of Observability

### WHY? (Problem Statement)

**Without Observability:**
- "It works on my machine" → Production failures
- MTTR (Mean Time To Resolution) = Hours/Days
- No visibility into distributed systems
- Can't answer: "Why is it slow?" "What failed?" "Who is affected?"

**What Observability Solves:**
| Pillar | Purpose | Questions Answered |
|--------|---------|-------------------|
| **Logs** | Discrete events | What happened? When? |
| **Metrics** | Aggregated measurements | How much? How fast? How many errors? |
| **Traces** | Request flow | Where did it go? Where is latency? |

**Real-World Analogy:**
- Logs = Security camera footage (discrete events)
- Metrics = Dashboard gauges (speed, RPM, fuel)
- Traces = GPS route with timestamps (full journey)

---

### HOW? (Implementation Patterns)

#### 1. **Structured Logging**

```typescript
// Structured Logger Interface
interface LogEntry {
  timestamp: string;      // ISO 8601
  level: 'debug' | 'info' | 'warn' | 'error' | 'fatal';
  message: string;        // Human-readable
  service: string;        // Service name
  traceId?: string;       // For correlation
  spanId?: string;        // For correlation
  // Structured fields (searchable)
  userId?: string;
  requestId?: string;
  duration?: number;
  error?: {
    name: string;
    message: string;
    stack?: string;
    code?: string;
  };
  // Any additional context
  [key: string]: any;
}

// Logger Implementation
class StructuredLogger {
  constructor(
    private serviceName: string,
    private output: 'console' | 'file' | 'stdout' | 'elastic' = 'console',
    private level: LogLevel = 'info'
  ) {}

  private shouldLog(level: LogLevel): boolean {
    const levels = ['debug', 'info', 'warn', 'error', 'fatal'];
    return levels.indexOf(level) >= levels.indexOf(this.level);
  }

  log(level: LogLevel, message: string, meta: Record<string, any> = {}) {
    if (!this.shouldLog(level)) return;

    const entry: LogEntry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      service: this.serviceName,
      ...meta,
    };

    // Add trace context if available
    const traceContext = getTraceContext();
    if (traceContext) {
      entry.traceId = traceContext.traceId;
      entry.spanId = traceContext.spanId;
    }

    // Redact sensitive data
    const sanitized = this.sanitize(entry);
    
    // Output
    this.outputLog(sanitized);
  }

  private sanitize(entry: LogEntry): LogEntry {
    const sensitiveKeys = ['password', 'token', 'secret', 'authorization', 'cookie', 'ssn', 'creditCard'];
    const sanitized = { ...entry };
    
    const sanitizeObj = (obj: any): any => {
      if (Array.isArray(obj)) return obj.map(sanitizeObj);
      if (obj && typeof obj === 'object') {
        const result: any = {};
        for (const [key, value] of Object.entries(obj)) {
          if (sensitiveKeys.some(k => key.toLowerCase().includes(k))) {
            result[key] = '[REDACTED]';
          } else {
            result[key] = sanitizeObj(value);
          }
        }
        return result;
      }
      return obj;
    };

    return sanitizeObj(sanitized);
  }

  // Convenience methods
  debug(message: string, meta?: any) { this.log('debug', message, meta); }
  info(message: string, meta?: any) { this.log('info', message, meta); }
  warn(message: string, meta?: any) { this.log('warn', message, meta); }
  error(message: string, error?: Error, meta?: any) { 
    this.log('error', message, { 
      error: error ? { name: error.name, message: error.message, stack: error.stack } : undefined,
      ...meta 
    }); 
  }
  fatal(message: string, error?: Error, meta?: any) { 
    this.log('fatal', message, { 
      error: error ? { name: error.name, message: error.message, stack: error.stack } : undefined,
      ...meta 
    }); 
  }
}

// Usage
const logger = new StructuredLogger('order-service');

app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    logger.info('HTTP Request', {
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration: Date.now() - start,
      userId: req.user?.id,
      requestId: req.headers['x-request-id'],
    });
  });
  next();
});
```

#### 2. **Metrics Collection (Prometheus Format)**

```typescript
// Metric Types
type MetricType = 'counter' | 'gauge' | 'histogram' | 'summary';

interface Metric {
  name: string;
  type: MetricType;
  help: string;
  labels: Record<string, string>;
  value: number;
  timestamp: number;
}

// Metrics Collector
class MetricsCollector {
  private metrics = new Map<string, Metric>();
  private defaultLabels: Record<string, string> = {};

  constructor(private serviceName: string) {
    this.defaultLabels = { service: serviceName };
  }

  // Counter - Only increases (requests, errors, etc.)
  increment(name: string, labels: Record<string, string> = {}, value = 1): void {
    const key = this.getKey(name, labels);
    const existing = this.metrics.get(key);
    this.metrics.set(key, {
      name,
      type: 'counter',
      help: '',
      labels: { ...this.defaultLabels, ...labels },
      value: (existing?.value || 0) + value,
      timestamp: Date.now(),
    });
  }

  // Gauge - Can go up/down (memory, connections, queue size)
  setGauge(name: string, value: number, labels: Record<string, string> = {}): void {
    const key = this.getKey(name, labels);
    this.metrics.set(key, {
      name,
      type: 'gauge',
      help: '',
      labels: { ...this.defaultLabels, ...labels },
      value,
      timestamp: Date.now(),
    });
  }

  // Histogram - Distribution of values (latency, payload size)
  observeHistogram(name: string, value: number, labels: Record<string, string> = {}): void {
    const key = this.getKey(name, labels);
    const existing = this.metrics.get(key);
    
    // For simplicity, track sum/count/buckets
    // Real implementation uses buckets (e.g., [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10])
    const buckets = [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10];
    
    const existingMetric = this.metrics.get(key) as any || {
      name,
      type: 'histogram',
      help: '',
      labels: { ...this.defaultLabels, ...labels },
      sum: 0,
      count: 0,
      buckets: buckets.reduce((acc, b) => ({ ...acc, [`le:${b}`]: 0 }), {}),
      timestamp: Date.now(),
    };

    existingMetric.sum += value;
    existingMetric.count += 1;
    for (const bucket of buckets) {
      if (value <= bucket) {
        existingMetric.buckets[`le:${bucket}`] = (existingMetric.buckets[`le:${bucket}`] || 0) + 1;
      }
    }
    existingMetric.timestamp = Date.now();
    
    this.metrics.set(this.getKey(name, labels), existingMetric);
  }

  // Summary - Quantiles over sliding window
  observeSummary(name: string, value: number, labels: Record<string, string> = {}): void {
    // Similar to histogram but tracks quantiles
  }

  // Export Prometheus format
  export(): string {
    const lines: string[] = [];
    for (const metric of this.metrics.values()) {
      lines.push(`# HELP ${metric.name} ${metric.help}`);
      lines.push(`# TYPE ${metric.name} ${metric.type}`);
      
      const labelStr = Object.entries(metric.labels)
        .map(([k, v]) => `${k}="${v}"`)
        .join(',');

      if (metric.type === 'histogram') {
        const m = metric as any;
        for (const [bucket, count] of Object.entries(m.buckets)) {
          lines.push(`${metric.name}_bucket{${labelStr},${bucket}} ${count}`);
        }
        lines.push(`${metric.name}_sum{${labelStr}} ${m.sum}`);
        lines.push(`${metric.name}_count{${labelStr}} ${m.count}`);
      } else {
        lines.push(`${metric.name}{${labelStr}} ${metric.value}`);
      }
    }
    return lines.join('\n') + '\n';
  }

  private getKey(name: string, labels: Record<string, string>): string {
    const labelStr = Object.entries(labels).sort().map(([k, v]) => `${k}=${v}`).join(',');
    return `${name}{${labelStr}}`;
  }
}

// HTTP Middleware for Metrics
function metricsMiddleware(collector: MetricsCollector) {
  return (req: Request, res: Response, next: NextFunction) => {
    const start = process.hrtime.bigint();
    const route = req.route?.path || req.path;
    const method = req.method;

    res.on('finish', () => {
      const duration = Number(process.hrtime.bigint() - start) / 1e9; // seconds
      const labels = { 
        method, 
        route, 
        statusCode: res.statusCode.toString(),
        service: 'api-gateway',
      };

      collector.increment('http_requests_total', { ...labels });
      collector.observeHistogram('http_request_duration_seconds', duration, { 
        method, 
        route 
      });

      if (res.statusCode >= 500) {
        collector.increment('http_errors_total', { ...labels, type: 'server_error' });
      } else if (res.statusCode >= 400) {
        collector.increment('http_errors_total', { ...labels, type: 'client_error' });
      }
    });

    next();
  });
}

// Business Metrics
class BusinessMetrics {
  constructor(private collector: MetricsCollector) {}

  orderCreated(order: Order): void {
    this.collector.increment('orders_created_total', { 
      status: order.status,
      channel: order.channel,
    });
    this.collector.observeHistogram('order_value_dollars', order.total / 100, {
      currency: order.currency,
    });
  }

  userRegistered(user: User): void {
    this.collector.increment('users_registered_total', { 
      source: user.source,
      plan: user.plan,
    });
  }

  activeUsers(count: number): void {
    this.collector.setGauge('active_users', count);
  }

  queueDepth(queue: string, depth: number): void {
    this.collector.setGauge('queue_depth', depth, { queue });
  }
}
```

#### 3. **Distributed Tracing (OpenTelemetry)**

```typescript
// Trace Context Propagation
interface TraceContext {
  traceId: string;
  spanId: string;
  traceFlags: number;
  traceState?: string;
}

// W3C Trace Context Header: traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
// Format: version-trace-id-span-id-flags

class Tracer {
  private sampler: Sampler;
  private exporter: SpanExporter;

  constructor(
    private serviceName: string,
    sampler: Sampler = new AlwaysOnSampler(),
    exporter: SpanExporter
  ) {
    this.sampler = sampler;
    this.exporter = exporter;
  }

  // Start a new span
  startSpan(name: string, options: SpanOptions = {}): Span {
    const traceId = options.parentContext?.traceId || this.generateTraceId();
    const spanId = this.generateSpanId();
    const parentSpanId = options.parentContext?.spanId;

    const span: Span = {
      traceId,
      spanId,
      parentSpanId: parentSpanId,
      name,
      kind: options.kind || SpanKind.INTERNAL,
      startTime: process.hrtime.bigint(),
      attributes: {},
      events: [],
      links: [],
      status: { code: SpanStatusCode.UNSET },
    };

    // Check sampling decision
    const samplingResult = this.sampler.shouldSample({
      traceId,
      name,
      kind: span.kind,
      attributes: options.attributes,
    });

    if (!samplingResult.sampled) {
      span.dropped = true;
    }

    // Set initial attributes
    if (options.attributes) {
      for (const [key, value] of Object.entries(options.attributes)) {
        span.setAttribute(key, value);
      }
    }

    return span;
  }

  // End span and export
  endSpan(span: Span): void {
    if (span.dropped) return;
    
    span.endTime = process.hrtime.bigint();
    this.exporter.export([span]);
  }

  // HTTP Client Instrumentation
  async traceFetch<T>(
    name: string, 
    fetchFn: () => Promise<T>,
    attributes: Record<string, any> = {}
  ): Promise<T> {
    const span = this.startSpan(name, { 
      kind: SpanKind.CLIENT,
      attributes: {
        'http.method': 'GET',
        ...attributes,
      },
    });

    // Inject trace context into headers
    const headers = new Headers();
    this.injectContext(span, headers);

    try {
      const response = await fetchFn();
      
      span.setAttribute('http.status_code', (response as any).status || 200);
      if ((response as any).status >= 400) {
        span.setStatus({ code: SpanStatusCode.ERROR });
      }
      
      return response;
    } catch (error) {
      span.setStatus({ 
        code: SpanStatusCode.ERROR, 
        message: error instanceof Error ? error.message : String(error) 
      });
      span.recordException(error);
      throw error;
    } finally {
      this.endSpan(span);
    }
  }

  // Inject trace context for propagation
  injectContext(span: Span, carrier: Record<string, string> | Headers): void {
    const traceparent = `00-${span.traceId}-${span.spanId}-${span.traceFlags.toString(16).padStart(2, '0')}`;
    
    if (carrier instanceof Headers) {
      carrier.set('traceparent', traceparent);
      if (span.traceState) carrier.set('tracestate', span.traceState);
    } else {
      carrier['traceparent'] = traceparent;
      if (span.traceState) carrier['tracestate'] = span.traceState;
    }
  }

  // Extract trace context from incoming request
  extractContext(carrier: Record<string, string> | Headers): TraceContext | null {
    let traceparent: string | undefined;
    let tracestate: string | undefined;

    if (carrier instanceof Headers) {
      traceparent = carrier.get('traceparent');
      tracestate = carrier.get('tracestate');
    } else {
      traceparent = carrier['traceparent'];
      tracestate = carrier['tracestate'];
    }

    if (!traceparent) return null;

    // Parse: version-trace-id-span-id-flags
    const parts = traceparent.split('-');
    if (parts.length !== 4) return null;

    return {
      traceId: parts[1],
      spanId: parts[2],
      traceFlags: parseInt(parts[3], 16),
      traceState: tracestate,
    };
  }

  private generateTraceId(): string {
    return crypto.randomBytes(16).toString('hex');
  }

  private generateSpanId(): string {
    return crypto.randomBytes(8).toString('hex');
  }
}

// Automatic HTTP Instrumentation
function tracingMiddleware(tracer: Tracer) {
  return (req: Request, res: Response, next: NextFunction) => {
    // Extract parent context
    const parentContext = tracer.extractContext(req.headers);
    
    // Start server span
    const span = tracer.startSpan(`${req.method} ${req.route?.path || req.path}`, {
      kind: SpanKind.SERVER,
      parentContext,
      attributes: {
        'http.method': req.method,
        'http.route': req.route?.path || req.path,
        'http.target': req.path,
        'http.scheme': req.protocol,
        'http.host': req.get('host'),
        'http.user_agent': req.get('user-agent'),
        'net.host.port': req.socket.remotePort?.toString(),
        'net.peer.ip': req.ip,
      },
    });

    // Store span on request for child spans
    req.span = span;

    // Inject response headers for downstream
    const originalSend = res.send;
    res.send = function(body) {
      tracer.endSpan(span);
      return originalSend.call(this, body);
    };

    // Handle errors
    res.on('finish', () => {
      span.setAttribute('http.status_code', res.statusCode);
      if (res.statusCode >= 400) {
        span.setStatus({ code: SpanStatusCode.ERROR });
      }
    });

    next();
  };
}

// Database Instrumentation
function traceDatabase(query: string, params: any[], span: Span): void {
  span.setAttribute('db.system', 'postgresql');
  span.setAttribute('db.operation', query.split(' ')[0].toUpperCase());
  span.setAttribute('db.statement', query);
  // Don't log params in production (PII risk)
  if (process.env.NODE_ENV === 'development') {
    span.setAttribute('db.params', JSON.stringify(params));
  }
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Structured JSON Logs** | Plain text / printf | Not queryable, no correlation |
| **Structured Metrics** | Custom formats | Prometheus/Grafana won't work |
| **Trace Context Propagation** | Manual header passing | Broken traces across services |
| **Semantic Attributes** | Custom attribute names | No standardization, tooling fails |
| **Sampling** | 100% tracing always | Cost explosion, noise |
| **Cardinality Control** | High-cardinality labels (userId, requestId) | Prometheus OOM, slow queries |
| **Log Levels** | Everything ERROR | Alert fatigue, missed issues |
| **Centralized Collection** | SSH to servers | Doesn't scale |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "Three Pillars - how do they work together?"

```
LOGS + METRICS + TRACES = COMPLETE PICTURE

Example: "API latency spike"
1. METRICS alert: p99 latency > 2s (detect)
2. TRACES: Find slow requests, see which span (localize)
   - DB query taking 1.8s
3. LOGS: Check DB query logs for that traceId
   - "Slow query: missing index on orders.user_id"
4. METRICS: Verify fix - latency back to normal

CORRELATION:
- Trace ID in logs
- Trace ID in metrics (exemplars)
- Span ID links parent/child
```

#### 2. **Code**: "Implement Cardinality-Safe Metrics"

```typescript
// BAD: High Cardinality
collector.increment('requests', { userId: req.user.id, requestId: req.id });
// Creates millions of time series → Prometheus OOM

// GOOD: Low Cardinality
collector.increment('requests_total', { 
  method: req.method,
  route: req.route?.path || 'unknown',
  statusClass: `${Math.floor(res.statusCode / 100)}xx`, // 2xx, 4xx, 5xx
});

// For high-cardinality data → Use LOGS or TRACES
logger.info('Request completed', {
  userId: req.user.id,        // High cardinality - OK in logs
  requestId: req.id,          // High cardinality - OK in logs
  duration: Date.now() - start,
});

// For user-level metrics → Pre-aggregate
class UserMetricsAggregator {
  private aggregates = new Map<string, { count: number; errors: number; latency: number[] }>();

  record(userId: string, success: boolean, latency: number): void {
    const bucket = this.getTimeBucket(); // e.g., 1-minute buckets
    const key = `${userId}:${bucket}`;
    
    const agg = this.aggregates.get(key) || { count: 0, errors: 0, latency: [] };
    agg.count++;
    if (!success) agg.errors++;
    agg.latency.push(latency);
    
    // Keep only recent latency values (memory bound)
    if (agg.latency.length > 1000) agg.latency.shift();
    
    this.aggregates.set(key, agg);
  }

  flush(): void {
    for (const [key, agg] of this.aggregates) {
      const [userId, bucket] = key.split(':');
      collector.increment('user_requests_total', { userId, bucket }, agg.count);
      collector.increment('user_errors_total', { userId, bucket }, agg.errors);
      collector.observeHistogram('user_latency_seconds', agg.latency.reduce((a,b)=>a+b)/agg.latency.length, { userId, bucket });
    }
    this.aggregates.clear();
  }
}
```

#### 3. **Design**: "OpenTelemetry Collector Pipeline"

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────┐     ┌─────────────┐
│  Services   │────►│  OTel Collector  │────►│  Processors │────►│  Exporters  │
│  (Instrument│     │  (Agent/Gateway) │     │             │     │             │
│   + SDK)    │     │                  │     │  • Batch    │     │  • Prometheus
│             │     │  • Receivers     │     │  • Memory   │     │  • Jaeger
│             │     │  • Processors    │     │  • Tail     │     │  • Zipkin
│             │     │  • Exporters     │     │  • Filter   │     │  • OTLP
│             │     │                  │     │  • Transform│     │  • CloudWatch
└─────────────┘     └──────────────────┘     └─────────────┘     └─────────────┘

COLLECTOR CONFIG (YAML):
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch:
    timeout: 10s
    send_batch_size: 1000
  memory_limiter:
    check_interval: 1s
    limit_mib: 500
  tail_sampling:
    decision_wait: 10s
    num_traces: 10000
    policies:
      - name: errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: slow
        type: latency
        latency: { threshold_ms: 500 }
      - name: random
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
  jaeger:
    endpoint: "jaeger:14250"
    tls: { insecure: true }
  otlp:
    endpoint: "tempo:4317"
    tls: { insecure: true }

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, tail_sampling]
      exporters: [jaeger, otlp]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
```

#### 4. **Performance**: "Log Sampling & Cost Optimization"

```typescript
class SamplingLogger extends StructuredLogger {
  private sampleRates: Map<string, number> = new Map();
  private counters: Map<string, number> = new Map();

  constructor(serviceName: string, defaultSampleRate = 1.0) {
    super(serviceName);
    // Configure per-level sampling
    this.sampleRates.set('debug', 0.01);   // 1% of debug
    this.sampleRates.set('info', 0.1);     // 10% of info
    this.sampleRates.set('warn', 1.0);     // 100% of warn
    this.sampleRates.set('error', 1.0);    // 100% of error
    this.sampleRates.set('fatal', 1.0);    // 100% of fatal
  }

  shouldSample(level: LogLevel, attributes: Record<string, any>): boolean {
    // Always sample errors/fatals
    if (level === 'error' || level === 'fatal') return true;

    // Rate-based sampling
    const rate = this.sampleRates.get(level) || 1.0;
    if (Math.random() < rate) return true;

    // Always sample if has error attribute
    if (attributes.error || attributes.statusCode >= 500) return true;

    // Always sample slow requests
    if (attributes.duration > 1000) return true;

    return false;
  }

  log(level: LogLevel, message: string, meta: Record<string, any> = {}) {
    if (!this.shouldSample(level, meta)) return;
    super.log(level, message, meta);
  }
}

// Cost-Aware Logging
class CostAwareLogger {
  private dailyBudget = 10 * 1024 * 1024 * 1024; // 10 GB/day
  private dailyUsage = 0;
  private lastReset = Date.now();

  log(entry: LogEntry): void {
    const size = JSON.stringify(entry).length;
    
    // Reset daily counter
    if (Date.now() - this.lastReset > 24 * 60 * 60 * 1000) {
      this.dailyUsage = 0;
      this.lastReset = Date.now();
    }

    // Check budget
    if (this.dailyUsage + size > this.dailyBudget) {
      // Switch to minimal logging
      if (!this.isCritical(entry)) return;
    }

    this.dailyUsage += size;
    this.output(entry);
  }

  private isCritical(entry: LogEntry): boolean {
    return entry.level === 'error' || entry.level === 'fatal' || 
           entry.attributes?.alert === true;
  }
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **High Cardinality** - Never use userId, requestId, sessionId as metric labels
- [ ] **Missing Context** - Always include traceId/spanId in logs
- [ ] **Clock Sync** - Use NTP; distributed traces need synchronized clocks
- [ ] **Sampling Bias** - Tail sampling misses early errors; use head + tail
- [ ] **Log Volume** - Structured logs can be 10x larger; budget accordingly
- [ ] **Metric Naming** - Follow Prometheus conventions: `http_requests_total`, not `httpRequests`
- [ ] **Histogram Buckets** - Default buckets often wrong; customize for your latency profile
- [ ] **Exemplars** - Link metrics to traces via exemplars (Prometheus + Grafana)
- [ ] **Log Redaction** - Automatically redact PII/secrets before output
- [ ] **Context Propagation** - W3C TraceContext standard; don't invent your own

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Observability Setup**
```typescript
// observability.ts
import { NodeTracerProvider } from '@opentelemetry/sdk-trace-node';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { JaegerExporter } from '@opentelemetry/exporter-jaeger';
import { PrometheusExporter } from '@opentelemetry/exporter-prometheus';
import { MeterProvider } from '@opentelemetry/sdk-metrics';

// Tracing
const tracerProvider = new NodeTracerProvider({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'order-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.2.3',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV,
  }),
});

const jaegerExporter = new JaegerExporter({
  endpoint: process.env.JAEGER_ENDPOINT || 'http://jaeger:14268/api/traces',
});

tracerProvider.addSpanProcessor(new BatchSpanProcessor(jaegerExporter));
tracerProvider.register();

// Metrics
const meterProvider = new MeterProvider({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'order-service',
  }),
});

const prometheusExporter = new PrometheusExporter({
  port: 9464,
  endpoint: '/metrics',
}, () => {
  console.log('Prometheus metrics available at http://localhost:9464/metrics');
});

meterProvider.addMetricReader(prometheusExporter);

// Global access
export const tracer = tracerProvider.getTracer('order-service');
export const meter = meterProvider.getMeter('order-service');

// Custom Metrics
export const httpRequestsTotal = meter.createCounter('http_requests_total', {
  description: 'Total HTTP requests',
});
export const httpRequestDuration = meter.createHistogram('http_request_duration_seconds', {
  description: 'HTTP request latency',
  boundaries: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
});
export const activeConnections = meter.createUpDownCounter('active_connections', {
  description: 'Active connections',
});

// Structured Logging
export const logger = new StructuredLogger('order-service');

// Helper: Get current span
export function getCurrentSpan(): Span | undefined {
  return trace.getSpan(context.active());
}

// Helper: Add attributes to current span
export function addSpanAttributes(attributes: Record<string, any>): void {
  const span = getCurrentSpan();
  if (span) {
    for (const [key, value] of Object.entries(attributes)) {
      span.setAttribute(key, value);
    }
  }
}
```

#### 2. **Health Check with Observability**
```typescript
// health.ts
class HealthChecker {
  private checks = new Map<string, HealthCheck>();

  register(name: string, check: HealthCheck): void {
    this.checks.set(name, check);
  }

  async check(): Promise<HealthResult> {
    const results = await Promise.allSettled(
      Array.from(this.checks.entries()).map(async ([name, check]) => {
        const start = Date.now();
        try {
          const healthy = await check();
          return { name, healthy: true, latency: Date.now() - start };
        } catch (error) {
          return { 
            name, 
            healthy: false, 
            latency: Date.now() - start,
            error: error instanceof Error ? error.message : String(error),
          };
        }
      })
    );

    const healthy = results.every(r => r.status === 'fulfilled' && r.value.healthy);
    
    return {
      status: healthy ? 'healthy' : 'unhealthy',
      timestamp: new Date().toISOString(),
      checks: results.map(r => 
        r.status === 'fulfilled' ? r.value : { 
          name: 'unknown', 
          healthy: false, 
          error: r.reason?.message 
        }
      ),
    };
  }
}

// Usage
const health = new HealthChecker();

health.register('database', async () => {
  await db.query('SELECT 1');
  return true;
});

health.register('redis', async () => {
  await redis.ping();
  return true;
});

health.register('external-api', async () => {
  const response = await fetch('https://api.example.com/health', { timeout: 2000 });
  return response.ok;
});

// Express endpoint
app.get('/health', async (req, res) => {
  const result = await health.check();
  res.status(result.status === 'healthy' ? 200 : 503).json(result);
});

app.get('/health/live', (req, res) => {
  res.json({ status: 'alive' }); // Liveness - process is running
});

app.get('/health/ready', async (req, res) => {
  const result = await health.check();
  res.status(result.status === 'healthy' ? 200 : 503).json(result);
});
```

---

### PRACTICE PROBLEMS

1. **Implement** a distributed tracing system with baggage propagation
2. **Build** a log aggregation pipeline: Fluent Bit → Kafka → Elasticsearch
3. **Create** a custom Prometheus metrics exporter for business KPIs
4. **Design** an alerting strategy: SLI/SLO/SLA, error budgets, burn rates
5. **Implement** trace sampling: head-based, tail-based, adaptive

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Three Pillars | Logs (events), Metrics (aggregated), Traces (request flow) |
| Trace Context Header | `traceparent: version-traceId-spanId-flags` (W3C standard) |
| Span Attributes | Semantic conventions: http.method, db.statement, net.peer.ip |
| Cardinality Problem | High unique label values → Prometheus OOM. Fix: aggregate, don't label |
| Head vs Tail Sampling | Head: decide at start (fast, may miss errors). Tail: decide at end (sees full trace) |
| Exemplars | Link metric data point to trace ID (Grafana + Prometheus) |
| SLI/SLO/SLA | SLI = metric, SLO = target, SLA = contract with penalty |
| Error Budget | 1 - SLO = error budget. Burn rate = how fast consuming budget |
| Log Levels | DEBUG < INFO < WARN < ERROR < FATAL |
| Structured Logging | JSON with consistent fields (timestamp, level, traceId, message) |
| Metric Types | Counter (inc), Gauge (set), Histogram (observe), Summary (quantiles) |
| RED Metrics | Rate, Errors, Duration (per service) |
| USE Metrics | Utilization, Saturation, Errors (per resource) |

---

## NEXT TOPIC: `06-SYSTEM-DESIGN.md`

> **Study Tip**: Instrument a service with OpenTelemetry. Export to Jaeger + Prometheus. Create a dashboard in Grafana with RED metrics + trace links.