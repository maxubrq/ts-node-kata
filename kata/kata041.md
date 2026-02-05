# Kata 41 — Max In-flight per Downstream

## Bảng bài làm (Definition-ready)

| Mục                           | Nội dung                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Goal**                      | (1) Xây **per-downstream concurrency limiter** (max in-flight). (2) Có **queue + FIFO fairness** (ít nhất theo downstream key). (3) Có **deadline/timeout + AbortSignal** để hủy chờ/đang chạy. (4) Có **metrics**: inflight, queued, acquired, rejected, timedOut, aborted, latency.                                                                                                                                               |
| **Context**                   | Bạn có service gọi nhiều downstream: `payments`, `inventory`, `shipping`. Khi spike, một downstream chậm sẽ kéo sập toàn hệ (connection pool cạn, event-loop lag, timeouts dây chuyền). Bạn cần “đập trần” số request đồng thời **cho từng downstream**.                                                                                                                                                                            |
| **Constraints**               | ❌ Không dùng thư viện limiter có sẵn (p-limit/bottleneck). ✅ Typescript strict. ✅ Must support: per-key limit, global limit (optional), FIFO queue per key, `timeoutMs`, `AbortSignal`, `maxQueue` (để load-shedding). ✅ Không leak memory: request bị abort phải rời queue.                                                                                                                                                        |
| **Deliverables**              | 1) `src/limiter.ts` triển khai `InFlightLimiter`. 2) `src/downstream.ts` mock downstream (delay/fail). 3) `test/limiter.test.ts` unit + concurrency tests. 4) `bench/limiter.bench.ts` (microbench) hoặc script load nhỏ. 5) `README.md` ngắn: trade-offs + failure modes.                                                                                                                                                          |
| **Checks (pass/fail)**        | (A) Không bao giờ vượt `maxInFlight` theo key (assert bằng counter). (B) FIFO: task vào trước được chạy trước (trong cùng key). (C) Abort khi đang **chờ**: bị remove khỏi queue, không chạy. (D) Abort khi đang **chạy**: signal propagate vào task, slot được release đúng. (E) Timeout khi chờ: trả error đúng loại, không chạy. (F) `maxQueue` vượt → reject ngay (load-shedding). (G) Metrics phản ánh đúng (queued/inflight). |
| **Stretch (Senior+ hơn nữa)** | 1) **Weighted slots**: mỗi task có “cost” (ví dụ 2 slots). 2) **Fairness cross-key**: global scheduler round-robin giữa keys (tránh key A chiếm CPU). 3) **Adaptive limit**: tăng/giảm limit theo latency p95 của downstream (đụng Kata 47). 4) **TryAcquire** + “fast-fail” mode cho endpoint quan trọng. 5) **AsyncLocalStorage**: trace `request_id` trong metrics/log.                                                          |

---

## API bạn sẽ build (đúng “production shape”)

Bạn implement interface kiểu này (đủ dùng ở service thật):

```ts
export type DownstreamKey = string;

export type AcquireOptions = {
  key: DownstreamKey;
  signal?: AbortSignal;
  timeoutMs?: number;   // timeout waiting in queue (not execution)
  maxQueue?: number;    // per-key queue cap (load-shed)
};

export type LimiterMetricsSnapshot = {
  inflightByKey: Record<string, number>;
  queuedByKey: Record<string, number>;
  acquiredTotal: number;
  rejectedQueueFullTotal: number;
  timedOutTotal: number;
  abortedTotal: number;
};

export class InFlightLimiter {
  constructor(cfg: {
    maxInFlightByKey: Record<string, number>;
    defaultMaxInFlight: number;
  });

  run<T>(
    opts: AcquireOptions,
    fn: (ctx: { signal: AbortSignal }) => Promise<T>
  ): Promise<T>;

  snapshot(): LimiterMetricsSnapshot;
}
```

---

## Starter Skeleton (điền TODO)

> Mục tiêu: đủ skeleton để bạn code thẳng + test thẳng.

```ts
// src/limiter.ts
export type AcquireOptions = {
  key: string;
  signal?: AbortSignal;
  timeoutMs?: number;
  maxQueue?: number;
};

type QueueItem<T> = {
  enqueuedAt: number;
  resolve: (v: T) => void;
  reject: (e: unknown) => void;
  fn: (ctx: { signal: AbortSignal }) => Promise<T>;
  // internal
  abortController: AbortController;
  cleanup: () => void;
};

export type LimiterMetricsSnapshot = {
  inflightByKey: Record<string, number>;
  queuedByKey: Record<string, number>;
  acquiredTotal: number;
  rejectedQueueFullTotal: number;
  timedOutTotal: number;
  abortedTotal: number;
};

export class QueueFullError extends Error {
  name = "QueueFullError";
}
export class AcquireTimeoutError extends Error {
  name = "AcquireTimeoutError";
}
export class AcquireAbortedError extends Error {
  name = "AcquireAbortedError";
}

export class InFlightLimiter {
  private inflight = new Map<string, number>();
  private queues = new Map<string, Array<QueueItem<unknown>>>();
  private maxByKey: Record<string, number>;
  private defaultMax: number;

  private acquiredTotal = 0;
  private rejectedQueueFullTotal = 0;
  private timedOutTotal = 0;
  private abortedTotal = 0;

  constructor(cfg: { maxInFlightByKey: Record<string, number>; defaultMaxInFlight: number }) {
    this.maxByKey = cfg.maxInFlightByKey;
    this.defaultMax = cfg.defaultMaxInFlight;
  }

  snapshot(): LimiterMetricsSnapshot {
    const inflightByKey: Record<string, number> = {};
    const queuedByKey: Record<string, number> = {};
    for (const [k, v] of this.inflight.entries()) inflightByKey[k] = v;
    for (const [k, q] of this.queues.entries()) queuedByKey[k] = q.length;

    return {
      inflightByKey,
      queuedByKey,
      acquiredTotal: this.acquiredTotal,
      rejectedQueueFullTotal: this.rejectedQueueFullTotal,
      timedOutTotal: this.timedOutTotal,
      abortedTotal: this.abortedTotal,
    };
  }

  async run<T>(opts: AcquireOptions, fn: (ctx: { signal: AbortSignal }) => Promise<T>): Promise<T> {
    const key = opts.key;
    const max = this.maxByKey[key] ?? this.defaultMax;

    // fast path: if slot available, execute immediately
    if (this.getInflight(key) < max) {
      return this.executeNow(key, opts.signal, fn);
    }

    // else enqueue with timeout/abort/maxQueue
    const q = this.getQueue(key);

    if (opts.maxQueue !== undefined && q.length >= opts.maxQueue) {
      this.rejectedQueueFullTotal++;
      throw new QueueFullError(`Queue full for key=${key}`);
    }

    return new Promise<T>((resolve, reject) => {
      const abortController = new AbortController();

      // merge external signal into internal signal
      const onExternalAbort = () => {
        abortController.abort();
      };
      if (opts.signal) {
        if (opts.signal.aborted) onExternalAbort();
        else opts.signal.addEventListener("abort", onExternalAbort, { once: true });
      }

      let timeoutId: NodeJS.Timeout | undefined;
      if (opts.timeoutMs !== undefined) {
        timeoutId = setTimeout(() => {
          // TODO: remove from queue + reject timeout
          this.timedOutTotal++;
          abortController.abort(); // optional
          reject(new AcquireTimeoutError(`Acquire timeout key=${key}`));
        }, opts.timeoutMs);
      }

      const item: QueueItem<T> = {
        enqueuedAt: Date.now(),
        resolve,
        reject,
        fn,
        abortController,
        cleanup: () => {
          if (timeoutId) clearTimeout(timeoutId);
          if (opts.signal) opts.signal.removeEventListener("abort", onExternalAbort);
        },
      };

      // IMPORTANT: when aborted while waiting, must remove from queue
      abortController.signal.addEventListener(
        "abort",
        () => {
          // TODO: if still in queue => remove + reject AcquireAbortedError
          // ensure item.cleanup() called once
        },
        { once: true }
      );

      q.push(item as QueueItem<unknown>);
      // TODO: attempt drain in case a slot freed between check and enqueue
      this.drain(key);
    });
  }

  private getInflight(key: string): number {
    return this.inflight.get(key) ?? 0;
  }

  private setInflight(key: string, n: number) {
    this.inflight.set(key, n);
  }

  private getQueue(key: string) {
    let q = this.queues.get(key);
    if (!q) {
      q = [];
      this.queues.set(key, q);
    }
    return q;
  }

  private async executeNow<T>(
    key: string,
    externalSignal: AbortSignal | undefined,
    fn: (ctx: { signal: AbortSignal }) => Promise<T>
  ): Promise<T> {
    this.setInflight(key, this.getInflight(key) + 1);
    this.acquiredTotal++;

    const abortController = new AbortController();
    const onExternalAbort = () => abortController.abort();
    if (externalSignal) {
      if (externalSignal.aborted) onExternalAbort();
      else externalSignal.addEventListener("abort", onExternalAbort, { once: true });
    }

    try {
      return await fn({ signal: abortController.signal });
    } finally {
      if (externalSignal) externalSignal.removeEventListener("abort", onExternalAbort);
      this.setInflight(key, this.getInflight(key) - 1);
      this.drain(key); // wake next queued
    }
  }

  private drain(key: string) {
    const max = this.maxByKey[key] ?? this.defaultMax;
    const q = this.getQueue(key);

    while (this.getInflight(key) < max && q.length > 0) {
      const item = q.shift()!;
      // TODO: skip items already aborted/timed out (must not run)
      // TODO: execute item.fn with item.abortController.signal
      // IMPORTANT: resolve/reject + cleanup + release slot in finally
      this.spawn(key, item);
    }
  }

  private spawn(key: string, item: QueueItem<unknown>) {
    // fire-and-forget wrapper, but promise resolution goes to item.resolve/reject
    this.setInflight(key, this.getInflight(key) + 1);
    this.acquiredTotal++;

    const signal = item.abortController.signal;

    // If already aborted before running, don't execute
    if (signal.aborted) {
      this.abortedTotal++;
      item.cleanup();
      item.reject(new AcquireAbortedError(`Aborted before execution key=${key}`));
      this.setInflight(key, this.getInflight(key) - 1);
      this.drain(key);
      return;
    }

    (async () => {
      try {
        const v = await item.fn({ signal });
        item.resolve(v);
      } catch (e) {
        item.reject(e);
      } finally {
        item.cleanup();
        this.setInflight(key, this.getInflight(key) - 1);
        this.drain(key);
      }
    })().catch(() => {
      // swallow - item.reject already handled
    });
  }
}
```

---

## Test plan (đủ “Senior+”, không chơi cho có)

Dùng `vitest`. Bạn viết test theo nhóm sau:

### 1) Never exceed max inflight (hard invariant)

* Setup limiter: `payments: 2`
* Spawn 50 tasks `payments` (mỗi task delay 30ms)
* Trong task: tăng `currentInflight++`, assert `<=2`, rồi giảm khi xong
* Kỳ vọng: không bao giờ vượt 2.

### 2) FIFO fairness per key

* `payments: 1`
* Enqueue tasks A,B,C (delay trong body khác nhau nhưng start order phải theo enqueue)
* Ghi lại `started[]`
* Kỳ vọng: started = [A,B,C]

### 3) Abort while waiting removes from queue

* `payments: 1`
* Task1 chạy lâu giữ slot
* Enqueue Task2 với `AbortController`, abort trước khi Task1 xong
* Kỳ vọng: Task2 reject `AcquireAbortedError` và **không bao giờ start**

### 4) Timeout while waiting (deadline for acquire)

* `payments: 1`
* Task1 chạy lâu
* Task2 timeoutMs=20, trong khi Task1 delay=100
* Kỳ vọng: Task2 reject `AcquireTimeoutError`, không start, queued giảm đúng

### 5) Abort while executing propagates

* Task body tôn trọng signal: nếu aborted thì throw `AbortError`
* Start Task, abort signal khi đang chạy
* Kỳ vọng: reject đúng loại + slot release + queue drain tiếp

### 6) maxQueue load-shedding

* `payments: 1`, `maxQueue=2`
* Đang chạy 1 + queue 2 => task thứ 4 phải throw `QueueFullError` ngay

### 7) Metrics sanity

* Khi enqueue 3 tasks: snapshot queued=3
* Khi chạy: inflight=limit
* Sau khi xong: inflight=0, queued=0, counters tăng hợp lý

---

## Mock downstream (để bạn inject latency/fail/cancel)

```ts
// src/downstream.ts
export async function fakeDownstream(
  ms: number,
  opts?: { signal?: AbortSignal; fail?: boolean }
): Promise<string> {
  const signal = opts?.signal;
  await new Promise<void>((resolve, reject) => {
    const t = setTimeout(resolve, ms);
    const onAbort = () => {
      clearTimeout(t);
      reject(Object.assign(new Error("Aborted"), { name: "AbortError" }));
    };
    if (signal) {
      if (signal.aborted) return onAbort();
      signal.addEventListener("abort", onAbort, { once: true });
    }
  });
  if (opts?.fail) throw new Error("DownstreamFailure");
  return "ok";
}
```

---

## README yêu cầu (nhỏ nhưng sắc)

Trong `README.md`, bạn bắt buộc ghi đúng 6 ý này (1–2 dòng/ý):

1. **Why** max inflight cứu hệ gì (pool, tail latency, retry storm)
2. **Fairness model** bạn chọn (FIFO per key) + trade-off
3. **Cancel semantics**: waiting vs executing
4. **Timeout semantics**: timeout waiting, không timeout execution
5. **Load shedding**: maxQueue quyết định “fail fast” ở đâu
6. **Failure modes**: leak queue item nếu quên cleanup; double-resolve; race drain

---

## Stretch nâng độ khó (chọn 1 trong 5, làm là “đúng Senior+”)

* **Weighted slots**: `run({cost:2})` chiếm 2 inflight, max=10
* **Global cap** + per-key cap (hai tầng limiter)
* **Round-robin scheduler** giữa keys (tránh one key starving others)
* **Deadline propagation**: `timeoutMs` trở thành “deadline absolute” truyền sang downstream (Abort)
* **Tail latency protection**: nếu p95 > ngưỡng thì hạ limit (proto của Kata 47)