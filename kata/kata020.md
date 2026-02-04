# Kata 20 — Worker Threads Case

**Offload CPU-bound sang worker threads + đo throughput + chứng minh event loop lag giảm**

## Goal

Bạn xây một mini service xử lý job CPU-bound (hashing / JSON transform nặng) theo 2 mode:

1. **Single-thread (main thread)**: chạy CPU trực tiếp → event loop lag tăng, throughput kém
2. **Worker threads**: offload sang pool → main thread responsive hơn, throughput tăng

Bạn phải đo:

* throughput (jobs/s)
* p50/p95 latency (ms)
* event loop lag snapshot (reuse Kata 16)
* CPU utilization (optional)

---

## Constraints

* ❌ Không dùng `child_process`
* ✅ Dùng `worker_threads`
* ✅ Có worker pool (ít nhất 2 workers), không spawn mỗi job
* ✅ Có backpressure: queue size cap, reject/await khi đầy
* ✅ Có graceful shutdown (nối Kata 17): stop accept, drain, terminate workers
* ✅ Không `any`

---

## Deliverables

Folder `kata20/`:

* `main.ts` (producer + benchmark + event loop lag)
* `worker.ts` (CPU task implementation)
* `pool.ts` (worker pool + queue + promise correlation)
* `README.md` (cách chạy + kết quả đo)

---

# Spec

## CPU task (fixed)

Bạn implement function `heavyWork(input: string, rounds: number): string`

* Use `crypto` to hash repeatedly (CPU-bound):

  * `sha256` vòng lặp `rounds` (vd 30_000)
* Output final hex digest

> Đây là CPU-bound đủ để thấy khác biệt.

---

## Benchmark scenario

* Generate `N = 2_000` jobs
* Each job: `{ id, payload: random string, rounds }`
* Run benchmark twice:

  1. `mode="inline"`
  2. `mode="workers"` with pool size `os.cpus().length - 1` (cap 4–8 để ổn)

Metrics:

```ts
type BenchResult = {
  mode: "inline" | "workers";
  jobs: number;
  concurrency: number;
  totalMs: number;
  throughput: number; // jobs/s
  p50Ms: number;
  p95Ms: number;
  maxMs: number;
  lagP99Ms: number;  // from Kata 16 monitor snapshot
};
```

---

# Implementation skeleton

## 1) `worker.ts`

```ts
/* kata20/worker.ts */
import { parentPort } from "node:worker_threads";
import crypto from "node:crypto";

type WorkReq = { id: number; payload: string; rounds: number };
type WorkRes = { id: number; ok: true; digest: string } | { id: number; ok: false; message: string };

function heavyWork(payload: string, rounds: number): string {
  let x = payload;
  for (let i = 0; i < rounds; i++) {
    x = crypto.createHash("sha256").update(x).digest("hex");
  }
  return x;
}

if (!parentPort) throw new Error("No parentPort");

parentPort.on("message", (m: WorkReq) => {
  try {
    const digest = heavyWork(m.payload, m.rounds);
    const res: WorkRes = { id: m.id, ok: true, digest };
    parentPort!.postMessage(res);
  } catch (e) {
    const res: WorkRes = { id: m.id, ok: false, message: e instanceof Error ? e.message : String(e) };
    parentPort!.postMessage(res);
  }
});
```

---

## 2) `pool.ts`

```ts
/* kata20/pool.ts */
import { Worker } from "node:worker_threads";
import path from "node:path";

type WorkReq = { id: number; payload: string; rounds: number };
type WorkRes = { id: number; ok: true; digest: string } | { id: number; ok: false; message: string };

type Pending = { resolve: (v: WorkRes) => void; reject: (e: unknown) => void };

export class WorkerPool {
  private workers: Worker[] = [];
  private nextWorker = 0;

  private pending = new Map<number, Pending>();

  private queue: WorkReq[] = [];
  private inFlight = 0;

  constructor(private opts: { size: number; queueLimit: number; workerPath: string }) {}

  async start(): Promise<void> {
    for (let i = 0; i < this.opts.size; i++) {
      const w = new Worker(this.opts.workerPath);
      w.on("message", (res: WorkRes) => this.onMessage(res));
      w.on("error", (e) => this.onWorkerError(e));
      this.workers.push(w);
    }
  }

  // backpressure: if queue full, reject (or await capacity in stretch)
  submit(req: WorkReq): Promise<WorkRes> {
    if (this.queue.length >= this.opts.queueLimit) {
      return Promise.reject(new Error("queue_full"));
    }
    return new Promise<WorkRes>((resolve, reject) => {
      this.pending.set(req.id, { resolve, reject });
      this.queue.push(req);
      this.pump();
    });
  }

  private pump() {
    // TODO:
    // - while we have queued reqs and capacity (inFlight < workers.length)
    // - dispatch to worker in round-robin
    // - track inFlight count
  }

  private onMessage(res: WorkRes) {
    // TODO:
    // - resolve pending promise by id
    // - decrement inFlight
    // - pump next
  }

  private onWorkerError(e: unknown) {
    // TODO:
    // - fail all pending? (simpler) or fail only ones in flight
    // - for kata: fail all pending + clear
  }

  async shutdown(): Promise<void> {
    // TODO:
    // - stop accepting new? (you can set a flag)
    // - wait until inFlight==0 and queue empty (drain)
    // - terminate all workers
  }
}
```

---

## 3) `main.ts`

```ts
/* kata20/main.ts */
import crypto from "node:crypto";
import os from "node:os";
import path from "node:path";
import { performance } from "node:perf_hooks";
import { WorkerPool } from "./pool";

// reuse percentile from kata16
function percentile(sorted: number[], p: number): number {
  if (sorted.length === 0) return 0;
  const idx = (sorted.length - 1) * p;
  const lo = Math.floor(idx);
  const hi = Math.ceil(idx);
  if (lo === hi) return sorted[lo];
  const w = idx - lo;
  return sorted[lo] * (1 - w) + sorted[hi] * w;
}

function heavyWorkInline(payload: string, rounds: number): string {
  let x = payload;
  for (let i = 0; i < rounds; i++) {
    x = crypto.createHash("sha256").update(x).digest("hex");
  }
  return x;
}

type Job = { id: number; payload: string; rounds: number };

function makeJobs(n: number, rounds: number): Job[] {
  const jobs: Job[] = [];
  for (let i = 0; i < n; i++) {
    jobs.push({ id: i, payload: crypto.randomBytes(16).toString("hex"), rounds });
  }
  return jobs;
}

// simple concurrency runner
async function runConcurrent<T, R>(
  items: readonly T[],
  concurrency: number,
  fn: (item: T) => Promise<R>
): Promise<{ results: R[]; latenciesMs: number[] }> {
  const results: R[] = new Array(items.length);
  const lats: number[] = new Array(items.length);

  let idx = 0;
  const workers = Array.from({ length: concurrency }, async () => {
    while (true) {
      const i = idx;
      idx++;
      if (i >= items.length) return;

      const t0 = performance.now();
      const r = await fn(items[i]);
      const dt = performance.now() - t0;

      results[i] = r;
      lats[i] = dt;
    }
  });

  await Promise.all(workers);
  return { results, latenciesMs: lats };
}

async function benchInline(jobs: Job[], concurrency: number) {
  const t0 = performance.now();
  const { latenciesMs } = await runConcurrent(jobs, concurrency, async (j) => {
    // CPU-bound in main thread => "async" doesn't help; still blocks
    heavyWorkInline(j.payload, j.rounds);
    return true;
  });
  const totalMs = performance.now() - t0;
  return { totalMs, latenciesMs };
}

async function benchWorkers(jobs: Job[], concurrency: number, pool: WorkerPool) {
  const t0 = performance.now();
  const { latenciesMs } = await runConcurrent(jobs, concurrency, async (j) => {
    const res = await pool.submit(j);
    if (!res.ok) throw new Error(res.message);
    return true;
  });
  const totalMs = performance.now() - t0;
  return { totalMs, latenciesMs };
}

function summarize(latenciesMs: number[]) {
  const sorted = [...latenciesMs].sort((a, b) => a - b);
  return {
    p50Ms: percentile(sorted, 0.5),
    p95Ms: percentile(sorted, 0.95),
    maxMs: sorted.length ? sorted[sorted.length - 1] : 0,
  };
}

async function main() {
  const jobs = makeJobs(2000, 15_000);
  const concurrency = 32;

  // TODO: integrate kata16 lag monitor and record lag p99 during each run

  console.log("INLINE...");
  const a = await benchInline(jobs, concurrency);
  console.log({ mode: "inline", totalMs: a.totalMs, throughput: jobs.length / (a.totalMs / 1000), ...summarize(a.latenciesMs) });

  console.log("WORKERS...");
  const pool = new WorkerPool({
    size: Math.min(Math.max(os.cpus().length - 1, 2), 8),
    queueLimit: 256,
    workerPath: path.join(__dirname, "worker.js"),
  });
  await pool.start();

  const b = await benchWorkers(jobs, concurrency, pool);
  console.log({ mode: "workers", totalMs: b.totalMs, throughput: jobs.length / (b.totalMs / 1000), ...summarize(b.latenciesMs) });

  await pool.shutdown();
}

main().catch((e) => {
  console.error("FAILED:", e);
  process.exitCode = 1;
});
```

---

## Tasks (bạn phải làm)

1. Implement `WorkerPool.pump / onMessage / shutdown` đúng:

* dispatch round-robin
* inFlight <= workers.length
* backpressure: queueLimit enforced
* resolve đúng promise theo `id`

2. Add **lag monitor** (reuse Kata 16) để đo lag p99 trong inline vs worker mode
3. Report `BenchResult` (in console hoặc JSON)
4. Prove:

* workers mode có **throughput cao hơn** hoặc **lag thấp hơn đáng kể**
* main thread không bị “đơ” như inline

---

## Required behaviors

* Inline: event loop lag p99 tăng rõ khi chạy (do CPU block)
* Workers: lag p99 thấp hơn, main thread responsive hơn
* Pool không leak pending promises (pending Map về 0 sau run)
* Shutdown clean: stop accept new, drain, terminate workers

---

## Definition of Done

* WorkerPool stable (no lost responses, no hanging)
* Throughput/latency measured và in ra
* Lag monitor integrated
* No `any`

---

## Stretch (rất production)

1. **Transferable**: dùng `ArrayBuffer` thay string để giảm serialization overhead
2. **Adaptive pool**: tăng/giảm workers theo lag / queue depth (nối Kata 16/17)
3. **Per-job timeout + abort**: cancel job nếu quá hạn (nối Kata 13/14)
4. **Crash policy**: worker chết → respawn + fail inflight jobs (nối Kata 18)
5. **Batching**: gửi batch jobs 10–50 per message để giảm overhead