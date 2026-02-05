# Kata 42 — Queue Worker: Concurrency=N + Retry + DLQ

## Bảng bài làm (Definition-ready)

| Mục                           | Nội dung                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Goal**                      | Xây một **queue worker** chạy nhiều job song song (concurrency=N) với: (1) retry có policy (max attempts, backoff+jitter), (2) **DLQ** khi vượt attempts hoặc lỗi “permanent”, (3) **visibility timeout/lease** để tránh job “kẹt” khi worker chết, (4) graceful shutdown: stop pulling + drain in-flight, (5) metrics + structured logs.                                                                                                                            |
| **Context**                   | Bạn có hàng đợi job (email, webhook, image processing). Production sẽ gặp: job fail tạm thời (network), job fail vĩnh viễn (bad payload), worker crash giữa chừng, poison message làm nghẽn hàng. Bạn phải chứng minh hệ **sống** và **quan sát được**.                                                                                                                                                                                                              |
| **Constraints**               | ❌ Không dùng BullMQ/Rabbit libs. ✅ Tự làm in-memory queue + DLQ nhưng mô phỏng đúng semantics. ✅ Concurrency phải là “hard cap”. ✅ Retry có jitter. ✅ Có phân loại lỗi: `RetryableError` vs `PermanentError`. ✅ Có test deterministically (fake timers).                                                                                                                                                                                                             |
| **Deliverables**              | 1) `src/queue.ts` (Queue + DLQ + lease). 2) `src/worker.ts` (Worker loop). 3) `src/policy.ts` (retry policy). 4) `test/worker.test.ts`. 5) `README.md` (semantics, trade-offs).                                                                                                                                                                                                                                                                                      |
| **Checks (pass/fail)**        | (A) Không bao giờ chạy quá N job đồng thời. (B) Retry đúng schedule (backoff, jitter bounded). (C) Job permanent → DLQ ngay. (D) Job vượt maxAttempts → DLQ. (E) Worker crash simulation: job đang lease nhưng không ack → sau visibilityTimeout job quay lại queue. (F) Poison message không chặn queue: sau X lần → DLQ, worker tiếp tục xử lý job khác. (G) Shutdown: không nhận job mới, đợi in-flight xong hoặc abort theo deadline. (H) Metrics counters đúng. |
| **Stretch (Senior+ hơn nữa)** | 1) **Priority queue** (high/low) + fairness. 2) **Idempotency key** cho handler (liên kết Kata 82). 3) **Batch pull** + prefetch window. 4) **Rate limit per job type**. 5) **Exactly-once illusion**: at-least-once + idempotent handler.                                                                                                                                                                                                                           |

---

## Semantics bạn cần mô phỏng (đúng production)

Bạn làm “queue” kiểu lease-based:

* `pull()` trả về job + `leaseId`, và set `visibleAt = now + visibilityTimeout`
* Worker phải `ack(leaseId)` khi done
* Nếu worker chết / không ack: sau `visibleAt`, job trở lại eligible để pull
* Retry: khi fail retryable → `rescheduleAt = now + backoff(attempt)` và job quay lại queue (không vào DLQ)
* DLQ: job fail permanent hoặc attempts vượt limit

---

## Data model (đủ cho test & production thinking)

```ts
export type JobId = string;

export type JobPayload = {
  type: "email" | "webhook" | "image";
  data: unknown;
};

export type Job = {
  id: JobId;
  payload: JobPayload;
  attempts: number;        // number of times started
  maxAttempts: number;
  createdAt: number;
  availableAt: number;     // when eligible to be pulled (retry scheduling)
  leasedUntil?: number;    // visibility timeout end
  leaseId?: string;        // identifies current lease
  lastError?: string;
};
```

---

## Starter skeleton (điền TODO)

### `src/errors.ts`

```ts
export class RetryableError extends Error {
  name = "RetryableError";
}
export class PermanentError extends Error {
  name = "PermanentError";
}
export class LeaseLostError extends Error {
  name = "LeaseLostError";
}
```

### `src/policy.ts` (retry policy: exponential backoff + jitter)

```ts
export type RetryPolicy = {
  baseMs: number;      // e.g. 100
  maxMs: number;       // e.g. 10_000
  jitter: number;      // 0..1, e.g. 0.2 means +/-20%
};

export function computeBackoffMs(attempt: number, p: RetryPolicy, rand = Math.random): number {
  // attempt starts at 1
  const exp = p.baseMs * Math.pow(2, attempt - 1);
  const capped = Math.min(exp, p.maxMs);
  const j = (rand() * 2 - 1) * p.jitter; // [-jitter, +jitter]
  const withJitter = Math.round(capped * (1 + j));
  return Math.max(0, withJitter);
}
```

### `src/queue.ts` (in-memory queue + DLQ + lease)

```ts
import { Job } from "./types";

export class InMemoryQueue {
  private main: Job[] = [];
  private dlq: Job[] = [];

  constructor(private now: () => number) {}

  enqueue(job: Job) {
    this.main.push(job);
  }

  listDLQ(): Job[] {
    return [...this.dlq];
  }

  // Pull ONE eligible job and lease it
  pull(opts: { visibilityTimeoutMs: number }): { job: Job; leaseId: string } | null {
    const t = this.now();

    // eligible: availableAt <= now and (not leased or lease expired)
    const idx = this.main.findIndex(j => {
      const leaseExpired = j.leasedUntil === undefined || j.leasedUntil <= t;
      return j.availableAt <= t && leaseExpired;
    });

    if (idx === -1) return null;

    const job = this.main[idx];
    const leaseId = `lease_${job.id}_${t}_${Math.random().toString(16).slice(2)}`;

    job.leaseId = leaseId;
    job.leasedUntil = t + opts.visibilityTimeoutMs;

    return { job: { ...job }, leaseId };
  }

  ack(leaseId: string): boolean {
    // TODO: remove job from main if leaseId matches current
    return false;
  }

  failToRetry(leaseId: string, err: Error, nextAvailableAt: number): boolean {
    // TODO: find job by leaseId, clear lease, set availableAt, store lastError
    return false;
  }

  moveToDLQ(leaseId: string, err: Error): boolean {
    // TODO: remove from main, push into dlq with lastError, clear lease
    return false;
  }

  // helpful for tests
  size(): number {
    return this.main.length;
  }
}
```

### `src/worker.ts` (worker loop with concurrency cap)

```ts
import { InMemoryQueue } from "./queue";
import { PermanentError, RetryableError } from "./errors";
import { computeBackoffMs, RetryPolicy } from "./policy";
import { Job } from "./types";

export type WorkerMetrics = {
  started: number;
  succeeded: number;
  failedRetryable: number;
  failedPermanent: number;
  movedToDLQ: number;
  leaseLost: number;
  inFlight: number;
  maxInFlightObserved: number;
};

export type WorkerOptions = {
  concurrency: number;
  visibilityTimeoutMs: number;
  pollIntervalMs: number;   // when no job, wait this much
  retryPolicy: RetryPolicy;
  rand?: () => number;
};

export class QueueWorker {
  private stopped = false;
  private inFlight = 0;
  public metrics: WorkerMetrics = {
    started: 0,
    succeeded: 0,
    failedRetryable: 0,
    failedPermanent: 0,
    movedToDLQ: 0,
    leaseLost: 0,
    inFlight: 0,
    maxInFlightObserved: 0,
  };

  constructor(
    private q: InMemoryQueue,
    private now: () => number,
    private sleep: (ms: number) => Promise<void>,
    private handle: (job: Job, ctx: { signal: AbortSignal }) => Promise<void>,
    private opts: WorkerOptions
  ) {}

  async start(signal?: AbortSignal) {
    // main loop
    while (!this.stopped) {
      if (signal?.aborted) break;

      // fill capacity
      while (this.inFlight < this.opts.concurrency) {
        const pulled = this.q.pull({ visibilityTimeoutMs: this.opts.visibilityTimeoutMs });
        if (!pulled) break;

        this.spawn(pulled.job, pulled.leaseId, signal);
      }

      if (this.inFlight === 0) {
        // no work; avoid busy-loop
        await this.sleep(this.opts.pollIntervalMs);
      } else {
        // small yield; still allow new pulls soon
        await this.sleep(0);
      }
    }

    // graceful drain
    while (this.inFlight > 0) await this.sleep(5);
  }

  stop() {
    this.stopped = true;
  }

  private spawn(job: Job, leaseId: string, parentSignal?: AbortSignal) {
    this.inFlight++;
    this.metrics.inFlight = this.inFlight;
    this.metrics.maxInFlightObserved = Math.max(this.metrics.maxInFlightObserved, this.inFlight);
    this.metrics.started++;

    const ac = new AbortController();
    const onAbort = () => ac.abort();
    if (parentSignal) {
      if (parentSignal.aborted) onAbort();
      else parentSignal.addEventListener("abort", onAbort, { once: true });
    }

    (async () => {
      try {
        // bump attempts in “queue state” by re-writing through fail/ack paths,
        // but in-memory we can do it logically here:
        job.attempts += 1;

        await this.handle(job, { signal: ac.signal });

        const ok = this.q.ack(leaseId);
        if (!ok) {
          this.metrics.leaseLost++;
          return;
        }
        this.metrics.succeeded++;
      } catch (e) {
        const err = e instanceof Error ? e : new Error(String(e));

        // classify
        const isPermanent = err instanceof PermanentError;
        const isRetryable = err instanceof RetryableError;

        if (isPermanent) {
          this.metrics.failedPermanent++;
          if (this.q.moveToDLQ(leaseId, err)) this.metrics.movedToDLQ++;
          return;
        }

        // default: retryable unless attempts exceeded
        this.metrics.failedRetryable++;

        const maxAttempts = job.maxAttempts;
        if (job.attempts >= maxAttempts) {
          if (this.q.moveToDLQ(leaseId, err)) this.metrics.movedToDLQ++;
          return;
        }

        const rand = this.opts.rand ?? Math.random;
        const backoff = computeBackoffMs(job.attempts, this.opts.retryPolicy, rand);
        const nextAt = this.now() + backoff;
        this.q.failToRetry(leaseId, err, nextAt);
      }
    })().finally(() => {
      if (parentSignal) parentSignal.removeEventListener("abort", onAbort);
      this.inFlight--;
      this.metrics.inFlight = this.inFlight;
    });
  }
}
```

---

## Test plan (7 bài test bắt buộc, deterministic)

> Dùng `vitest` + fake timers (`vi.useFakeTimers()`), inject `now()` và `sleep()` để điều khiển thời gian.

1. **Concurrency hard cap**

* concurrency=3, enqueue 20 job delay 50ms
* trong handler: tăng `running++`, assert `running<=3`
* expect `maxInFlightObserved === 3`

2. **Retry schedule (backoff + jitter bounded)**

* rand stub cố định (vd 0.0) để jitter deterministic
* job fail `RetryableError` 2 lần rồi success
* assert `availableAt` đúng theo computeBackoffMs

3. **PermanentError goes DLQ immediately**

* handler throw `PermanentError`
* expect DLQ size=1, job attempts=1, main queue decreased

4. **MaxAttempts exceeded goes DLQ**

* job maxAttempts=3, handler luôn throw RetryableError
* advance time đủ để retry chạy đủ 3 lần
* expect DLQ=1, main=0

5. **Poison message doesn’t block**

* enqueue poison job (luôn fail), và 5 job normal
* expect normal jobs succeed dù poison bị retry rồi DLQ

6. **Visibility timeout re-delivery**

* visibilityTimeout=20ms
* handler “hang” (không resolve) để simulate worker crash: bạn stop worker trước khi ack
* advance time > 20ms, start worker khác
* expect job được pull lại và xử lý

7. **Graceful shutdown drains**

* enqueue 10 job, handler delay 30ms
* start worker, đợi vài job chạy, call stop()
* expect không pull thêm job mới sau stop, nhưng in-flight hoàn thành

---

## README bắt buộc (ngắn nhưng đúng chỗ đau)

Bạn viết 8 dòng, mỗi dòng 1 ý:

1. at-least-once delivery (vì lease/visibility timeout)
2. retry vs DLQ: vì sao tách permanent/retryable
3. poison message impact & cách DLQ giải nghẽn
4. backoff+jitter chống retry storm
5. concurrency cap bảo vệ downstream
6. graceful shutdown semantics
7. metrics tối thiểu cần có để vận hành
8. failure modes bạn đã test (lease lost, crash, hang)

---

## Stretch (chọn 1 để “đậm Senior+”)

* **DLQ replayer**: tool chuyển DLQ → main với rate limit + reason filter
* **Batch pull + prefetch**: giảm overhead poll
* **Per-type concurrency**: email=5, webhook=20
* **Circuit breaker integration**: downstream fail nhiều → pause type đó 30s
* **Idempotency in handler**: xử lý re-delivery không double-effect (nối Kata 82/34)