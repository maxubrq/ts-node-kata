# Kata 44 — Bulkhead Isolation: tách resource per route

## Goal

1. Implement **bulkhead isolation** theo route (hoặc “operation key”): mỗi route có **resource pool riêng** (concurrency + queue + timeout).
2. Chứng minh: route A bị spike/slow **không làm route B chết** (latency/timeout/reject được cô lập).
3. Có **fairness + load shedding** per route: `maxInFlight`, `maxQueue`, `queueTimeoutMs`.
4. Có **metrics** theo route và global: inflight, queued, rejects, timeouts, latency histogram (p50/p95/p99 optional).
5. Có integration points: middleware HTTP (Express/Fastify) hoặc generic wrapper dùng cho bất kỳ handler nào.

---

### Context

Trong production, “sập” thường không đến từ bug, mà từ **resource contention**:

* 1 route nặng (search/export/report) làm cạn DB pool / CPU / event loop → route quan trọng (checkout/login) cũng chết.
  Bulkhead = vách ngăn tàu: khoang bị ngập thì khoang khác vẫn nổi.

---

### Constraints

* ❌ Không dùng thư viện bulkhead/limiter có sẵn.
* ✅ Không được “1 limiter global cho tất cả” rồi gọi là done. Phải **per-route limiter** (và optional global).
* ✅ Mỗi route có config riêng: `maxInFlight`, `maxQueue`, `queueTimeoutMs`.
* ✅ Hỗ trợ `AbortSignal` + deadline.
* ✅ Test phải chứng minh isolation bằng số liệu (không chỉ “chạy được”).
* ✅ Không leak memory: request bị abort/timeout phải rời queue.

---

### Deliverables

1. `src/bulkhead.ts` — BulkheadManager + RouteBulkhead
2. `src/limiter.ts` — bạn có thể reuse từ Kata 41 nhưng **phải extend** thêm queueTimeout + maxQueue theo “compartment”
3. `src/server.ts` — demo HTTP hoặc simulation runner
4. `test/bulkhead.test.ts` — isolation tests + concurrency invariants
5. `README.md` — mapping resource → bulkhead + trade-offs

---

## Bạn sẽ build cái gì (API)

```ts
export type BulkheadKey = string;

export type BulkheadConfig = {
  maxInFlight: number;
  maxQueue: number;
  queueTimeoutMs: number;
};

export type BulkheadManagerConfig = {
  byKey: Record<BulkheadKey, BulkheadConfig>;
  default: BulkheadConfig;
  // optional global cap (stretch)
  globalMaxInFlight?: number;
};

export type BulkheadMetricsSnapshot = {
  byKey: Record<string, {
    inflight: number;
    queued: number;
    allowedTotal: number;
    rejectedQueueFullTotal: number;
    timedOutInQueueTotal: number;
    abortedTotal: number;
    completedTotal: number;
    failedTotal: number;
  }>;
};

export class BulkheadManager {
  constructor(cfg: BulkheadManagerConfig);

  run<T>(
    key: BulkheadKey,
    fn: (ctx: { signal: AbortSignal }) => Promise<T>,
    opts?: { signal?: AbortSignal; timeoutMs?: number } // execution timeout optional (stretch)
  ): Promise<T>;

  snapshot(): BulkheadMetricsSnapshot;
}
```

> “tách resource per route” nghĩa là **key = route** (vd: `GET /search`, `POST /checkout`) hoặc key = “operation” (vd: `search`, `checkout`).

---

## Semantics bắt buộc (đúng production)

### Admission control per route

Khi request vào route key:

* Nếu `inflight < maxInFlight` ⇒ chạy ngay
* Else nếu `queued < maxQueue` ⇒ vào queue
* Else ⇒ reject ngay (fail fast) `QueueFullError`

### Queue timeout

Nếu request nằm trong queue quá `queueTimeoutMs` ⇒ reject `QueueTimeoutError` (không được chạy sau đó nữa).

### Cancel

Nếu client abort khi đang chờ ⇒ remove khỏi queue và reject `AbortedError`.
Nếu abort khi đang chạy ⇒ propagate AbortSignal vào handler và release slot đúng.

### Isolation requirement

* Route A full queue / inflight **không được làm** route B bị reject/timeout do A (trừ khi bạn có global cap, nhưng đó là stretch và phải giải thích).

---

## Starter skeleton (nhìn là code)

### `src/errors.ts`

```ts
export class QueueFullError extends Error { name = "QueueFullError"; }
export class QueueTimeoutError extends Error { name = "QueueTimeoutError"; }
export class RequestAbortedError extends Error { name = "RequestAbortedError"; }
export class ExecutionTimeoutError extends Error { name = "ExecutionTimeoutError"; } // stretch
```

### `src/bulkhead.ts`

```ts
import { QueueFullError, QueueTimeoutError, RequestAbortedError, ExecutionTimeoutError } from "./errors";

export type BulkheadConfig = { maxInFlight: number; maxQueue: number; queueTimeoutMs: number; };

type Item<T> = {
  enqAt: number;
  resolve: (v: T) => void;
  reject: (e: unknown) => void;
  fn: (ctx: { signal: AbortSignal }) => Promise<T>;
  ac: AbortController;
  timeoutId?: NodeJS.Timeout;
  cleanup: () => void;
};

class RouteBulkhead {
  inflight = 0;
  queue: Array<Item<unknown>> = [];

  allowedTotal = 0;
  rejectedQueueFullTotal = 0;
  timedOutInQueueTotal = 0;
  abortedTotal = 0;
  completedTotal = 0;
  failedTotal = 0;

  constructor(
    private cfg: BulkheadConfig,
    private now: () => number
  ) {}

  snapshot() {
    return {
      inflight: this.inflight,
      queued: this.queue.length,
      allowedTotal: this.allowedTotal,
      rejectedQueueFullTotal: this.rejectedQueueFullTotal,
      timedOutInQueueTotal: this.timedOutInQueueTotal,
      abortedTotal: this.abortedTotal,
      completedTotal: this.completedTotal,
      failedTotal: this.failedTotal,
    };
  }

  run<T>(
    fn: (ctx: { signal: AbortSignal }) => Promise<T>,
    opts?: { signal?: AbortSignal; timeoutMs?: number } // exec timeout stretch
  ): Promise<T> {
    // fast path
    if (this.inflight < this.cfg.maxInFlight) {
      return this.execNow(fn, opts);
    }

    // queue admission
    if (this.queue.length >= this.cfg.maxQueue) {
      this.rejectedQueueFullTotal++;
      throw new QueueFullError("bulkhead queue full");
    }

    return new Promise<T>((resolve, reject) => {
      const ac = new AbortController();
      const onAbort = () => ac.abort();
      if (opts?.signal) {
        if (opts.signal.aborted) onAbort();
        else opts.signal.addEventListener("abort", onAbort, { once: true });
      }

      const item: Item<T> = {
        enqAt: this.now(),
        resolve,
        reject,
        fn,
        ac,
        cleanup: () => {
          if (item.timeoutId) clearTimeout(item.timeoutId);
          if (opts?.signal) opts.signal.removeEventListener("abort", onAbort);
        }
      };

      // queue-timeout
      item.timeoutId = setTimeout(() => {
        // TODO: remove from queue if still waiting, then reject
        this.timedOutInQueueTotal++;
        item.cleanup();
        reject(new QueueTimeoutError("queue timeout"));
      }, this.cfg.queueTimeoutMs);

      // abort while waiting
      ac.signal.addEventListener("abort", () => {
        // TODO: remove from queue if still waiting, then reject
        this.abortedTotal++;
        item.cleanup();
        reject(new RequestAbortedError("aborted while waiting"));
      }, { once: true });

      this.queue.push(item as Item<unknown>);
      this.drain(opts);
    });
  }

  private async execNow<T>(
    fn: (ctx: { signal: AbortSignal }) => Promise<T>,
    opts?: { signal?: AbortSignal; timeoutMs?: number }
  ): Promise<T> {
    this.inflight++;
    this.allowedTotal++;

    const ac = new AbortController();
    const onAbort = () => ac.abort();
    if (opts?.signal) {
      if (opts.signal.aborted) onAbort();
      else opts.signal.addEventListener("abort", onAbort, { once: true });
    }

    let execTimeoutId: NodeJS.Timeout | undefined;
    const timeoutMs = opts?.timeoutMs; // stretch: execution timeout

    try {
      if (timeoutMs !== undefined) {
        return await Promise.race([
          fn({ signal: ac.signal }),
          new Promise<never>((_, rej) => {
            execTimeoutId = setTimeout(() => rej(new ExecutionTimeoutError("execution timeout")), timeoutMs);
          })
        ]);
      }
      return await fn({ signal: ac.signal });
    } catch (e) {
      this.failedTotal++;
      throw e;
    } finally {
      if (execTimeoutId) clearTimeout(execTimeoutId);
      if (opts?.signal) opts.signal.removeEventListener("abort", onAbort);
      this.inflight--;
      this.completedTotal++;
      this.drain(opts);
    }
  }

  private drain(opts?: { signal?: AbortSignal; timeoutMs?: number }) {
    while (this.inflight < this.cfg.maxInFlight && this.queue.length > 0) {
      const item = this.queue.shift()!;

      // skip if already aborted before execution
      if (item.ac.signal.aborted) {
        this.abortedTotal++;
        item.cleanup();
        item.reject(new RequestAbortedError("aborted before start"));
        continue;
      }

      // IMPORTANT: clear queue timeout now that it's starting
      item.cleanup();

      // run in background, resolving original promise
      this.execFromQueue(item, opts);
    }
  }

  private execFromQueue(item: Item<unknown>, opts?: { timeoutMs?: number }) {
    this.inflight++;
    this.allowedTotal++;

    (async () => {
      try {
        const v = await item.fn({ signal: item.ac.signal });
        item.resolve(v);
      } catch (e) {
        this.failedTotal++;
        item.reject(e);
      } finally {
        this.inflight--;
        this.completedTotal++;
        this.drain(opts);
      }
    })().catch(() => {});
  }
}

export class BulkheadManager {
  private byKey = new Map<string, RouteBulkhead>();

  constructor(
    private cfg: {
      byKey: Record<string, BulkheadConfig>;
      default: BulkheadConfig;
    },
    private deps: { now?: () => number } = {}
  ) {}

  private now() { return this.deps.now?.() ?? Date.now(); }

  private get(key: string) {
    let b = this.byKey.get(key);
    if (!b) {
      const c = this.cfg.byKey[key] ?? this.cfg.default;
      b = new RouteBulkhead(c, () => this.now());
      this.byKey.set(key, b);
    }
    return b;
  }

  run<T>(key: string, fn: (ctx: { signal: AbortSignal }) => Promise<T>, opts?: { signal?: AbortSignal; timeoutMs?: number }) {
    return this.get(key).run(fn, opts);
  }

  snapshot() {
    const byKey: Record<string, any> = {};
    for (const [k, b] of this.byKey.entries()) byKey[k] = b.snapshot();
    return { byKey };
  }
}
```

---

## Test plan bắt buộc (chứng minh isolation bằng số liệu)

### Test 1 — Route A overload không ảnh hưởng route B

* Config:

  * `A(search)`: maxInFlight=2, maxQueue=2, queueTimeout=50ms
  * `B(checkout)`: maxInFlight=5, maxQueue=10, queueTimeout=200ms
* Tạo 50 requests vào A (handler delay 200ms)
* Đồng thời tạo 20 requests vào B (handler delay 10ms)
* Expect:

  * B: **success gần như 20/20**, latency không tăng bất thường
  * A: reject/timeout xảy ra (đúng), nhưng **B không bị kéo theo**
  * Snapshot: A rejected/timeout > 0, B rejected/timeout = 0

### Test 2 — Hard cap concurrency per route

* Với route B maxInFlight=3, bắn 30 requests
* Trong handler: `running++`, assert `running <= 3`

### Test 3 — Queue timeout per route

* route A: maxInFlight=1, queue=1, queueTimeout=20ms
* 1 job chạy lâu giữ slot, job thứ 2 vào queue, job thứ 3 reject queueFull
* Advance time: job thứ 2 phải timeout (QueueTimeoutError), không được chạy sau đó

### Test 4 — Abort while waiting removes item

* route A: maxInFlight=1, queue=10
* Enqueue item đang chờ, abort signal trước khi slot rảnh
* Expect aborted counter tăng, queued giảm, handler **không được chạy**

### Test 5 — Fairness FIFO trong cùng route

* route A: maxInFlight=1
* Enqueue A1, A2, A3
* Record start order: phải đúng FIFO

> Dùng `vitest` fake timers + injected `now()` để deterministic.

---

## Demo runner (optional nhưng rất “Senior+ signal”)

* Một script `src/server.ts` (Express/Fastify) có 2 route:

  * `/search` chậm + error burst
  * `/checkout` nhanh
* Bạn log metrics mỗi 1s: inflight/queued/reject/timeout per route
* Mục tiêu: cho thấy `/checkout` vẫn “xanh” khi `/search` đỏ.

---

## Checks (Definition of Done)

* A bị overload → A reject/timeout tăng, nhưng B vẫn success + latency ổn.
* Không route nào vượt `maxInFlight`.
* Queue timeout và abort không leak queue item.
* Metrics snapshot đúng và dùng được để alert (reject/timeout spikes).
* README giải thích trade-offs.

---

## README bắt buộc (10 dòng, không lan man)

1. Bulkhead là gì (resource isolation)
2. Resource bạn đang tách là gì (concurrency slots + queue)
3. Mapping route→bulkhead key
4. Queue timeout vs execution timeout (khác nhau)
5. Load shedding: khi nào reject ngay
6. FIFO fairness trong route và limitation của nó
7. Vì sao isolation giúp tránh “noisy neighbor”
8. Failure modes (misconfig nhỏ làm drop request quan trọng)
9. Tuning: chọn maxInFlight/maxQueue dựa vào latency & downstream capacity
10. Next step: global cap + priority (liên kết Kata 45/46)

---

## Stretch (chọn 1 để lên hẳn Lead-grade)

1. **Global cap + per-route cap** (2 tầng): chống tổng thể chết vì quá nhiều inflight toàn hệ.
2. **Priority lanes** trong 1 route (vip vs normal).
3. **Dynamic config reload** (hot reload) + safe defaults.
4. **Couple với Circuit Breaker**: breaker OPEN thì route bulkhead tự “deny fast” với reason code.
5. **Per-route budget theo SLO**: nếu p95 route vượt ngưỡng, tự giảm maxInFlight.
