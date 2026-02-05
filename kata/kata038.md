# Kata 38 — Connection Pool Tuning: simulate load + tune pool

## Goal

1. Viết một **connection pool** (semaphore + queue) có:

* `acquire()` / `release()`
* `max` (pool size)
* `acquireTimeoutMs`
* metrics: `inUse`, `queueLength`, `acquireWaitMs`, `timeouts`

2. Mô phỏng downstream DB:

* mỗi “query” tốn `latencyMs` (có jitter)
* optional: khi concurrency quá cao thì latency tăng (contention curve)

3. Chạy load simulation:

* N requests, mỗi request chạy K queries
* request concurrency = C
* đo: throughput (req/s), p50/p95/p99 latency, timeout rate

4. Tune pool:

* tìm `poolSize` tốt (trade-off throughput vs latency vs timeouts)
* viết **tuning note** ngắn: “chọn X vì …”

---

## Constraints

* ✅ Có **simulation runner** cho phép chạy nhiều config (poolSize khác nhau).
* ✅ Có output summary (console) + tiêu chí “best config”.
* ✅ Có test logic pool (không deadlock, release đúng, timeout đúng).
* ❌ Không “tune” bằng câu: “pool càng lớn càng tốt”.

---

## Context (bài toán demo)

Giả sử một API request cần:

* Query A: 30–60ms
* Query B: 30–60ms
  Tổng 2 queries tuần tự (đơn giản hóa), và có 100 requests đồng thời.

Bạn sẽ thấy:

* pool quá nhỏ ⇒ queue wait cao ⇒ p95 tăng
* pool quá lớn ⇒ contention ⇒ query latency tăng ⇒ throughput không tăng tương ứng, đôi khi còn tệ

---

## Deliverables

`kata-38/`

* `src/pool.ts` (pool + metrics)
* `src/fakeDb.ts` (simulate db query + contention)
* `src/load.ts` (load generator + percentile)
* `src/run.ts` (sweep configs, print table, pick best)
* `test/kata38.test.ts` (pool correctness)
* `notes/tuning.md` (kết luận tuning 1 trang)

---

## “Nhìn vào làm được” table

| Phần | Bạn làm gì     | Output                    | Checks         |
| ---- | -------------- | ------------------------- | -------------- |
| 1    | Implement pool | `Pool.acquire/release`    | no deadlock    |
| 2    | Fake DB query  | `query()` uses pool       | respects max   |
| 3    | Load sim       | concurrency C, N requests | metrics stable |
| 4    | Sweep          | poolSize=[2..64]          | pick best      |
| 5    | Write note     | decision + evidence       | not hand-wavy  |

---

## Starter Skeleton

### `src/pool.ts`

```ts
type Deferred<T> = {
  promise: Promise<T>;
  resolve: (v: T) => void;
  reject: (e: unknown) => void;
};

function deferred<T>(): Deferred<T> {
  let resolve!: (v: T) => void;
  let reject!: (e: unknown) => void;
  const promise = new Promise<T>((res, rej) => { resolve = res; reject = rej; });
  return { promise, resolve, reject };
}

export type PoolMetrics = {
  inUse: number;
  queueLength: number;
  acquireCount: number;
  acquireTimeouts: number;
  acquireWaitMsTotal: number;
};

export class ConnectionPool {
  private inUse = 0;
  private queue: Array<{ d: Deferred<void>; enqueuedAt: number }> = [];

  private metrics: PoolMetrics = {
    inUse: 0,
    queueLength: 0,
    acquireCount: 0,
    acquireTimeouts: 0,
    acquireWaitMsTotal: 0,
  };

  constructor(
    private readonly max: number,
    private readonly acquireTimeoutMs: number
  ) {}

  snapshot(): PoolMetrics {
    return { ...this.metrics, inUse: this.inUse, queueLength: this.queue.length };
  }

  async acquire(): Promise<void> {
    this.metrics.acquireCount++;

    if (this.inUse < this.max) {
      this.inUse++;
      this.metrics.inUse = this.inUse;
      return;
    }

    const d = deferred<void>();
    const enqueuedAt = Date.now();
    this.queue.push({ d, enqueuedAt });
    this.metrics.queueLength = this.queue.length;

    const t = setTimeout(() => {
      // timeout: remove from queue if still there
      const idx = this.queue.findIndex((x) => x.d === d);
      if (idx >= 0) {
        this.queue.splice(idx, 1);
        this.metrics.acquireTimeouts++;
        this.metrics.queueLength = this.queue.length;
        d.reject(new Error("ACQUIRE_TIMEOUT"));
      }
    }, this.acquireTimeoutMs);

    try {
      await d.promise;
      const wait = Date.now() - enqueuedAt;
      this.metrics.acquireWaitMsTotal += wait;
    } finally {
      clearTimeout(t);
    }
  }

  release(): void {
    if (this.inUse <= 0) throw new Error("RELEASE_WITHOUT_ACQUIRE");

    // If queue has waiters, hand off slot to next waiter (no decrement of inUse)
    const next = this.queue.shift();
    this.metrics.queueLength = this.queue.length;

    if (next) {
      next.d.resolve();
      return;
    }

    this.inUse--;
    this.metrics.inUse = this.inUse;
  }
}
```

---

### `src/fakeDb.ts`

```ts
import { ConnectionPool } from "./pool";

function sleep(ms: number) {
  return new Promise((r) => setTimeout(r, ms));
}

// Simulate contention: if concurrent queries > knee, latency increases
export type FakeDbOptions = {
  baseMinMs: number;       // e.g. 30
  baseMaxMs: number;       // e.g. 60
  contentionKnee: number;  // e.g. 8
  contentionFactor: number;// e.g. 0.15 (15% extra per over-knee)
};

export class FakeDb {
  private concurrent = 0;

  constructor(
    private readonly pool: ConnectionPool,
    private readonly opts: FakeDbOptions
  ) {}

  async query(tag?: string): Promise<void> {
    await this.pool.acquire();
    this.concurrent++;

    try {
      const base =
        this.opts.baseMinMs +
        Math.floor(Math.random() * (this.opts.baseMaxMs - this.opts.baseMinMs + 1));

      const over = Math.max(0, this.concurrent - this.opts.contentionKnee);
      const multiplier = 1 + over * this.opts.contentionFactor;

      const latency = Math.floor(base * multiplier);
      await sleep(latency);
    } finally {
      this.concurrent--;
      this.pool.release();
    }
  }
}
```

---

### `src/load.ts`

```ts
import { FakeDb } from "./fakeDb";

function nowMs() { return Date.now(); }

export type LoadResult = {
  totalRequests: number;
  ok: number;
  timeouts: number;
  latenciesMs: number[];
  durationMs: number;
};

export function percentile(xs: number[], p: number): number {
  if (xs.length === 0) return 0;
  const sorted = [...xs].sort((a, b) => a - b);
  const idx = Math.min(sorted.length - 1, Math.floor((p / 100) * sorted.length));
  return sorted[idx];
}

export async function runLoad(opts: {
  db: FakeDb;
  totalRequests: number;      // e.g. 2000
  concurrency: number;        // e.g. 100
  queriesPerRequest: number;  // e.g. 2
}): Promise<LoadResult> {
  const latencies: number[] = [];
  let ok = 0;
  let timeouts = 0;

  let i = 0;
  const start = nowMs();

  async function worker() {
    while (true) {
      const my = i++;
      if (my >= opts.totalRequests) break;

      const t0 = nowMs();
      try {
        for (let k = 0; k < opts.queriesPerRequest; k++) {
          await opts.db.query(`q${k}`);
        }
        ok++;
        latencies.push(nowMs() - t0);
      } catch (e: any) {
        const msg = e instanceof Error ? e.message : String(e);
        if (msg === "ACQUIRE_TIMEOUT") timeouts++;
        // treat as failed request; still record latency to see pain
        latencies.push(nowMs() - t0);
      }
    }
  }

  const workers = Array.from({ length: opts.concurrency }, () => worker());
  await Promise.all(workers);

  const end = nowMs();
  return {
    totalRequests: opts.totalRequests,
    ok,
    timeouts,
    latenciesMs: latencies,
    durationMs: end - start,
  };
}
```

---

### `src/run.ts` (sweep configs & pick best)

```ts
import { ConnectionPool } from "./pool";
import { FakeDb } from "./fakeDb";
import { percentile, runLoad } from "./load";

type SweepRow = {
  poolSize: number;
  okRate: number;
  rps: number;
  p50: number;
  p95: number;
  p99: number;
  timeouts: number;
  avgAcquireWait: number;
};

export async function sweep() {
  const totalRequests = 2000;
  const concurrency = 100;
  const queriesPerRequest = 2;

  const rows: SweepRow[] = [];

  for (const poolSize of [2, 4, 8, 12, 16, 24, 32, 48, 64]) {
    const pool = new ConnectionPool(poolSize, 50); // 50ms acquire timeout
    const db = new FakeDb(pool, {
      baseMinMs: 30,
      baseMaxMs: 60,
      contentionKnee: 8,
      contentionFactor: 0.15,
    });

    const res = await runLoad({ db, totalRequests, concurrency, queriesPerRequest });
    const p50 = percentile(res.latenciesMs, 50);
    const p95 = percentile(res.latenciesMs, 95);
    const p99 = percentile(res.latenciesMs, 99);

    const rps = res.ok / (res.durationMs / 1000);
    const okRate = res.ok / res.totalRequests;

    const m = pool.snapshot();
    const avgAcquireWait = m.acquireCount ? m.acquireWaitMsTotal / m.acquireCount : 0;

    rows.push({
      poolSize,
      okRate,
      rps,
      p50,
      p95,
      p99,
      timeouts: res.timeouts,
      avgAcquireWait,
    });
  }

  // Pick a sane “best”: okRate>=0.999, minimize p95, then maximize rps
  const candidates = rows.filter((r) => r.okRate >= 0.999);
  const best = (candidates.length ? candidates : rows)
    .sort((a, b) => (a.p95 - b.p95) || (b.rps - a.rps))[0];

  console.table(rows.map(r => ({
    pool: r.poolSize,
    okRate: r.okRate.toFixed(3),
    rps: r.rps.toFixed(1),
    p50: r.p50,
    p95: r.p95,
    p99: r.p99,
    timeouts: r.timeouts,
    avgWait: r.avgAcquireWait.toFixed(1),
  })));

  console.log("BEST:", best);
}

sweep().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

---

## Tests (vitest) — correctness trước khi tune

### `test/kata38.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { ConnectionPool } from "../src/pool";

function sleep(ms: number) { return new Promise(r => setTimeout(r, ms)); }

describe("kata38 - connection pool", () => {
  it("limits concurrency to max", async () => {
    const pool = new ConnectionPool(2, 1000);

    let inCritical = 0;
    let maxSeen = 0;

    async function task() {
      await pool.acquire();
      inCritical++;
      maxSeen = Math.max(maxSeen, inCritical);
      await sleep(10);
      inCritical--;
      pool.release();
    }

    await Promise.all(Array.from({ length: 10 }, () => task()));
    expect(maxSeen).toBe(2);
  });

  it("times out if acquire waits too long", async () => {
    const pool = new ConnectionPool(1, 10);

    await pool.acquire(); // take the only slot

    const p = pool.acquire(); // should timeout
    await expect(p).rejects.toThrow("ACQUIRE_TIMEOUT");

    pool.release();
  });

  it("release hands off to queued waiter", async () => {
    const pool = new ConnectionPool(1, 1000);

    await pool.acquire(); // occupy slot
    const waiter = pool.acquire(); // queued

    // release should unblock waiter
    pool.release();
    await expect(waiter).resolves.toBeUndefined();

    // now waiter holds slot, release it
    pool.release();
    expect(pool.snapshot().inUse).toBe(0);
  });
});
```

---

## What you must do (Task list)

1. Chạy tests: `pnpm test` (hoặc `vitest`)
2. Chạy sweep: `pnpm tsx src/run.ts`
3. Tạo `notes/tuning.md` trả lời 5 câu:

* Workload params (C, N, queries/request)
* Constraint (okRate target, timeout target)
* Best pool size bạn chọn + bảng evidence (p95/rps/avgWait)
* Vì sao không chọn lớn hơn/nhỏ hơn (trade-off)
* Guardrails production: pool max, timeout, alert

---

## Definition of Done (Checks)

Bạn pass kata này khi:

* Pool correctness tests pass.
* Sweep in ra bảng có sự khác biệt rõ (pool nhỏ → wait/timeout; pool lớn → contention).
* Bạn chọn được pool size dựa trên tiêu chí (ví dụ okRate >= 99.9%, p95 thấp).
* Có `tuning.md` giải thích decision như một tech lead.

---

## Stretch (Senior+)

1. **Little’s Law sanity check**

* Ước lượng: `pool ~ targetRPS * avgQueryTime` (chỉ để check hợp lý, không phải chân lý).

2. **Multiple pools**

* pool riêng cho read vs write, hoặc per downstream (bulkhead) — nối Kata 44.

3. **Adaptive pool**

* tự điều chỉnh max theo p95 acquire wait hoặc saturation.

4. **SLO alerts**

* alert khi `acquireTimeouts > 0` hoặc `avgAcquireWait > Xms`.