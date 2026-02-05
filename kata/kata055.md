# Kata 55 — Health vs Readiness

*Phân biệt đúng “process sống” vs “sẵn sàng nhận traffic”, và test các kịch bản production.*

---

## 1) Goal (năng lực cần đạt)

Sau kata này bạn làm được:

* Thiết kế **2 endpoint chuẩn**:

  * **Liveness / health**: “process còn sống không?”
  * **Readiness**: “service có **đủ dependency** để nhận traffic không?”
* Có **startup gating** + **graceful degradation** đúng kiểu production.
* Có **test kịch bản**: DB down, queue lag, warmup chưa xong, shutdown đang diễn ra…
* Không tự bắn vào chân: readiness flapping, thundering herd, false positive.

---

## 2) Context (bài toán)

Bạn có Node/TS service có dependencies giả lập:

* DB connection (hoặc pool)
* Cache (optional)
* Downstream payment (optional)
* Background worker warmup (ví dụ load config / build in-memory index)

Bạn phải expose:

* `GET /healthz` (liveness)
* `GET /readyz` (readiness)
* `GET /livez` (optional tách rõ, nhưng kata yêu cầu ít nhất 2 endpoint)

---

## 3) Constraints (Senior+ bắt buộc)

1. **Health (liveness)**

   * Không được check dependency nặng (DB, payment) theo kiểu “ping mỗi lần”
   * Không block lâu
   * Chỉ phản ánh: event loop không bị kẹt chết, process không trong trạng thái “fatal”

2. **Readiness**

   * Phải dựa trên **readiness state machine** + cache kết quả check theo TTL
   * Có **timeout** cho dependency check
   * Có **degrade strategy**: dependency optional thì readiness vẫn OK nhưng set warning signal (stretch)

3. **State transitions**

   * `starting` → `ready`
   * `ready` → `not_ready` (dep down)
   * `not_ready` → `ready` (dep recovered)
   * `draining` (shutdown) → readiness phải trả NOT ready ngay

4. **Test**

   * Phải có test mô phỏng:

     * Startup warmup chưa xong
     * DB down rồi up
     * Dependency check timeout
     * Shutdown drain

---

## 4) Deliverables (repo)

```
kata-55/
├─ src/
│  ├─ server.ts
│  ├─ health/
│  │  ├─ state.ts        (state machine)
│  │  ├─ readiness.ts    (checkers + TTL cache)
│  │  ├─ liveness.ts     (fast checks)
│  │  └─ deps.ts         (fake deps + toggles)
│  └─ shutdown.ts        (drain + signal handlers)
├─ test/
│  ├─ readiness.spec.ts
│  ├─ liveness.spec.ts
│  └─ scenarios.spec.ts
└─ README.md             (contract + kịch bản + ops notes)
```

---

## 5) Spec (contract rõ ràng)

### 5.1 Liveness: `GET /healthz`

**Ý nghĩa**: “Process còn sống, event loop không bị kẹt chết, chưa vào trạng thái fatal.”

Response:

* `200` + JSON:

  * `status: "ok"`
  * `uptime_s`
  * `event_loop_lag_ms` (optional, cached)
  * `version`, `build_sha` (optional)

Rules:

* **Không gọi DB** trực tiếp.
* Nếu process đang `draining` (shutdown), liveness **vẫn 200** (process sống), nhưng readiness sẽ fail.

### 5.2 Readiness: `GET /readyz`

**Ý nghĩa**: “Service có thể nhận traffic và xử lý thành công với dependency hiện tại.”

Response:

* `200` nếu ready
* `503` nếu not ready
  Body gồm:
* `status: "ready" | "not_ready"`
* `state: "starting" | "ready" | "not_ready" | "draining"`
* `checks`: từng dependency `ok|fail|timeout|skipped` + `last_success_at`

Rules:

* Check dependency theo **TTL cache** (vd 2s) để tránh overload.
* Mỗi checker phải có **timeout** (vd 200ms).
* Nếu shutdown bắt đầu: state = `draining` và readiness trả `503` ngay.

---

## 6) Implementation plan (làm theo)

### Step 1 — State machine

`state.ts`:

* Enum: `STARTING`, `READY`, `NOT_READY`, `DRAINING`
* Atomic state (in-memory)
* Methods:

  * `setStarting()`, `setReady()`, `setNotReady(reason)`, `setDraining()`
  * `getState()`

Startup flow:

* init deps (connect DB, warm cache)
* only after success: `setReady()`

Shutdown flow:

* on SIGTERM: `setDraining()`, stop accept new requests, drain inflight, then exit.

---

### Step 2 — Readiness checkers + TTL cache

`readiness.ts`:

* `checkDb(): Promise<CheckResult>`
* `checkQueue(): Promise<CheckResult>` (optional)
* TTL cache:

  * `lastResult`, `lastCheckedAt`
  * If now - lastCheckedAt < TTL → reuse
* Aggregation:

  * If any **required** check fail/timeout → NOT READY

> Senior+ point: readiness endpoint có thể bị kube gọi liên tục. Không cache là tự đốt DB.

---

### Step 3 — Liveness (fast, no deps)

`liveness.ts`:

* uptime
* event loop lag: đọc từ metric cached (đã làm ở kata 16), hoặc đo nhanh nhưng không block.

---

### Step 4 — Wiring routes

* `/healthz` gọi liveness
* `/readyz` gọi readiness aggregator
* `/readyz` phải return 503 khi `state === DRAINING` hoặc `STARTING`

---

## 7) Scenario tests (bắt buộc)

### Scenario A — Startup warmup chưa xong

* Start server với `warmupDelay=2000ms`
* Immediately call:

  * `/healthz` → **200**
  * `/readyz` → **503** với `state=starting`
* After 2s:

  * `/readyz` → **200** `state=ready`

### Scenario B — DB down rồi up

* Fake DB có toggle `db.setHealthy(false)`
* When down:

  * `/readyz` → **503**, check `db=fail`
  * `/healthz` → **200**
* When up:

  * `/readyz` → **200** sau 1–2 vòng TTL

### Scenario C — Dependency check timeout

* Fake DB “hang” > timeout (vd 500ms)
* `/readyz` phải trả trong < timeout + epsilon
* Status: `db=timeout`, overall **503**

### Scenario D — Shutdown draining

* Send SIGTERM (hoặc call internal `beginShutdown()`)
* Expect:

  * `/readyz` → **503** ngay, `state=draining`
  * `/healthz` → **200** vẫn ok (process còn sống)
* Inflight requests được drain (stretch kiểm bằng hook)

---

## 8) Definition of Done (pass/fail)

Bạn PASS khi:

1. `/healthz` luôn nhanh, không check deps, không bị 503 do DB down.
2. `/readyz` phản ánh dependency & state machine đúng.
3. Có TTL cache + timeout cho readiness checks.
4. Có tests cho 4 scenario A–D chạy xanh.
5. README mô tả contract + lý do tách health vs readiness.

---

## 9) Stretch (Senior++)

1. **Optional deps & degraded-ready**
   * DB required, payment optional: readiness vẫn 200 nhưng `degraded=true` + emit metric `readiness_degraded_total`.
2. **Readiness flapping protection**
   * Require N successes before flip to READY; N fails before NOT_READY.
3. **Outage-safe behavior**
   * Khi DB down: vẫn serve read-only endpoints? (route-level readiness, advanced)
4. **Kubernetes probe tuning notes**
   * initialDelaySeconds, periodSeconds, failureThreshold hợp lý theo warmup.
