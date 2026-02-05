# Kata 53 — Log Sampling Strategy

*Sampling theo error + latency tail, giữ signal, giảm cost, vẫn debug được.*

---

## 1) Goal (năng lực cần đạt)

Sau kata này bạn làm được một hệ **log sampling production-grade** cho Node/TS service:

* **Giảm log volume** mạnh (10–100×) mà **không mất khả năng debug**.
* Luôn giữ đủ log cho:

  * **Errors** (5xx, exceptions) → gần như 100%
  * **Latency tail** (p95/p99, hoặc > threshold) → ưu tiên giữ
  * **Rare events** (payment declined, retries, circuit open…) → keep theo rule
* Sampling **deterministic theo request_id / trace_id** để:

  * Một request đã “được chọn” thì **giữ toàn bộ logs** của request đó (coherent).
* Có **metrics cho sampling** (dropped/kept), và **kill-switch** để tăng log khi incident.
* Có **tests** chứng minh rule hoạt động, không “random hand-wavy”.

---

## 2) Context (bài toán)

Bạn có HTTP service (Fastify/Express) đã có:

* **Golden Signals** (kata 51)
* **Tracing + context** (kata 52) → có `trace_id`

Routes:

* `GET /users/:id`
* `POST /orders`

Downstream giả lập có latency + fail.

Bạn đang bị:

* Log quá nhiều → tốn tiền, khó tìm.
* Nhưng nếu giảm log bừa → mất dấu khi incident.

---

## 3) Constraints (ép đúng kiểu Senior+)

1. **Không được** “sample 10% fixed” cho mọi thứ.
2. Error logs: **100%** (hoặc ~100% nếu flood-protection)
3. Latency tail: **ưu tiên giữ** theo rule rõ ràng (ví dụ `duration_ms >= 800`).
4. Sampling phải **deterministic** theo `trace_id` hoặc `request_id`:

   * Không dùng `Math.random()` cho quyết định chính (trừ fallback).
5. Must-have context fields trong mọi log:

   * `timestamp`, `level`, `msg`
   * `service`, `env`
   * `trace_id`, `span_id` (nếu có)
   * `request_id`
   * `route`, `method`, `status_code`, `duration_ms`
6. Có **metrics**:

   * `logs_kept_total{reason=...}`
   * `logs_dropped_total{reason=...}`
7. Có **kill switch**:

   * Env var / dynamic config để tăng sampling ngay lập tức (incident mode).

---

## 4) Deliverables (repo structure)

```
kata-53/
├─ src/
│  ├─ server.ts
│  ├─ logger/
│  │  ├─ index.ts          (pino logger + hooks)
│  │  ├─ sampler.ts        (policy engine)
│  │  ├─ decisions.ts      (reasons + rules)
│  │  └─ hash.ts           (deterministic sampling)
│  ├─ middleware/
│  │  └─ requestContext.ts (request_id/trace_id/duration)
│  └─ metrics.ts           (prom metrics kept/dropped)
├─ test/
│  ├─ sampler.spec.ts
│  └─ integration.spec.ts
└─ README.md               (policy, runbook)
```

---

## 5) Sampling policy (bạn implement đúng y vậy)

### 5.1 Event categories (reasons)

Bạn phải phân loại lý do giữ/dropping:

**KEEP reasons**

* `error` (exception, status 5xx)
* `tail_latency` (duration_ms >= TAIL_MS)
* `rare_event` (domain code: `payment_declined`, `retry_exhausted`, `circuit_open`)
* `debug_force` (header / env / allowlist)
* `baseline` (deterministic baseline sample)

**DROP reasons**

* `normal_traffic` (success fast, not selected)

---

### 5.2 Decision tree (thứ tự quan trọng)

**Rule order** (đừng đảo):

1. **Force keep**
   Nếu:

   * header `x-debug-log: 1`
   * hoặc `DEBUG_LOG=1`
   * hoặc `trace_id` nằm trong allowlist (incident)
     ⇒ keep `debug_force`

2. **Keep errors**
   Nếu:

   * `status_code >= 500`
   * hoặc có exception
     ⇒ keep `error`
     (Stretch: flood-protection theo key để tránh bị spam)

3. **Keep tail latency**
   Nếu `duration_ms >= TAIL_MS` (vd 800ms)
   ⇒ keep `tail_latency`

4. **Keep rare events**
   Nếu log có `event_code` thuộc allowlist
   ⇒ keep `rare_event`

5. **Baseline deterministic sampling**
   Nếu `hash(trace_id) % 10000 < BASE_RATE_BPS`

   * BASE_RATE_BPS ví dụ 100 = 1%
     ⇒ keep `baseline`

6. Else drop.

---

## 6) Deterministic sampling (bắt buộc)

Tạo `hash.ts`:

* Input: `trace_id` (hoặc `request_id` nếu thiếu trace)
* Output: integer 0..9999
* Dùng hash ổn định (FNV-1a 32-bit là đủ)

Khi baseline sample chọn request, bạn **giữ toàn bộ logs** của request đó (coherent logging).
Cách làm: trong request context lưu `sampled=true/false` từ đầu request.

---

## 7) Logger integration (pino)

Yêu cầu:

* Logger có thể **drop** log events dựa trên sampler decision.
* Nhưng vẫn **count metrics** dropped/kept.

Gợi ý cách làm (không bắt bạn đúng 1 cách, miễn đạt DoD):

* Dùng pino hooks (`hooks.logMethod`) để intercept
* Hoặc wrap logger API: `log.info(payload, msg)` → `sampler.decide(ctx, payload)`.

Mỗi log entry phải tự động mixin context:

* `trace_id`, `request_id`, `route`, `duration_ms` (khi response finish), `status_code`.

---

## 8) Metrics (để biết mình đang drop gì)

Expose Prometheus metrics:

* `logs_kept_total{reason, level}`
* `logs_dropped_total{reason, level}`
* `log_sampling_baseline_bps` (gauge)
* `log_sampling_tail_ms` (gauge)

Bạn sẽ dùng nó để tự trả lời:

* “Hôm nay drop 98% logs, nhưng vẫn giữ 100% errors + 100% tail. OK.”

---

## 9) Tests (bắt buộc)

### 9.1 Unit tests — sampler.spec.ts

Test các case:

1. **Force keep**

   * ctx.debug=true ⇒ keep, reason=debug_force

2. **Error**

   * status=500 ⇒ keep, reason=error
   * exception present ⇒ keep

3. **Tail latency**

   * duration_ms=1200 ⇒ keep, reason=tail_latency

4. **Rare events**

   * payload.event_code="payment_declined" ⇒ keep

5. **Baseline deterministic**

   * Với trace_id cố định, decision luôn giống nhau qua 100 runs
   * Với BASE_RATE_BPS=10000 ⇒ luôn keep
   * Với BASE_RATE_BPS=0 ⇒ luôn drop (nếu không thuộc rules khác)

6. **Drop normal**

   * status=200, duration=20ms, no event_code, baseline miss ⇒ drop

### 9.2 Integration tests — integration.spec.ts

* Spin server
* Gọi 1000 requests:

  * 950 requests fast success
  * 30 requests slow (inject latency)
  * 20 requests fail (inject 500)
* Assert:

  * Kept logs gần = (all errors + all slow + baseline portion)
  * Dropped logs chủ yếu là normal traffic
  * Metrics counters match (kept + dropped = total attempts)

---

## 10) Failure modes (bắt bạn nghĩ như production)

Bạn phải xử lý (ít nhất mô tả trong README + 1 test nếu làm được):

* **Log flood on error storm**: 100% errors có thể giết storage
  → Stretch: per-key rate limit (ví dụ theo `error_code+route`) vẫn giữ sample + counts.
* **Missing trace_id**: fallback to request_id.
* **Route unknown**: normalize “unknown”, không log raw path.
* **PII leak**: sampler không được dựa vào PII; logger phải redact fields.

---

## 11) Definition of Done (pass/fail rõ)

Bạn PASS khi:

1. Có policy engine đúng decision tree.
2. Sampling deterministic, request-coherent.
3. Errors và tail latency **luôn được giữ**.
4. Baseline sample chạy đúng rate.
5. Có metrics kept/dropped theo reason.
6. Test unit + integration xanh.

---

## 12) Stretch (Senior++)

1. **Dynamic sampling theo incident**
   * Khi error rate > X hoặc p95 > Y, tự tăng baseline bps (adaptive).
2. **Tail-based sampling theo trace**
   * Dựa vào tracing span duration để quyết định giữ logs/traces (tail sampling).
3. **Per-route budgets**
   * Route hot (`/healthz`) sample thấp hơn route quan trọng (`/orders`).
4. **Exemplar linking**
   * Khi request thuộc tail latency, log `trace_id` để mở trace nhanh.