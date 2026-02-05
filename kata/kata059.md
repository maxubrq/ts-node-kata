# Kata 59 — Incident Repro Harness

*Script tái hiện sự cố deterministically: replay traffic + faults + timings, chứng minh fix thật sự giải quyết vấn đề.*

---

## 1) Goal (năng lực cần đạt)

Sau kata này bạn có một “incident harness” kiểu production:

* Có thể **tái hiện sự cố** bằng 1 lệnh: latency spike, error storm, queue backlog, timeout cascade…
* Tái hiện **deterministic** (seed cố định) để:

  * bug “lúc được lúc không” biến thành “lúc nào cũng được”
* Có thể chạy:

  * **baseline** (no fault)
  * **faulted** (inject faults)
  * **compare** trước/sau fix (đo metrics/logs/traces)
* Tự động **assert**: incident đã xảy ra (p95 vượt ngưỡng, error rate vượt ngưỡng, readiness fail…)
* Xuất **artifact**: report JSON + log bundle để attach postmortem.

---

## 2) Context (bài toán)

Bạn dùng stack từ tầng 6 (ít nhất):

* Service API + Worker (kata 58) hoặc chỉ API (kata 51–55)
* Metrics + logs + tracing + sampling
* Có các “fault knobs” (bạn phải thêm nếu chưa có):

  * `LATENCY_MS`, `FAIL_RATE`, `TIMEOUT_MS`, `QUEUE_DELAY_MS`
  * hoặc endpoint admin dev-only: `/__faults`

Bạn sẽ build harness chạy local (docker-compose).

---

## 3) Constraints (Senior+ bắt buộc)

1. **Deterministic**

   * Không dùng `Math.random()` trực tiếp
   * Tất cả randomness phải qua RNG có `seed`
2. **Time control**

   * Incident có phases (warmup → trigger → cooldown)
   * Mỗi phase có duration rõ
3. **Fault injection có mục tiêu**

   * “Fail payment 20%”, “Add 800ms to DB for 10% calls”, “Queue slow consumer”
4. **Observation**

   * Harness phải scrape/collect:

     * p50/p95/p99
     * error rate
     * saturation indicators (event loop lag / in-flight / queue depth)
   * Không cần Prometheus thật nhưng phải có endpoint `/metrics` để parse.
5. **Assertions**

   * Harness phải fail CI nếu incident *không tái hiện* hoặc *fix không còn hiệu lực*.
6. **Artifacts**

   * Tạo file report: `artifacts/incident-<name>-<ts>.json`
   * Kèm raw samples (latencies/errors) + config seed.

---

## 4) Deliverables (repo)

```
kata-59/
├─ docker-compose.yml
├─ services/ (hoặc packages/ từ kata trước)
│  ├─ api/
│  └─ worker/
├─ harness/
│  ├─ src/
│  │  ├─ main.ts          (CLI)
│  │  ├─ rng.ts           (seeded RNG)
│  │  ├─ scenarios/
│  │  │  ├─ latency-tail.ts
│  │  │  ├─ error-storm.ts
│  │  │  ├─ timeout-cascade.ts
│  │  │  └─ queue-backlog.ts
│  │  ├─ loadgen.ts       (traffic generator)
│  │  ├─ faults.ts        (set/reset fault knobs)
│  │  ├─ metrics.ts       (scrape + compute p50/p95/p99)
│  │  ├─ assertions.ts
│  │  └─ artifacts.ts     (write report bundle)
│  └─ test/
│     └─ harness.spec.ts  (runs one scenario quickly)
└─ README.md              (how to add new scenario)
```

---

## 5) Interface contract (bạn phải đảm bảo service hỗ trợ)

Service(s) phải có:

1. `GET /metrics` (Prometheus text)
2. `POST /__faults` (dev-only, guarded) nhận JSON:

   ```json
   {
     "db_latency_ms_add": 0,
     "db_latency_tail_pct": 0.1,
     "payment_fail_rate": 0.2,
     "payment_latency_ms_add": 300,
     "worker_delay_ms": 0,
     "force_timeout": false
   }
   ```
3. `POST /__faults/reset` để về baseline
4. (Optional) `GET /readyz`, `GET /healthz`

**Constraint**: `__faults` chỉ chạy local/dev; production không được expose (liên quan kata 57).

---

## 6) Harness CLI spec

CLI usage:

```bash
pnpm harness run latency-tail --seed 1337 --duration 60s --qps 50
pnpm harness run timeout-cascade --seed 42 --duration 90s --qps 30
pnpm harness compare latency-tail --seed 1337 --baseline-tag before --candidate-tag after
```

Output:

* exit code 0 nếu pass
* exit code 1 nếu assertions fail
* viết artifact JSON

---

## 7) Scenario design (4 scenario tối thiểu)

Mỗi scenario phải có:

* name
* seed
* phases: warmup/trigger/cooldown
* fault profile
* expected outcomes (assertions)

### Scenario A — `latency-tail`

**Trigger**

* 10% requests thêm +1500ms ở payment
* baseline QPS ổn định

**Expected**

* p95 > 800ms trong trigger
* p50 vẫn < 150ms
* log sampling giữ tail requests (nếu tích hợp kata 53)

### Scenario B — `error-storm`

**Trigger**

* payment_fail_rate 20%

**Expected**

* 5xx rate > 2% (hoặc domain errors spike)
* alert condition (kata 56) “đúng ngưỡng” (optional)

### Scenario C — `timeout-cascade`

**Trigger**

* db latency +800ms
* api timeout 300ms
* retries enabled (kata 81 sau này) → tạo cascade

**Expected**

* error rate tăng
* in-flight tăng
* event loop lag tăng (nếu CPU bound)
* readiness có thể flip not-ready (nếu required dep)

### Scenario D — `queue-backlog` (nếu có worker)

**Trigger**

* worker_delay_ms = 500ms
* producer QPS 50

**Expected**

* queue depth tăng
* consumer processing latency tăng
* DLQ/retry metrics tăng (nếu poison injection)

---

## 8) Traffic generator (loadgen) — yêu cầu kỹ thuật

* Deterministic request pattern theo seed:

  * route mix: 70% `/users`, 30% `/orders`
  * payload variations deterministic
* Concurrency control:

  * stable QPS (token bucket) hoặc fixed arrival schedule
* Record per-request:

  * timestamp
  * route
  * status
  * latency_ms
  * correlation_id/trace_id (nếu có)

Lưu samples vào artifact.

---

## 9) Metrics scrape + quantiles (p50/p95/p99)

Harness phải:

* scrape `/metrics` định kỳ (vd mỗi 2s)
* parse histogram buckets từ:

  * `http_request_duration_seconds_bucket` hoặc `dep_call_duration_seconds_bucket`
* compute p50/p95/p99 bằng `histogram_quantile` (bạn có thể:

  * gọi PromQL engine (không cần)
  * hoặc tự implement quantile approx từ buckets — đủ cho kata)

**Senior+ requirement**: kèm sanity:

* count tăng đúng (traffic guard)
* nếu count ~0 thì skip assertions.

---

## 10) Assertions (bắt buộc)

Ví dụ assert schema:

* `during(trigger): p95_http_orders_ms > 800`
* `during(trigger): error_rate_5xx > 0.02`
* `during(warmup): p95 < 300` (đảm bảo baseline sạch)
* `after(cooldown): p95 back to normal`

Nếu fail:

* print diff + pointer tới artifact file.

---

## 11) Artifact report (bắt buộc)

File JSON gồm:

* scenario meta: name, seed, qps, duration, timestamps
* fault profile từng phase
* summary:

  * p50/p95/p99 per phase
  * error rate per phase
  * saturation metrics per phase
* samples (có thể truncate):

  * top 50 slowest requests
  * top error signatures (redacted)
* links/hints:

  * “grep trace_id …”
  * “open dashboard query …” (nếu có)

---

## 12) Definition of Done (pass/fail)

PASS khi:

1. Chạy `harness run latency-tail --seed 1337` tái hiện tail **mỗi lần**.
2. Harness tạo artifact JSON + summary.
3. Assertions fail đúng khi:

   * baseline không đủ traffic
   * fault không gây impact như expected
4. Có ít nhất 2 scenario chạy được (A + B), tốt nhất đủ 4.

---

## 13) Stretch (Senior++)

1. **Regression gate**: `compare before/after` và assert “after improved”

   * p95 giảm ≥ X%
   * error rate giảm ≥ Y%
2. **Replay exact requests**:

   * record request schedule → replay y hệt
3. **Trace-based validation**:

   * lấy exemplar trace_id từ slowest requests, auto fetch trace (nếu có backend)
4. **Chaos matrix**:

   * combine 2 faults (latency + error) theo matrix, auto-run nightly.
