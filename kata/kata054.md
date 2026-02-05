# Kata 54 — Custom Metrics: Histogram p50/p95/p99

*Thiết kế histogram đúng, xuất metric đúng, query đúng, chứng minh p50/p95/p99 phản ánh tail latency thật.*

---

## 1) Goal (năng lực cần đạt)

Sau kata này bạn làm được:

* Tạo **custom histogram** cho latency (hoặc domain duration) với **buckets có lý do**.
* Tính **p50/p95/p99** đúng cách từ histogram (PromQL `histogram_quantile`).
* Tránh sai lầm phổ biến: percentiles “ảo”, bucket sai, label nổ cardinality.
* Có **benchmark / load test + failure injection** để chứng minh tail behavior.
* Có **dash-ready queries** + **alert threshold** dựa trên p95/p99.

---

## 2) Context (bài toán)

Bạn có service Node/TS có 2 loại latency:

1. **HTTP request latency** (route-level)
2. **Downstream dependency latency** (ví dụ DB / Payment) — domain metric

Bạn sẽ instrument cả hai, nhưng trọng tâm kata là **domain histogram**:

* `dep_call_duration_seconds{dep="db|payment", op="getUser|charge", outcome="ok|error|timeout"}`

Downstream giả lập:

* `fakeDb(latencyMs)`
* `fakePayment(latencyMs, failRate)`

---

## 3) Constraints (Senior+ bắt buộc)

1. **Histogram bắt buộc**, không dùng summary để “cho nhanh”.
2. Buckets phải:

   * Có **thang đo hợp lý** (theo SLO/latency expected)
   * Không quá ít (mất tail), không quá nhiều (tốn memory/CPU)
3. Labels phải low-cardinality:

   * ✅ `dep`, `op`, `outcome`
   * ❌ `userId`, raw URL, error message
4. p50/p95/p99 phải được tính bằng PromQL, không tự tính trong app.
5. Phải có test chứng minh:

   * Khi inject latency, p95/p99 tăng theo expected
   * Khi traffic mix thay đổi (10% slow), p95/p99 phản ánh tail
6. Có metric cho **count + sum** (histogram tự có), và bạn phải dùng chúng để sanity-check.

---

## 4) Deliverables (repo)

```
kata-54/
├─ src/
│  ├─ server.ts
│  ├─ metrics.ts
│  ├─ downstream.ts
│  └─ routes.ts
├─ test/
│  ├─ buckets.spec.ts
│  └─ quantiles.spec.ts
├─ scripts/
│  └─ load.ts (hoặc autocannon)
└─ README.md (bucket rationale + queries + alerts)
```

---

## 5) Metrics design (bạn implement đúng tên/labels)

### 5.1 Histogram: dependency call duration

Tên metric (Prometheus convention):

* `dep_call_duration_seconds` (Histogram)

Labels:

* `dep`: `"db" | "payment"`
* `op`: `"getUser" | "createOrder" | "charge"`
* `outcome`: `"ok" | "error" | "timeout"`

Buckets (gợi ý theo “service typical”):

* 5ms → 5s, thiên về vùng nhỏ, vẫn đủ tail:

```
[0.005, 0.01, 0.02, 0.03, 0.05, 0.075, 0.1, 0.2, 0.3, 0.5, 0.75, 1, 2, 3, 5]
```

> Bạn phải viết trong README: “SLO kỳ vọng của dep call là gì” và bucket map theo đó.

---

### 5.2 Optional: Histogram cho HTTP (route-level)

* `http_request_duration_seconds{route,method,status_class}`
  (Cái này Kata 51 đã có, ở đây chỉ thêm nếu muốn so sánh.)

---

## 6) Implementation steps

### Step 1 — Create histogram

`src/metrics.ts` (prom-client)

* export:

  * `depDuration` histogram
  * `registry` và endpoint `/metrics`

### Step 2 — Wrap dependency calls bằng timer chuẩn

Bạn tạo helper:

* `withDepTimer({dep, op}, fn)`:

  * start timer
  * try await fn
  * set `outcome`
  * observe duration
  * record exceptions nếu cần (nhưng không label theo message)

Pseudo contract:

* timeout => `outcome="timeout"`
* throw => `outcome="error"`
* success => `outcome="ok"`

### Step 3 — Inject traffic mix để tạo tail

Ví dụ trong route `/orders`:

* 90% payment latency 50ms
* 10% payment latency 1200ms (tail)
* failRate 2%

Bạn sẽ thấy:

* p50 ~ 50ms
* p95/p99 nhảy lên gần tail

---

## 7) PromQL queries (bắt buộc ghi trong README)

### p50 / p95 / p99 cho payment.charge

```promql
histogram_quantile(
  0.50,
  sum by (le) (rate(dep_call_duration_seconds_bucket{dep="payment",op="charge"}[5m]))
)
```

```promql
histogram_quantile(
  0.95,
  sum by (le) (rate(dep_call_duration_seconds_bucket{dep="payment",op="charge"}[5m]))
)
```

```promql
histogram_quantile(
  0.99,
  sum by (le) (rate(dep_call_duration_seconds_bucket{dep="payment",op="charge"}[5m]))
)
```

### Sanity checks (Senior+)

**Avg latency** (đừng nhầm với p95):

```promql
sum(rate(dep_call_duration_seconds_sum{dep="payment",op="charge"}[5m]))
/
sum(rate(dep_call_duration_seconds_count{dep="payment",op="charge"}[5m]))
```

**Traffic**:

```promql
sum(rate(dep_call_duration_seconds_count{dep="payment",op="charge"}[1m]))
```

---

## 8) Tests (bắt buộc)

### 8.1 buckets.spec.ts — bucket rationale guard

* Assert bucket array monotonic increasing
* Assert có bucket <= 10ms, và >= 2s (tùy spec)
* Assert bucket count không vượt 20 (guard cost)

### 8.2 quantiles.spec.ts — tail correctness bằng deterministic dataset

Bạn làm theo hướng “không cần Prometheus thật”:

* Record N observations vào histogram:

  * 900 samples @ 0.05s
  * 100 samples @ 1.2s
* Scrape metric text, parse bucket counts
* Tự tính approximate quantile từ buckets (approx) và assert:

  * p50 nằm gần 0.05
  * p95 nằm giữa 0.75–1.2 (tùy bucket)
  * p99 gần 1.2

> Đây là điểm Senior+: hiểu histogram quantile là **approx**, phụ thuộc buckets.

---

## 9) Load test (chứng minh bằng thực nghiệm)

`autocannon -c 50 -d 30 http://localhost:3000/orders`

Bạn phải quan sát:

* `dep_call_duration_seconds_bucket` tăng đúng
* p95/p99 tăng mạnh khi bật tail mode

---

## 10) Failure modes phải xử lý/ghi rõ

* Bucket sai → p99 “đơ” hoặc nhảy vô lý (bạn phải mô tả dấu hiệu)
* Label explosion (op/dep không bounded) → memory blow
* Low traffic → quantile rung (nên dùng window 5m, không 30s)

---

## 11) Definition of Done

PASS khi:

1. Có `dep_call_duration_seconds` histogram đúng labels.
2. Có helper wrap dependency và set outcome đúng.
3. README có bucket rationale + p50/p95/p99 queries + sanity queries.
4. Tests chứng minh tail mix làm p95/p99 tăng.
5. Load test tái hiện tail trong runtime metrics.

---

## 12) Stretch (Senior++)

1. **Per-route / per-dep SLO buckets**: db khác payment.
2. **Exemplars** (nếu stack hỗ trợ) link metric spike ↔ trace_id.
3. **Alert theo burn rate** (kết hợp error rate + p99).
4. **Multi-dimensional rollups**: p95 theo `op` nhưng có guard tránh high cardinality.