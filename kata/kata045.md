# Kata 45 — Priority Queue: SLA-aware processing

## Goal

1. Xây **priority queue scheduler** để xử lý job theo **SLA** (vip/standard/bulk) thay vì FIFO thuần.
2. Đảm bảo **không starve** (job priority thấp vẫn được chạy).
3. Có **admission control** (maxQueue per class) + **drop policy** khi quá tải.
4. Có **deadline/expiry** theo SLA: job quá hạn thì **expire** (drop) hoặc **downgrade** (tuỳ policy).
5. Có **metrics**: queue depth per class, wait time, deadline miss rate, throughput per class, starvation indicator.

---

### Context

Bạn có 3 loại workload:

* **P0 / VIP**: checkout, auth, payment webhook (SLA rất ngắn)
* **P1 / Standard**: search bình thường, user actions
* **P2 / Bulk**: export/report, backfill, reindex

Nếu dùng FIFO/global concurrency, bulk spike sẽ kéo VIP chết. Priority queue giúp hệ “ưu tiên thứ quan trọng” có kiểm soát, nhưng nếu làm ngu sẽ starve P2 mãi mãi.

---

### Constraints

* ❌ Không dùng thư viện priority queue có sẵn.
* ✅ Scheduler phải chạy với **concurrency=N** (giống worker) và lấy job theo priority.
* ✅ Phải có **anti-starvation**: ít nhất 1 trong 2 cơ chế sau (core chọn 1):

  * **Weighted Fair Queuing** (WFQ / weighted round robin theo tỷ lệ)
  * **Aging** (job chờ lâu sẽ được “nâng điểm”)
* ✅ Có **SLA deadline**: mỗi job có `deadlineAt` (hoặc `slaMs` + createdAt).
* ✅ Có test deterministic (fake clock).
* ✅ Có metrics snapshot.

---

## Bạn sẽ build cái gì (API)

```ts
export type PriorityClass = "P0" | "P1" | "P2";

export type Job<T> = {
  id: string;
  klass: PriorityClass;
  createdAt: number;
  deadlineAt: number;     // SLA deadline
  payload: T;
};

export type SchedulerConfig = {
  concurrency: number;

  // queue caps
  maxQueue: Record<PriorityClass, number>;

  // SLA policy
  expirePolicy: "drop" | "send_to_dlq";
  allowLatePolicy: "drop" | "process_with_tag";

  // anti-starvation
  mode: "wfq" | "aging";

  // wfq
  weights?: Record<PriorityClass, number>; // e.g. P0:8, P1:3, P2:1

  // aging
  agingHalfLifeMs?: number; // larger => slower aging

  // optional: admission rule under overload
  overloadDropOrder?: PriorityClass[]; // e.g. ["P2","P1"] (never drop P0)
};

export type MetricsSnapshot = {
  inflight: number;
  startedTotal: number;
  completedTotal: number;
  failedTotal: number;

  enqueuedTotal: Record<PriorityClass, number>;
  droppedQueueFullTotal: Record<PriorityClass, number>;
  expiredTotal: Record<PriorityClass, number>;
  deadlineMissTotal: Record<PriorityClass, number>;

  queued: Record<PriorityClass, number>;
  avgWaitMs: Record<PriorityClass, number>;
  maxWaitMs: Record<PriorityClass, number>;
};

export class PriorityScheduler<T> {
  constructor(cfg: SchedulerConfig, deps: { now: () => number });

  enqueue(job: Job<T>): { ok: true } | { ok: false; reason: "queue_full" | "expired" };

  start(handler: (job: Job<T>) => Promise<void>): Promise<void>;
  stop(): void;

  snapshot(): MetricsSnapshot;
}
```

---

## Semantics bắt buộc

### Enqueue / Admission

* Nếu `now > deadlineAt`:

  * `expirePolicy=drop` ⇒ reject ngay (expired)
  * hoặc send_to_dlq (stretch)
* Nếu queue của class đầy:

  * reject (queue_full) **hoặc** overload policy: drop lớp thấp hơn để nhận lớp cao hơn (stretch)

### Dequeue / Scheduling

Mỗi lần có slot trống (inflight < concurrency):

* Chọn job theo policy:

  * **WFQ**: lấy theo pattern tỷ lệ (vd P0:P1:P2 = 8:3:1) lặp vòng, có skip khi queue trống
  * **Aging**: compute dynamic score tăng theo thời gian chờ, chọn max score

### Deadline miss

* Khi bắt đầu xử lý job nếu `now > deadlineAt`:

  * nếu `allowLatePolicy=drop` ⇒ drop + deadlineMiss++
  * nếu `process_with_tag` ⇒ vẫn chạy nhưng tag “late” (metrics vẫn tăng miss)

### Anti-starvation requirement

* Dù P0 luôn đầy, P2 vẫn phải được chạy **thỉnh thoảng** (WFQ đảm bảo, aging đảm bảo nếu cấu hình đúng).

---

## Core implementation plan (chọn 1 làm core)

### Option A (khuyến nghị core): WFQ / Weighted Round-Robin

* Mỗi class có queue riêng: `q[P0], q[P1], q[P2]`
* Xây một “schedule wheel” theo weights: ví dụ weights 8/3/1 ⇒ wheel:
  `P0 P0 P0 P0 P0 P0 P0 P0 P1 P1 P1 P2`
* Con trỏ `cursor` chạy vòng. Mỗi lần dequeue:

  * thử wheel[cursor], nếu queue trống thì advance cursor tiếp cho tới khi tìm được queue có job (hoặc none)
* Ưu: deterministic, dễ test, không starve nếu weights > 0.

### Option B (stretch): Aging score

* Score = basePriorityScore + agingBonus
* agingBonus có thể: `log2(waitMs / halfLifeMs + 1)` hoặc tuyến tính clamp
* Mỗi dequeue chọn job score lớn nhất trong head của mỗi queue (không scan toàn queue, chỉ head để O(1) per class)

---

## Starter skeleton (WFQ core) — nhìn vào là code

```ts
// src/priorityScheduler.ts
export type PriorityClass = "P0" | "P1" | "P2";

export type Job<T> = {
  id: string;
  klass: PriorityClass;
  createdAt: number;
  deadlineAt: number;
  payload: T;
};

export type SchedulerConfig = {
  concurrency: number;
  maxQueue: Record<PriorityClass, number>;
  expirePolicy: "drop" | "send_to_dlq";
  allowLatePolicy: "drop" | "process_with_tag";
  mode: "wfq" | "aging";
  weights?: Record<PriorityClass, number>;
};

type CountersByClass = Record<PriorityClass, number>;

type WaitStats = { sum: number; max: number; n: number };

export class PriorityScheduler<T> {
  private q: Record<PriorityClass, Array<Job<T>>> = { P0: [], P1: [], P2: [] };
  private inflight = 0;
  private stopped = false;

  // WFQ wheel
  private wheel: PriorityClass[] = [];
  private cursor = 0;

  // metrics
  private startedTotal = 0;
  private completedTotal = 0;
  private failedTotal = 0;

  private enqueuedTotal: CountersByClass = { P0: 0, P1: 0, P2: 0 };
  private droppedQueueFullTotal: CountersByClass = { P0: 0, P1: 0, P2: 0 };
  private expiredTotal: CountersByClass = { P0: 0, P1: 0, P2: 0 };
  private deadlineMissTotal: CountersByClass = { P0: 0, P1: 0, P2: 0 };

  private wait: Record<PriorityClass, WaitStats> = {
    P0: { sum: 0, max: 0, n: 0 },
    P1: { sum: 0, max: 0, n: 0 },
    P2: { sum: 0, max: 0, n: 0 },
  };

  constructor(private cfg: SchedulerConfig, private deps: { now: () => number }) {
    if (cfg.mode !== "wfq") throw new Error("TODO: implement aging in stretch");
    const w = cfg.weights ?? { P0: 8, P1: 3, P2: 1 };
    this.wheel = [
      ...Array(w.P0).fill("P0"),
      ...Array(w.P1).fill("P1"),
      ...Array(w.P2).fill("P2"),
    ];
    if (this.wheel.length === 0) throw new Error("invalid weights");
  }

  enqueue(job: Job<T>): { ok: true } | { ok: false; reason: "queue_full" | "expired" } {
    const now = this.deps.now();

    if (now > job.deadlineAt) {
      this.expiredTotal[job.klass]++;
      return { ok: false, reason: "expired" };
    }

    const cap = this.cfg.maxQueue[job.klass];
    if (this.q[job.klass].length >= cap) {
      this.droppedQueueFullTotal[job.klass]++;
      return { ok: false, reason: "queue_full" };
    }

    this.q[job.klass].push(job);
    this.enqueuedTotal[job.klass]++;
    return { ok: true };
  }

  stop() {
    this.stopped = true;
  }

  snapshot() {
    const avgWaitMs = { P0: 0, P1: 0, P2: 0 } as Record<PriorityClass, number>;
    const maxWaitMs = { P0: 0, P1: 0, P2: 0 } as Record<PriorityClass, number>;
    for (const k of ["P0","P1","P2"] as PriorityClass[]) {
      const ws = this.wait[k];
      avgWaitMs[k] = ws.n === 0 ? 0 : Math.round(ws.sum / ws.n);
      maxWaitMs[k] = ws.max;
    }

    return {
      inflight: this.inflight,
      startedTotal: this.startedTotal,
      completedTotal: this.completedTotal,
      failedTotal: this.failedTotal,
      enqueuedTotal: { ...this.enqueuedTotal },
      droppedQueueFullTotal: { ...this.droppedQueueFullTotal },
      expiredTotal: { ...this.expiredTotal },
      deadlineMissTotal: { ...this.deadlineMissTotal },
      queued: { P0: this.q.P0.length, P1: this.q.P1.length, P2: this.q.P2.length },
      avgWaitMs,
      maxWaitMs,
    };
  }

  async start(handler: (job: Job<T> & { late?: boolean }) => Promise<void>) {
    while (!this.stopped) {
      while (this.inflight < this.cfg.concurrency) {
        const job = this.dequeueNext();
        if (!job) break;

        this.spawn(job, handler);
      }

      // simple yield; in tests you can replace with injected sleep if needed
      await Promise.resolve();
      if (this.inflight === 0 && this.totalQueued() === 0) {
        // avoid hot loop
        await new Promise(r => setTimeout(r, 0));
      }
    }

    // drain
    while (this.inflight > 0) await new Promise(r => setTimeout(r, 1));
  }

  private totalQueued() {
    return this.q.P0.length + this.q.P1.length + this.q.P2.length;
  }

  private dequeueNext(): Job<T> | null {
    // WFQ wheel scan until find a non-empty queue, at most wheel.length steps
    for (let i = 0; i < this.wheel.length; i++) {
      const k = this.wheel[this.cursor];
      this.cursor = (this.cursor + 1) % this.wheel.length;

      const qq = this.q[k];
      if (qq.length === 0) continue;

      const job = qq.shift()!;
      return job;
    }
    return null;
  }

  private spawn(job: Job<T>, handler: (job: Job<T> & { late?: boolean }) => Promise<void>) {
    this.inflight++;
    this.startedTotal++;

    const now = this.deps.now();
    const waitMs = Math.max(0, now - job.createdAt);
    const ws = this.wait[job.klass];
    ws.sum += waitMs; ws.n += 1; ws.max = Math.max(ws.max, waitMs);

    let late = false;
    if (now > job.deadlineAt) {
      this.deadlineMissTotal[job.klass]++;
      if (this.cfg.allowLatePolicy === "drop") {
        // drop without running handler
        this.inflight--;
        this.completedTotal++;
        return;
      }
      late = true;
    }

    (async () => {
      try {
        await handler({ ...job, late });
        this.completedTotal++;
      } catch {
        this.failedTotal++;
        this.completedTotal++;
      } finally {
        this.inflight--;
      }
    })().catch(() => {});
  }
}
```

---

## Test plan bắt buộc (SLA-aware + anti-starvation)

> Dùng `vitest` + injected `now()` (fake clock). Bạn không cần real timers; chạy loop theo microtasks.

### Test 1 — Priority dominance (P0 wins)

* enqueue 50 P0 (deadline xa), 50 P2
* weights 8:3:1, concurrency=1
* chạy xử lý 20 jobs đầu
* expect số P0 xử lý > P2 rõ rệt (ví dụ ≥ 15/20)

### Test 2 — No starvation (P2 still progresses)

* enqueue P0 “vô hạn”: mỗi lần handler P0 xong lại enqueue P0 mới (simulate constant VIP load)
* enqueue 10 P2 một lần
* chạy đủ lâu (ví dụ 200 dispatches)
* expect P2 được xử lý ít nhất X (với weights 8:3:1 thì P2 ~ 1/12, kỳ vọng ≥ 10 trong 200 là hợp lý)

### Test 3 — Queue caps & drop per class

* maxQueue P2=2
* enqueue 5 P2
* expect 2 ok, 3 queue_full, droppedQueueFullTotal.P2 = 3

### Test 4 — Expire on enqueue

* now=1000, job.deadlineAt=900
* enqueue => expired
* expiredTotal[class]++

### Test 5 — Deadline miss at start

* enqueue job deadlineAt=1050 createdAt=1000
* advance now=1100 trước khi worker bắt đầu chạy nó
* allowLatePolicy=drop => deadlineMiss++ và handler không được gọi
* đổi allowLatePolicy=process_with_tag => handler gọi với `late: true`

### Test 6 — Concurrency cap

* concurrency=3
* trong handler: running++ assert running<=3

---

## Checks (Definition of Done)

* Under VIP flood, bulk vẫn tiến triển (anti-starvation).
* P0 latency/wait thấp hơn P2 (metrics wait).
* Deadline miss và expired được đếm đúng.
* Queue cap hoạt động và drop đúng class.
* Concurrency hard cap đúng.

---

## README bắt buộc (10 dòng, “ops-ready”)

1. Priority classes và SLA tương ứng
2. Policy scheduling (WFQ weights) + vì sao chọn
3. Anti-starvation guarantee (định tính + định lượng)
4. Admission control (maxQueue) và lý do fail-fast
5. Deadline semantics: expired vs miss-at-start
6. allowLate policy trade-off
7. Metrics tối thiểu để alert: deadlineMiss rate, queue depth, drop rate
8. Failure modes: mis-tuned weights làm P2 “chết đói” hoặc P0 bị jitter
9. Tuning guideline dựa trên traffic mix và latency budget
10. Link tới Bulkhead (Kata 44) và Load Shedding (Kata 46)

---

## Stretch (chọn 1 để “đúng Senior+ thật”)

1. **Aging mode**: job càng chờ lâu càng tăng score, giảm phụ thuộc weights cứng.
2. **EDF scheduling** (Earliest Deadline First) trong từng class hoặc toàn cục (rất SLA-centric).
3. **Dynamic weights**: tăng weight P0 khi deadlineMiss tăng.
4. **Per-class concurrency** (như bulkhead trong scheduler): P0 có lane riêng.
5. **DLQ for expired** + replayer tool.