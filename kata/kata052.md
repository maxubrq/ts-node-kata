# Kata 52 — Tracing with Context

*Trace id xuyên service boundary (HTTP → HTTP, optional: queue), có context propagation chuẩn, nhìn được end-to-end.*

---

## 1) Goal (năng lực cần đạt)

Sau kata này bạn làm được:

* Tạo **distributed trace end-to-end**: request vào `api` → gọi sang `worker` → trả về.
* **Propagation đúng chuẩn** (W3C Trace Context: `traceparent`, `tracestate`), không tự bịa header.
* **Context không bị mất** qua async/await, timers, Promise chains (Node context manager).
* Có **log correlation**: log nào cũng có `trace_id` / `span_id`.
* Có **tests** chứng minh:

  * Nếu client gửi `traceparent` thì trace tiếp tục cùng `trace_id`.
  * Nếu không có thì service tạo trace mới.
  * Child span đúng parent span khi call downstream.
* (Senior+) Có **sampling + redaction** + tránh high-cardinality attributes.

---

## 2) Context (bài toán)

Bạn build 2 service Node/TS:

### Service A: `api`

* `GET /checkout`

  * tạo span `GET /checkout`
  * call sang Service B: `POST /charge`
  * trả `200` hoặc `502` nếu B fail

### Service B: `worker`

* `POST /charge`

  * span `POST /charge`
  * fake payment: sleep + fail rate
  * trả `200` hoặc `402` (declined) hoặc `500` (error)

Expose thêm:

* `GET /debug/trace` (optional) để dump trace ids gần nhất (chỉ phục vụ test/dev).

Exporter: **console exporter** là đủ cho kata (không cần Jaeger). Nếu bạn muốn “đã mắt”, dùng OTLP → Jaeger/Tempo cũng được, nhưng không bắt buộc.

---

## 3) Constraints (ép đúng “production tracing”)

1. **Không tự set trace id thủ công** trong business code. Chỉ dùng OpenTelemetry API.
2. **Bắt buộc propagation qua HTTP headers** theo chuẩn W3C.
3. **Span naming chuẩn**: `HTTP <method> <route>` (ví dụ `HTTP GET /checkout`), không dùng raw URL có id.
4. **Attributes không nổ cardinality**:

   * ✅ `http.route`, `http.method`, `http.status_code`
   * ❌ `userId`, `email`, `full URL`, `ip` (trừ khi đã hash/allowlist)
5. **Error semantics**:

   * 4xx không phải lúc nào cũng “error” span; 5xx thì thường là error.
   * Nếu downstream fail, span client phải record exception + status error.
6. **Test phải chứng minh propagation** + parent/child đúng.
7. **Log correlation** phải có `trace_id`, `span_id` trong mọi log trong request.

---

## 4) Deliverables (cấu trúc repo)

```
kata-52/
├─ packages/
│  ├─ api/
│  │  ├─ src/
│  │  │  ├─ server.ts
│  │  │  ├─ tracing.ts
│  │  │  └─ logger.ts
│  │  └─ test/tracing.spec.ts
│  ├─ worker/
│  │  ├─ src/
│  │  │  ├─ server.ts
│  │  │  ├─ tracing.ts
│  │  │  └─ logger.ts
│  │  └─ test/tracing.spec.ts
│  └─ shared/
│     └─ src/otel.ts (optional shared helpers)
├─ docker-compose.yml (optional)
└─ README.md
```

Dependencies (gợi ý):

* `@opentelemetry/api`
* `@opentelemetry/sdk-node`
* `@opentelemetry/auto-instrumentations-node`
* `@opentelemetry/exporter-trace-otlp-http` (optional) **hoặc** `@opentelemetry/sdk-trace-base` + `ConsoleSpanExporter`
* `pino` (logging)
* `vitest` + `undici` hoặc `supertest`

---

## 5) Implementation steps (làm theo thứ tự)

### Step 1 — Setup tracing bootstrap (mỗi service)

Tạo `src/tracing.ts`:

```ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { resourceFromAttributes } from "@opentelemetry/resources";
import { SemanticResourceAttributes } from "@opentelemetry/semantic-conventions";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { diag, DiagConsoleLogger, DiagLogLevel } from "@opentelemetry/api";
import { ConsoleSpanExporter, SimpleSpanProcessor } from "@opentelemetry/sdk-trace-base";

// Dev visibility
diag.setLogger(new DiagConsoleLogger(), DiagLogLevel.ERROR);

export async function startTracing(serviceName: string) {
  const sdk = new NodeSDK({
    resource: resourceFromAttributes({
      [SemanticResourceAttributes.SERVICE_NAME]: serviceName,
      // add service.version, deployment.environment nếu muốn
    }),
    // Auto instruments: http, undici, etc.
    instrumentations: [getNodeAutoInstrumentations()],
    // Exporter: console đủ cho kata
    spanProcessor: new SimpleSpanProcessor(new ConsoleSpanExporter()),
  });

  await sdk.start();
  return sdk;
}
```

Trong `src/server.ts` của mỗi service: **start tracing trước khi start server**.

```ts
import { startTracing } from "./tracing";

await startTracing("kata52-api");
// rồi start app
```

---

### Step 2 — Ensure HTTP propagation (W3C traceparent)

Auto-instrumentation thường đã làm phần lớn việc này, nhưng bạn vẫn phải:

* Đặt **span name theo route**
* Đảm bảo framework đưa `route` chuẩn (không raw path)

Ví dụ Fastify (gợi ý):

```ts
app.addHook("onRequest", async (req) => {
  // onRequest already under a span by auto-instrumentation
  // but route may not be known yet; better set name in onRoute/onResponse
});

app.addHook("onResponse", async (req, reply) => {
  const route = (req.routeOptions && req.routeOptions.url) ? req.routeOptions.url : "unknown";
  const method = req.method;

  // Set span name to normalized route
  // Use OpenTelemetry API to get active span and rename
  const { trace, context } = await import("@opentelemetry/api");
  const span = trace.getSpan(context.active());
  if (span) span.updateName(`HTTP ${method} ${route}`);
});
```

---

### Step 3 — Create explicit child spans for important operations

Auto-instrumentation tạo span HTTP server/client, nhưng Senior+ phải **thêm spans domain**:

Trong `api`, khi gọi `worker`:

```ts
import { trace } from "@opentelemetry/api";
import { request } from "undici";

const tracer = trace.getTracer("kata52");

async function callCharge() {
  return tracer.startActiveSpan("charge.request", async (span) => {
    try {
      const res = await request("http://localhost:3001/charge", { method: "POST" });
      span.setAttribute("downstream.service", "kata52-worker");
      span.setAttribute("http.status_code", res.statusCode);
      if (res.statusCode >= 500) {
        span.recordException(new Error(`worker_${res.statusCode}`));
        span.setStatus({ code: 2 }); // ERROR
      }
      return res;
    } catch (e) {
      span.recordException(e as Error);
      span.setStatus({ code: 2 });
      throw e;
    } finally {
      span.end();
    }
  });
}
```

> Note: `undici` auto-instrumentation sẽ tạo client span; bạn đang tạo “domain span” bao quanh để dễ đọc trace.

---

### Step 4 — Log correlation (trace_id/span_id trong log)

`src/logger.ts`:

```ts
import pino from "pino";
import { context, trace } from "@opentelemetry/api";

export const logger = pino({
  base: null,
  mixin() {
    const span = trace.getSpan(context.active());
    const sc = span?.spanContext();
    return sc
      ? { trace_id: sc.traceId, span_id: sc.spanId }
      : {};
  },
});
```

Trong handler: `logger.info({ route }, "checkout started")` → log sẽ kèm trace_id/span_id.

---

## 6) Tests (bắt buộc)

### Test 1 — Propagation: client gửi traceparent → services giữ traceId

Ý tưởng:

* Tạo một `traceparent` hợp lệ với traceId bạn chọn (32 hex), spanId (16 hex)
* Gọi `api /checkout` với header đó
* Trong response, bạn trả về `trace_id` hiện tại (chỉ cho kata/test), hoặc check qua logs capture (khó hơn)

Gợi ý thêm endpoint dev-only:

* `GET /debug/trace` trả `{ trace_id, span_id }` của request hiện tại
* Hoặc `GET /checkout` trả `x-trace-id` header (dev-only)

Ví dụ (dev-only) trong response:

```ts
reply.header("x-trace-id", sc?.traceId ?? "none");
```

Test (vitest):

* expect `x-trace-id` === traceId bạn inject
* và khi api gọi worker, worker cũng có cùng traceId (worker cũng trả `x-trace-id` về api; api forward lại)

### Test 2 — Parent/child correctness (span hierarchy)

Cách đơn giản:

* Dùng `InMemorySpanExporter` thay console exporter trong test mode
* Assert:

  * Có span server `HTTP GET /checkout`
  * Có span domain `charge.request`
  * Có span client HTTP sang worker
  * `charge.request` là child của server span (same traceId, parentSpanId đúng)

Gợi ý: cho phép tracing.ts nhận exporter từ env `OTEL_EXPORTER=inmemory` trong test.

---

## 7) Failure injection (bắt buộc)

Trong `worker`, thêm query:

* `POST /charge?fail=0.3&latency=200`

Yêu cầu chứng minh:

* Tăng latency → p95 trace duration tăng
* Tăng fail → span status = ERROR + recordException xuất hiện trong trace

---

## 8) Definition of Done (pass/fail rõ ràng)

Bạn PASS khi:

1. **End-to-end trace** chạy được: gọi `/checkout` thấy ít nhất:

   * server span ở api
   * domain span `charge.request`
   * client span call worker
   * server span ở worker
2. **Propagation đúng**:

   * Inject `traceparent` → `trace_id` giữ nguyên xuyên api → worker
3. **Context không rơi**:

   * Trong domain span, log có cùng `trace_id` với request
4. **No cardinality bombs**:

   * Span names dùng `route` normalized, không raw URL
5. **Test green**: propagation + hierarchy assertions.

---

## 9) Stretch (Senior++)

1. **Baggage propagation**: set `baggage` key (ví dụ `tenant_id`) và đảm bảo worker nhận được, nhưng chỉ allowlist & không log bừa.
2. **Sampling strategy**:

   * Head-based sampling: prod 1–5%, nhưng **always sample errors**
3. **Trace ↔ Metrics link**:

   * Add exemplars (nếu stack hỗ trợ) hoặc log trace_id khi latency vượt ngưỡng.
4. **Queue boundary** (hard mode):

   * Khi publish message, inject context vào message headers
   * Consumer extract + continue trace (span kind: CONSUMER/PRODUCER)