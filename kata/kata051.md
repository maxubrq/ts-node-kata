# Kata 51 — Golden Signals

*Instrument latency / traffic / errors / saturation cho HTTP service*

---

## 1. Goal (năng lực cần đạt)

Sau kata này, bạn **không còn “có metrics”** mà:

* Biết **đo đúng 4 Golden Signals**
* Biết **metric nào dùng để làm gì khi sự cố xảy ra**
* Biết **chọn label để không tự DDoS Prometheus**
* Có thể trả lời câu hỏi on-call:

  > “Service chậm vì đâu: traffic tăng, error tăng, hay saturation?”

---

## 2. Context (bài toán)

Bạn có một HTTP service Node.js (Fastify hoặc Express):

Routes:

* `GET /healthz`
* `GET /users/:id`
* `POST /orders`

Downstream giả lập:

* `fakeDb(ms)` → tạo latency
* `fakePayment(ms, failRate)` → tạo latency + error

Expose:

* `GET /metrics` (Prometheus scrape)

---

## 3. Constraints (bắt buộc – đây là chỗ Senior+)

1. **Latency**

   * Phải dùng **Histogram**
   * Có thể tính **p50 / p95 / p99**
   * Bucket phải hợp lý (không quá mịn)

2. **Labels**

   * ❌ Không dùng `userId`, `ip`, raw `path`
   * ✅ Dùng `route`, `method`, `status_class`
   * Cardinality phải **ổn định theo thời gian**

3. **Errors**

   * Phân biệt:

     * HTTP 5xx (server fault)
     * HTTP 4xx (client fault)
     * Domain error (`payment_declined`, `timeout`)

4. **Saturation**

   * Ít nhất **2 chỉ số**

     * Event loop lag
     * In-flight requests **hoặc** queue depth

5. **Chứng minh được**

   * Có **failure injection**
   * Có **load test**
   * Metrics phải phản ánh đúng hành vi hệ thống

---

## 4. Deliverables (những file phải có)

```
kata-51/
├─ src/
│  ├─ server.ts
│  ├─ metrics.ts
│  ├─ middleware/metrics.ts
│  ├─ downstream.ts
│  └─ eventLoopLag.ts
├─ test/
│  └─ metrics.spec.ts
├─ scripts/
│  └─ load.ts   (hoặc autocannon cmd)
└─ README.md    (queries + alerts + runbook)
```

---

## 5. Golden Signals – định nghĩa & cách đo

### 5.1 Latency

**Metric**

* `http_request_duration_seconds` (Histogram)

**Labels**

* `route` (ví dụ: `/users/:id`, không phải `/users/123`)
* `method`
* `status_class` (`2xx`, `4xx`, `5xx`)

**Buckets (gợi ý)**

```
[0.05, 0.1, 0.2, 0.3, 0.5, 0.75, 1, 1.5, 2, 3, 5]
```

**PromQL (p95)**

```promql
histogram_quantile(
  0.95,
  sum by (le, route)(
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

> Insight: Latency **tail** (p95/p99) mới là thứ làm user đau.

---

### 5.2 Traffic

**Metric**

* `http_requests_total` (Counter)

**Labels**

* `route`
* `method`
* `status_class`

**PromQL**

```promql
sum(rate(http_requests_total[1m]))
```

> Insight: Traffic tăng không xấu — **traffic + latency tăng** mới đáng lo.

---

### 5.3 Errors

**Metrics**

* `http_requests_total{status_class="5xx"}`
* `domain_errors_total{code="payment_declined"}`

**PromQL**

```promql
sum(rate(http_requests_total{status_class="5xx"}[5m]))
/
sum(rate(http_requests_total[5m]))
```

> Insight: **5xx là bug của mình**, 4xx không phải.

---

### 5.4 Saturation

#### a) Event loop lag

**Metric**

* `node_event_loop_lag_seconds` (Gauge / Histogram)

**Rule**

* > 100ms kéo dài ⇒ CPU / blocking

---

#### b) In-flight requests

**Metric**

* `http_in_flight_requests` (Gauge)

**Cách đo**

* `++` khi request bắt đầu
* `--` khi response kết thúc

> Insight: In-flight tăng + latency tăng = **kẹt cổ chai**

---

## 6. Instrumentation – skeleton bắt buộc

### `metrics.ts`

```ts
import { Histogram, Counter, Gauge, Registry } from "prom-client";

export const registry = new Registry();

export const httpDuration = new Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP request latency",
  labelNames: ["route", "method", "status_class"],
  buckets: [0.05, 0.1, 0.2, 0.3, 0.5, 0.75, 1, 1.5, 2, 3, 5],
});

export const httpRequests = new Counter({
  name: "http_requests_total",
  help: "Total HTTP requests",
  labelNames: ["route", "method", "status_class"],
});

export const inFlight = new Gauge({
  name: "http_in_flight_requests",
  help: "In-flight HTTP requests",
});

registry.registerMetric(httpDuration);
registry.registerMetric(httpRequests);
registry.registerMetric(inFlight);
```

---

### Metrics middleware (route-normalized)

```ts
export function metricsMiddleware(req, res, next) {
  const route = req.route?.path ?? "unknown";
  const method = req.method;

  inFlight.inc();
  const end = httpDuration.startTimer({ route, method });

  res.on("finish", () => {
    const statusClass = `${Math.floor(res.statusCode / 100)}xx`;
    httpRequests.inc({ route, method, status_class: statusClass });
    end({ status_class: statusClass });
    inFlight.dec();
  });

  next();
}
```

---

## 7. Failure injection (bắt buộc)

### `downstream.ts`

```ts
export async function fakePayment(ms: number, failRate: number) {
  await sleep(ms);
  if (Math.random() < failRate) {
    const err = new Error("payment_declined");
    // increment domain_errors_total here
    throw err;
  }
}
```

Bạn phải chứng minh:

* FailRate tăng ⇒ error rate tăng
* Latency tăng ⇒ p95/p99 tăng
* In-flight tăng ⇒ saturation phản ánh đúng

---

## 8. Tests (không được né)

### Runtime metric test

* Gọi `/users/:id`
* Scrape `/metrics`
* Assert:

  * `http_requests_total` tăng
  * `http_request_duration_seconds_bucket` có sample

---

## 9. Load test (chứng minh “tail”)

Ví dụ autocannon:

```bash
autocannon -c 50 -d 30 http://localhost:3000/orders
```

Bạn phải quan sát:

* p95 latency
* in-flight
* event loop lag

---

## 10. Alerts (Senior+ tối thiểu)

```yaml
- alert: HighLatencyP95
  expr: histogram_quantile(0.95,
        sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
        > 1.5
  for: 3m
```

```yaml
- alert: ErrorRateHigh
  expr: sum(rate(http_requests_total{status_class="5xx"}[5m]))
        / sum(rate(http_requests_total[5m])) > 0.02
  for: 2m
```

---

## 11. Definition of Done (không thương lượng)

* Đủ **4 Golden Signals**
* Không nổ cardinality
* Có p95/p99
* Có saturation thật (không giả)
* Có load test chứng minh
* Có alert chạy được

---

## 12. Stretch (Senior++ / Staff-level)

1. **RED vs USE mapping**: giải thích Golden Signals ↔ RED ↔ USE.
2. **Latency per dependency**: tách DB vs Payment.
3. **Adaptive buckets**: justify bucket selection theo SLO.
4. **Burn rate alert** (SLO-based).
5. **Tracing bridge**: link metric spike ↔ trace.