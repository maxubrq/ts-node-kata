# Kata 47 — Adaptive Concurrency: tăng/giảm concurrency theo latency

## Goal

1. Build **adaptive concurrency limiter** (ACC) cho 1 downstream/route: tự động tăng/giảm `concurrencyLimit` dựa trên **latency** (và optional error/timeout).
2. Chứng minh: khi downstream “đẹp” → tăng throughput; khi downstream “xấu” → giảm concurrency để tránh meltdown (queue phình, tail latency nổ).
3. Có **controller** ổn định (không flapping): smoothing + step rules + min/max bounds.
4. Có **metrics + events**: limit history, inflight, queue, latency EWMA/p95, drops/rejects, success rate.
5. Có test deterministic + simulation: latency model thay đổi theo load.

---

## Context

Fixed concurrency (vd 50) thường sai:

* lúc downstream khỏe: 50 là quá thấp → lãng phí
* lúc downstream yếu: 50 là quá cao → timeout storm
  Adaptive concurrency là “tự cân” theo tín hiệu thực tế (latency), giống kiểu Netflix/Envoy-ish thinking (nhưng bạn tự làm).

---

## Constraints

* ❌ Không dùng library ACC có sẵn.
* ✅ Phải có: `minLimit`, `maxLimit`, `initialLimit`.
* ✅ Phải có **smoothing** latency (EWMA hoặc sliding window).
* ✅ Phải có **tăng chậm / giảm nhanh** (asymmetry) để ổn định.
* ✅ Phải có **queue + maxInFlight** hard cap theo `currentLimit`.
* ✅ Tín hiệu chính: **latency** (p95 hoặc EWMA). Optional: errorRate/timeoutRate.
* ✅ Test phải chứng minh: khi latency tăng, limit giảm; khi latency giảm, limit tăng (với độ trễ hợp lý).

---

## What you will build (API)

```ts
export type AdaptiveConfig = {
  name: string;

  minLimit: number;
  maxLimit: number;
  initialLimit: number;

  // controller tick
  tickIntervalMs: number;

  // latency target
  targetP95Ms: number;       // or targetEwmaMs
  tolerance: number;         // 0..1, e.g. 0.15 (15%)

  // adjustment
  increaseStep: number;      // e.g. +1
  decreaseFactor: number;    // e.g. 0.7 (reduce to 70%)
  maxDecreaseStep?: number;  // optional clamp

  // measurement window
  windowMs: number;          // e.g. 10_000
  minSamples: number;        // e.g. 20

  // queue admission
  maxQueue: number;
  queueTimeoutMs: number;

  // optional: penalize errors/timeouts
  maxErrorRate?: number;     // e.g. 0.05
};

export type Snapshot = {
  limit: number;
  inflight: number;
  queued: number;

  samples: number;
  p95Ms?: number;
  ewmaMs?: number;
  errorRate?: number;

  allowedTotal: number;
  rejectedQueueFullTotal: number;
  timedOutInQueueTotal: number;

  adjustedUpTotal: number;
  adjustedDownTotal: number;
};

export class AdaptiveLimiter {
  constructor(cfg: AdaptiveConfig, deps: {
    now: () => number;
    rand?: () => number;
    setInterval: (fn: () => void, ms: number) => any;
    clearInterval: (h: any) => void;
  });

  start(): void;
  stop(): void;

  run<T>(
    fn: (ctx: { signal: AbortSignal }) => Promise<T>,
    opts?: { signal?: AbortSignal; timeoutMs?: number } // execution timeout optional
  ): Promise<T>;

  snapshot(): Snapshot;
}
```

---

## Controller design (Senior+ nhưng làm được)

### Measurements

Bạn thu thập per request:

* `latencyMs`
* `ok/fail/timeout` (optional)
  Cửa sổ `windowMs` lưu samples (ring buffer) để tính:
* `p95 latency` (core)
* `errorRate` (stretch)
* hoặc `EWMA latency` (thay p95 nếu bạn muốn đơn giản)

**Core khuyến nghị:** p95 trong window (minSamples >= N), vì tail latency mới là thứ giết hệ.

### Control law (tăng chậm / giảm nhanh)

Mỗi tick:

* Nếu `p95 > target * (1 + tolerance)` ⇒ **decrease fast**

  * `limit = max(minLimit, floor(limit * decreaseFactor))` (vd 0.7)
* Nếu `p95 < target * (1 - tolerance)` và queue/latency ổn ⇒ **increase slow**

  * `limit = min(maxLimit, limit + increaseStep)`
* Nếu nằm trong band ⇒ giữ nguyên

Thêm ổn định:

* Chỉ adjust nếu `samples >= minSamples`
* Có **cooldown** (optional): không adjust liên tục mỗi tick nếu vừa giảm.

---

## Admission + execution semantics (phải đúng)

### Hard cap inflight = currentLimit

* Nếu inflight < limit ⇒ chạy ngay
* Else nếu queue < maxQueue ⇒ enqueue
* Else ⇒ reject `QueueFullError`

### Queue timeout

* Chờ quá `queueTimeoutMs` ⇒ `QueueTimeoutError`, không được chạy sau đó.

### Cancel

* Abort trong queue ⇒ remove, reject
* Abort khi chạy ⇒ propagate signal vào fn, release slot, record latency (tuỳ policy)

---

## Starter skeleton (nhìn vào là code)

```ts
// src/adaptiveLimiter.ts
import { QueueFullError, QueueTimeoutError, RequestAbortedError } from "./errors";

type Sample = { t: number; ms: number; ok: boolean };

export class AdaptiveLimiter {
  private limit: number;
  private inflight = 0;
  private queue: Array<{
    enqAt: number;
    ac: AbortController;
    resolve: (v: any) => void;
    reject: (e: any) => void;
    fn: (ctx: { signal: AbortSignal }) => Promise<any>;
    timeoutId?: NodeJS.Timeout;
    cleanup: () => void;
  }> = [];

  private samples: Sample[] = [];
  private allowedTotal = 0;
  private rejectedQueueFullTotal = 0;
  private timedOutInQueueTotal = 0;
  private adjustedUpTotal = 0;
  private adjustedDownTotal = 0;

  private timer: any;

  constructor(
    private cfg: any,
    private deps: {
      now: () => number;
      setInterval: (fn: () => void, ms: number) => any;
      clearInterval: (h: any) => void;
    }
  ) {
    this.limit = cfg.initialLimit;
  }

  start() {
    if (this.timer) return;
    this.timer = this.deps.setInterval(() => this.tick(), this.cfg.tickIntervalMs);
  }

  stop() {
    if (!this.timer) return;
    this.deps.clearInterval(this.timer);
    this.timer = undefined;
  }

  snapshot() {
    const { p95, samples } = this.computeP95();
    return {
      limit: this.limit,
      inflight: this.inflight,
      queued: this.queue.length,
      samples,
      p95Ms: p95,
      allowedTotal: this.allowedTotal,
      rejectedQueueFullTotal: this.rejectedQueueFullTotal,
      timedOutInQueueTotal: this.timedOutInQueueTotal,
      adjustedUpTotal: this.adjustedUpTotal,
      adjustedDownTotal: this.adjustedDownTotal,
    };
  }

  async run<T>(fn: (ctx: { signal: AbortSignal }) => Promise<T>, opts?: { signal?: AbortSignal }): Promise<T> {
    if (this.inflight < this.limit) return this.execNow(fn, opts);

    if (this.queue.length >= this.cfg.maxQueue) {
      this.rejectedQueueFullTotal++;
      throw new QueueFullError("queue full");
    }

    return new Promise<T>((resolve, reject) => {
      const ac = new AbortController();
      const onAbort = () => ac.abort();
      if (opts?.signal) {
        if (opts.signal.aborted) onAbort();
        else opts.signal.addEventListener("abort", onAbort, { once: true });
      }

      const item = {
        enqAt: this.deps.now(),
        ac,
        resolve,
        reject,
        fn,
        cleanup: () => {
          if (item.timeoutId) clearTimeout(item.timeoutId);
          if (opts?.signal) opts.signal.removeEventListener("abort", onAbort);
        },
        timeoutId: undefined as any,
      };

      item.timeoutId = setTimeout(() => {
        // TODO: remove from queue if still waiting
        this.timedOutInQueueTotal++;
        item.cleanup();
        reject(new QueueTimeoutError("queue timeout"));
      }, this.cfg.queueTimeoutMs);

      ac.signal.addEventListener("abort", () => {
        // TODO: remove from queue if still waiting
        item.cleanup();
        reject(new RequestAbortedError("aborted"));
      }, { once: true });

      this.queue.push(item);
      this.drain();
    });
  }

  private async execNow<T>(fn: (ctx: { signal: AbortSignal }) => Promise<T>, opts?: { signal?: AbortSignal }): Promise<T> {
    this.inflight++;
    this.allowedTotal++;

    const ac = new AbortController();
    const onAbort = () => ac.abort();
    if (opts?.signal) {
      if (opts.signal.aborted) onAbort();
      else opts.signal.addEventListener("abort", onAbort, { once: true });
    }

    const start = this.deps.now();
    try {
      const v = await fn({ signal: ac.signal });
      this.recordSample(this.deps.now() - start, true);
      return v;
    } catch (e) {
      this.recordSample(this.deps.now() - start, false);
      throw e;
    } finally {
      if (opts?.signal) opts.signal.removeEventListener("abort", onAbort);
      this.inflight--;
      this.drain();
    }
  }

  private drain() {
    while (this.inflight < this.limit && this.queue.length > 0) {
      const item = this.queue.shift()!;
      if (item.ac.signal.aborted) {
        item.cleanup();
        item.reject(new RequestAbortedError("aborted before start"));
        continue;
      }
      item.cleanup(); // clear queue timeout & external listener
      this.execFromQueue(item);
    }
  }

  private execFromQueue(item: any) {
    this.inflight++;
    this.allowedTotal++;

    const start = this.deps.now();
    (async () => {
      try {
        const v = await item.fn({ signal: item.ac.signal });
        this.recordSample(this.deps.now() - start, true);
        item.resolve(v);
      } catch (e) {
        this.recordSample(this.deps.now() - start, false);
        item.reject(e);
      } finally {
        this.inflight--;
        this.drain();
      }
    })().catch(() => {});
  }

  private tick() {
    // controller tick: compute p95; adjust limit with hysteresis band
    const { p95, samples } = this.computeP95();
    if (!p95 || samples < this.cfg.minSamples) return;

    const hi = this.cfg.targetP95Ms * (1 + this.cfg.tolerance);
    const lo = this.cfg.targetP95Ms * (1 - this.cfg.tolerance);

    if (p95 > hi) {
      const next = Math.max(this.cfg.minLimit, Math.floor(this.limit * this.cfg.decreaseFactor));
      if (next < this.limit) {
        this.limit = next;
        this.adjustedDownTotal++;
      }
      return;
    }

    if (p95 < lo) {
      const next = Math.min(this.cfg.maxLimit, this.limit + this.cfg.increaseStep);
      if (next > this.limit) {
        this.limit = next;
        this.adjustedUpTotal++;
      }
      this.drain(); // new capacity may exist
    }
  }

  private recordSample(ms: number, ok: boolean) {
    const t = this.deps.now();
    this.samples.push({ t, ms, ok });
    const cutoff = t - this.cfg.windowMs;
    while (this.samples.length > 0 && this.samples[0].t < cutoff) this.samples.shift();
  }

  private computeP95(): { p95?: number; samples: number } {
    const arr = this.samples.map(s => s.ms);
    const n = arr.length;
    if (n === 0) return { samples: 0 };
    arr.sort((a, b) => a - b);
    const idx = Math.ceil(0.95 * n) - 1;
    return { p95: arr[Math.max(0, idx)], samples: n };
  }
}
```

> Đây là version “đủ làm”. Stretch sẽ tối ưu compute p95 (ring buffer/buckets) và add EWMA + error penalty.

---

## Test plan bắt buộc (deterministic)

### Test 1 — Increase when latency is low

* targetP95=100ms, tolerance=0.1
* fake downstream latency luôn 40ms
* start limit=2, max=10
* feed đủ samples (>=minSamples), tick nhiều lần
* expect limit tăng dần lên > 2

### Test 2 — Decrease fast when latency is high

* downstream latency 500ms
* start limit=10, min=1, decreaseFactor=0.7
* sau 1–2 ticks (đủ samples) expect limit giảm mạnh (10→7→4…)

### Test 3 — Band stability (no flapping)

* latency quanh target trong tolerance band (95–105ms khi target=100, tol=0.1)
* expect limit giữ nguyên

### Test 4 — Hard cap never exceeded

* limit hiện tại = 3
* bắn 50 requests song song
* trong handler assert running<=3

### Test 5 — Queue timeout works under low limit

* limit=1, maxQueue=1, queueTimeout=20ms
* job1 giữ 100ms, job2 vào queue, job3 queue full
* expect job2 timeout, job3 queue_full

### Test 6 — Recover after downstream improves

* phase 1: downstream chậm ⇒ limit giảm
* phase 2: downstream nhanh ⇒ limit tăng trở lại (không instant, nhưng có)

---

## Simulation bắt buộc (chứng minh “tăng/giảm theo load”)

Bạn mô phỏng downstream latency phụ thuộc inflight:

* `latency = base + k * inflight^2` (tail tăng phi tuyến)
* Khi limit tăng quá cao ⇒ p95 vượt target ⇒ controller giảm
* Kỳ vọng: hệ hội tụ quanh một limit ổn định

In mỗi 1s:

* limit, inflight, queued, p95, admitted/rejected

---

## Checks (Definition of Done)

* Limit không vượt bounds.
* Khi latency vượt target ⇒ limit giảm nhanh, queue/timeout giảm theo.
* Khi latency thấp ⇒ limit tăng dần, throughput tăng.
* Không flapping liên tục.
* Metrics snapshot đủ để vận hành.

---

## README bắt buộc (10 dòng, “lead-ready”)

1. Fixed concurrency fail ở đâu
2. Tín hiệu bạn dùng (p95) và vì sao
3. Control law (increase slow, decrease fast)
4. Smoothing/window + minSamples
5. Queue semantics (maxQueue, queueTimeout)
6. Failure modes: noisy measurements, oscillation, sample bias
7. Tuning playbook: targetP95/tolerance/steps
8. Khi nào ACC không phù hợp (hard rate limits, CPU-bound local)
9. Integration với Bulkhead/Circuit Breaker/Load Shedding
10. Metrics/alerts: limit stuck min, p95 vượt target, queue timeouts

---

## Stretch (Senior+ thật)

Chọn 1:

1. **Gradient-style controller**: điều chỉnh theo “distance” tới target thay vì step cố định.
2. **Error/timeout penalty**: nếu errorRate > maxErrorRate ⇒ giảm dù latency chưa cao.
3. **EWMA + p95 hybrid**: dùng EWMA để phản ứng nhanh, p95 để an toàn.
4. **Separate “queue wait” target**: giữ queueWaitP95 dưới ngưỡng thay vì latency.
5. **Per-route adaptive**: nhiều limiters + global cap (coupled control).
