# Kata 43 — Circuit Breaker (Open / Half-open / Close + Metrics)

## Goal

1. Implement **circuit breaker chuẩn production** với 3 trạng thái: **CLOSED → OPEN → HALF_OPEN → CLOSED**.
2. Có **failure detection** theo cửa sổ thời gian (rolling window) hoặc consecutive failures.
3. Có **probe policy** ở HALF_OPEN (cho phép N trial).
4. Có **metrics + events** để quan sát và debug: state transitions, reject count, latency, error ratio, probe success/fail.
5. Hỗ trợ **timeout + AbortSignal** (deadline propagation kiểu “đúng đời”).

---

### Context

Bạn gọi một downstream (payments/search/3rd-party). Khi downstream lỗi hoặc chậm:

* retry storm làm hệ tự giết mình
* saturation (pool cạn, event-loop lag) lan truyền sang endpoints khác
* tail latency tăng cực mạnh

Circuit breaker là cái “cầu dao”: khi downstream hỏng, **ngắt nhanh** để hệ còn thở; khi có tín hiệu hồi phục, **thử lại có kiểm soát**.

---

### Constraints

* ❌ Không dùng thư viện circuit breaker có sẵn.
* ✅ TypeScript strict, không `any`.
* ✅ Có 2 chế độ failure detection (chọn 1 để làm core, cái còn lại stretch):

  * **Rolling window**: errorRate >= threshold, minSamples
  * **Consecutive failures**: fail liên tiếp >= N
* ✅ Có `timeoutMs` cho mỗi call (nếu không xong trong timeout → tính là failure).
* ✅ Có `AbortSignal` để caller hủy và breaker vẫn release đúng.
* ✅ Có test deterministic: fake timers + clock injection.
* ✅ Metrics snapshot + event hooks.

---

### Deliverables

1. `src/circuitBreaker.ts` — implementation
2. `src/errors.ts` — error types (`CircuitOpenError`, `TimeoutError`)
3. `src/downstream.ts` — fake downstream (delay/fail)
4. `test/circuitBreaker.test.ts` — unit + timing + transition tests
5. `README.md` — semantics + trade-offs + failure modes

---

## API you will build

```ts
export type BreakerState = "CLOSED" | "OPEN" | "HALF_OPEN";

export type CircuitBreakerOptions = {
  name: string;

  // Failure detector (pick one as core)
  mode: "rolling-window" | "consecutive";

  // rolling window
  windowMs?: number;          // e.g. 10_000
  minSamples?: number;        // e.g. 20
  errorRateToOpen?: number;   // 0..1, e.g. 0.5

  // consecutive
  consecutiveFailuresToOpen?: number; // e.g. 5

  // state timing
  openDurationMs: number;     // e.g. 5_000 (sleep window)
  halfOpenMaxTrials: number;  // e.g. 3

  // per-call
  timeoutMs?: number;         // optional default timeout
};

export type BreakerMetricsSnapshot = {
  state: BreakerState;
  openedTotal: number;
  halfOpenedTotal: number;
  closedTotal: number;

  allowedTotal: number;       // calls passed into downstream
  rejectedTotal: number;      // short-circuited
  successTotal: number;
  failureTotal: number;
  timeoutTotal: number;

  inflight: number;

  // detector stats (best-effort)
  rolling?: { samples: number; failures: number; errorRate: number };
  consecutive?: { failures: number };

  lastStateChangeAt: number;
};

export class CircuitBreaker {
  constructor(opts: CircuitBreakerOptions, deps?: {
    now?: () => number;
    rand?: () => number;
  });

  exec<T>(
    fn: (ctx: { signal: AbortSignal }) => Promise<T>,
    opts?: { signal?: AbortSignal; timeoutMs?: number }
  ): Promise<T>;

  getState(): BreakerState;
  snapshot(): BreakerMetricsSnapshot;

  on(event: "state_change" | "reject" | "success" | "failure" | "timeout", cb: (e: any) => void): () => void;
}
```

---

## Core semantics (phải đúng)

### CLOSED

* Cho phép call đi qua.
* Thu thập kết quả (success/failure/timeout).
* Nếu vượt ngưỡng (rolling window) hoặc fail liên tiếp đủ N → chuyển **OPEN**.

### OPEN

* **Reject ngay** (fail fast) với `CircuitOpenError`.
* Sau `openDurationMs` kể từ khi OPEN → chuyển **HALF_OPEN** (tự động khi có call tới).

### HALF_OPEN

* Chỉ cho phép tối đa `halfOpenMaxTrials` calls “probe” đồng thời (hoặc tuần tự; bạn chọn, nhưng phải consistent).
* Nếu **probe success đủ** (ví dụ: tất cả trials thành công, hoặc success >= threshold) → **CLOSE** và reset stats.
* Nếu **bất kỳ probe fail** (simple policy) → quay lại **OPEN** (thường dùng nhất và dễ chứng minh).

> Senior+ note: HALF_OPEN là nơi dễ bug nhất: race conditions làm “leak trials” hoặc chuyển state sai.

---

## Error taxonomy (production-grade)

* `CircuitOpenError`: bị chặn vì breaker OPEN hoặc HALF_OPEN hết trial.
* `TimeoutError`: call vượt timeout.
* Downstream error: giữ nguyên (để upper layer quyết định retry/saga…).

Quan trọng: **timeout được tính là failure** cho breaker.

---

## Starter skeleton (điền TODO)

```ts
// src/errors.ts
export class CircuitOpenError extends Error {
  name = "CircuitOpenError";
}
export class TimeoutError extends Error {
  name = "TimeoutError";
}
```

```ts
// src/circuitBreaker.ts
import { CircuitOpenError, TimeoutError } from "./errors";

export type BreakerState = "CLOSED" | "OPEN" | "HALF_OPEN";

type Listener = (e: any) => void;

type RollingSample = { t: number; ok: boolean };

export type CircuitBreakerOptions = {
  name: string;
  mode: "rolling-window" | "consecutive";
  windowMs?: number;
  minSamples?: number;
  errorRateToOpen?: number;
  consecutiveFailuresToOpen?: number;
  openDurationMs: number;
  halfOpenMaxTrials: number;
  timeoutMs?: number;
};

export class CircuitBreaker {
  private state: BreakerState = "CLOSED";
  private lastStateChangeAt: number;

  private inflight = 0;

  // rolling-window
  private samples: RollingSample[] = [];

  // consecutive
  private consecutiveFailures = 0;

  // half-open
  private halfOpenInUse = 0;

  // metrics
  private openedTotal = 0;
  private halfOpenedTotal = 0;
  private closedTotal = 0;

  private allowedTotal = 0;
  private rejectedTotal = 0;
  private successTotal = 0;
  private failureTotal = 0;
  private timeoutTotal = 0;

  private listeners = new Map<string, Set<Listener>>();

  constructor(
    private opts: CircuitBreakerOptions,
    private deps: { now?: () => number; rand?: () => number } = {}
  ) {
    this.lastStateChangeAt = this.now();
  }

  private now() {
    return this.deps.now?.() ?? Date.now();
  }

  on(event: "state_change" | "reject" | "success" | "failure" | "timeout", cb: Listener) {
    let set = this.listeners.get(event);
    if (!set) {
      set = new Set();
      this.listeners.set(event, set);
    }
    set.add(cb);
    return () => set!.delete(cb);
  }

  private emit(event: string, e: any) {
    const set = this.listeners.get(event);
    if (!set) return;
    for (const cb of set) cb(e);
  }

  getState(): BreakerState {
    return this.state;
  }

  snapshot() {
    const base = {
      state: this.state,
      openedTotal: this.openedTotal,
      halfOpenedTotal: this.halfOpenedTotal,
      closedTotal: this.closedTotal,
      allowedTotal: this.allowedTotal,
      rejectedTotal: this.rejectedTotal,
      successTotal: this.successTotal,
      failureTotal: this.failureTotal,
      timeoutTotal: this.timeoutTotal,
      inflight: this.inflight,
      lastStateChangeAt: this.lastStateChangeAt,
    };

    if (this.opts.mode === "rolling-window") {
      const { samples, failures, errorRate } = this.rollingStats();
      return { ...base, rolling: { samples, failures, errorRate } };
    }
    return { ...base, consecutive: { failures: this.consecutiveFailures } };
  }

  async exec<T>(
    fn: (ctx: { signal: AbortSignal }) => Promise<T>,
    callOpts?: { signal?: AbortSignal; timeoutMs?: number }
  ): Promise<T> {
    // TODO: transition OPEN->HALF_OPEN when openDuration elapsed (lazy on request)
    this.maybeAdvanceState();

    // gate
    if (this.state === "OPEN") {
      this.rejectedTotal++;
      this.emit("reject", { name: this.opts.name, state: this.state });
      throw new CircuitOpenError(`Circuit OPEN: ${this.opts.name}`);
    }

    if (this.state === "HALF_OPEN") {
      if (this.halfOpenInUse >= this.opts.halfOpenMaxTrials) {
        this.rejectedTotal++;
        this.emit("reject", { name: this.opts.name, state: this.state });
        throw new CircuitOpenError(`Circuit HALF_OPEN (no trials left): ${this.opts.name}`);
      }
      this.halfOpenInUse++;
    }

    this.allowedTotal++;
    this.inflight++;

    const ac = new AbortController();
    const onAbort = () => ac.abort();
    if (callOpts?.signal) {
      if (callOpts.signal.aborted) onAbort();
      else callOpts.signal.addEventListener("abort", onAbort, { once: true });
    }

    const timeoutMs = callOpts?.timeoutMs ?? this.opts.timeoutMs;
    let timeoutId: NodeJS.Timeout | undefined;

    try {
      const p = fn({ signal: ac.signal });

      const raced =
        timeoutMs !== undefined
          ? await Promise.race([
              p,
              new Promise<never>((_, reject) => {
                timeoutId = setTimeout(() => reject(new TimeoutError(`Timeout ${this.opts.name}`)), timeoutMs);
              }),
            ])
          : await p;

      this.recordSuccess();
      this.emit("success", { name: this.opts.name });
      return raced as T;
    } catch (e) {
      const err = e instanceof Error ? e : new Error(String(e));
      if (err instanceof TimeoutError) {
        this.timeoutTotal++;
        this.emit("timeout", { name: this.opts.name });
      }
      this.recordFailure(err);
      this.emit("failure", { name: this.opts.name, error: err.name });
      throw err;
    } finally {
      if (timeoutId) clearTimeout(timeoutId);
      if (callOpts?.signal) callOpts.signal.removeEventListener("abort", onAbort);

      this.inflight--;

      if (this.state === "HALF_OPEN") {
        // release trial slot
        this.halfOpenInUse = Math.max(0, this.halfOpenInUse - 1);
      }
    }
  }

  private maybeAdvanceState() {
    if (this.state !== "OPEN") return;
    const t = this.now();
    if (t - this.lastStateChangeAt >= this.opts.openDurationMs) {
      this.setState("HALF_OPEN");
      // reset half-open probe counters
      this.halfOpenInUse = 0;
    }
  }

  private setState(next: BreakerState) {
    if (this.state === next) return;
    this.state = next;
    this.lastStateChangeAt = this.now();

    if (next === "OPEN") this.openedTotal++;
    if (next === "HALF_OPEN") this.halfOpenedTotal++;
    if (next === "CLOSED") this.closedTotal++;

    this.emit("state_change", { name: this.opts.name, state: next });
  }

  private recordSuccess() {
    this.successTotal++;
    if (this.opts.mode === "rolling-window") {
      this.samples.push({ t: this.now(), ok: true });
      this.gcRolling();
      // in CLOSED: check open condition; in HALF_OPEN: close policy
      this.evaluateAfterResult(true);
      return;
    }

    // consecutive mode
    this.consecutiveFailures = 0;
    this.evaluateAfterResult(true);
  }

  private recordFailure(_err: Error) {
    this.failureTotal++;
    if (this.opts.mode === "rolling-window") {
      this.samples.push({ t: this.now(), ok: false });
      this.gcRolling();
      this.evaluateAfterResult(false);
      return;
    }

    this.consecutiveFailures++;
    this.evaluateAfterResult(false);
  }

  private evaluateAfterResult(ok: boolean) {
    // TODO: implement transitions:
    // - if CLOSED and threshold breached -> OPEN
    // - if HALF_OPEN:
    //    - if any failure -> OPEN (simple policy)
    //    - if enough successes (e.g. trials completed and all ok) -> CLOSED + reset stats
  }

  private gcRolling() {
    const w = this.opts.windowMs ?? 10_000;
    const cutoff = this.now() - w;
    // drop old samples
    while (this.samples.length > 0 && this.samples[0].t < cutoff) this.samples.shift();
  }

  private rollingStats() {
    const samples = this.samples.length;
    const failures = this.samples.reduce((acc, s) => acc + (s.ok ? 0 : 1), 0);
    const errorRate = samples === 0 ? 0 : failures / samples;
    return { samples, failures, errorRate };
  }
}
```

---

## Checks (Definition of Done)

1. **OPEN rejects fast**

* Force OPEN, call exec => throws `CircuitOpenError` immediately (no downstream call).

2. **CLOSED → OPEN based on detector**

* Rolling: within window, samples>=minSamples and errorRate>=threshold → OPEN.
* Consecutive: failures>=N → OPEN.

3. **OPEN → HALF_OPEN after openDuration**

* Advance fake clock vượt `openDurationMs`, call exec => state becomes HALF_OPEN and allows probes.

4. **HALF_OPEN probe gating**

* halfOpenMaxTrials=2, start 3 exec concurrently => 3rd bị `CircuitOpenError`.

5. **HALF_OPEN failure snaps back to OPEN**

* One probe fails => OPEN ngay, subsequent calls reject.

6. **HALF_OPEN success closes**

* Policy đơn giản: “khi đã chạy đủ K probes và tất cả success” => CLOSED + reset stats.

7. **Timeout counts as failure**

* timeoutMs small + downstream delay => TimeoutError thrown, breaker records failure, can open.

8. **Metrics sanity**

* openedTotal/halfOpenedTotal/closedTotal increments đúng
* rejectedTotal increments đúng
* inflight không âm, max inflight bounded (bạn có thể track thêm nếu muốn)

---

## Test plan tối thiểu (vitest + fake timers)

### Test 1 — consecutive opens

* mode=consecutive, N=3
* downstream luôn throw
* call 3 lần => state OPEN
* call lần 4 => CircuitOpenError (no downstream hit)

### Test 2 — open then half-open then close

* openDuration=1000, halfOpenMaxTrials=2
* làm OPEN bằng 3 fails
* advance time 1000
* downstream chuyển sang success
* chạy 2 probes success => CLOSED

### Test 3 — half-open failure re-open

* sau khi HALF_OPEN, để probe fail
* expect state OPEN, rejected tăng

### Test 4 — rolling window opens only when minSamples met

* minSamples=10, threshold=0.5
* tạo 9 samples fail => chưa OPEN
* sample thứ 10 fail => OPEN

### Test 5 — timeout triggers failure and can open

* timeoutMs=10, downstream delay 50
* lặp đủ để open, verify timeoutTotal tăng

---

## README (ngắn nhưng “lead-ready”)

Bạn viết 10 dòng:

1. Vì sao breaker cần (tail latency + retry storm)
2. Detector bạn chọn làm core + trade-off
3. CLOSED/OPEN/HALF_OPEN semantics
4. HALF_OPEN probe policy bạn chọn (any-fail re-open, all-success close)
5. Timeout semantics (count as failure)
6. Abort semantics (caller cancel, no leak)
7. Metrics bạn expose và dùng để alert
8. Failure modes: flapping, mis-tuned threshold, false open
9. Khi nào breaker không giúp (CPU bound local, deterministic bugs)
10. Link đến runbook: “tune openDuration / thresholds” dựa trên p95 + error bursts

---

## Stretch (Senior+ hơn nữa)

Chọn 1:

1. **Two-threshold close policy**: HALF_OPEN requires successRate>=X thay vì all-success.
2. **Rolling window with buckets** (ring buffer) để O(1) updates, không shift array.
3. **Per-error classification**: chỉ đếm 5xx/Timeout, không đếm 4xx (tuỳ domain).
4. **Hedged requests guard**: nếu dùng hedging, breaker phải tính đúng failure.
5. **Integration with limiter (Kata 41)**: breaker OPEN ⇒ limiter deny ngay + expose reason.