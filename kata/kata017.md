# Kata 17 — Graceful Shutdown

**Stop accept new → drain in-flight → shutdown clean** (Node/TS production standard)

## Goal

Bạn viết 1 “mini server” giả lập (không cần HTTP thật) có:

1. **Accept** request/jobs liên tục
2. Mỗi job mất thời gian (DB call + external call) và hỗ trợ `AbortSignal`
3. Khi nhận shutdown signal:

* **ngừng nhận job mới**
* **drain** jobs đang chạy (hoặc abort theo deadline)
* **close resources** (db pool, timers, monitors)
* exit code đúng

---

## Constraints

* ❌ Không “process.exit()” ngay lập tức
* ✅ Có state machine rõ (RUNNING → DRAINING → STOPPED)
* ✅ Không leak timers/listeners
* ✅ Có deadline cho shutdown (vd 5s): quá hạn thì abort còn lại
* ✅ Không `any`
* ✅ Log structured mỗi bước

---

## Deliverables

File `kata17.ts` gồm:

1. `ShutdownManager` (core)
2. `InFlightTracker` (đếm promise đang chạy, await drain)
3. `Server` giả lập:

* `accept(job)` chỉ khi RUNNING
* `handle(job)` tạo work (sleep + abort)

4. Demo:

* chạy 10 jobs/giây
* sau 2s trigger shutdown
* chứng minh:

  * job mới bị reject
  * jobs cũ finish hoặc bị abort khi deadline đến

---

## Spec

### State machine

```ts
type ShutdownState = "running" | "draining" | "stopped";
```

### In-flight tracker API

* `track(promise)` → tự add/remove
* `drain(deadlineMs)` → resolve khi inFlight=0 hoặc deadline reached
* `abortAll(reason)` → abort controllers của jobs (nếu bạn thiết kế)

---

## Starter skeleton (điền TODO)

```ts
/* kata17.ts */
import { performance } from "node:perf_hooks";

type ShutdownState = "running" | "draining" | "stopped";

type LogLevel = "info" | "warn" | "error";
function log(level: LogLevel, msg: string, extra: Record<string, unknown> = {}) {
  console.log(JSON.stringify({ ts: Date.now(), level, msg, ...extra }));
}

function sleep(ms: number, signal: AbortSignal, operation: string): Promise<void> {
  return new Promise((resolve, reject) => {
    const t = setTimeout(resolve, ms);
    const onAbort = () => {
      clearTimeout(t);
      reject(new Error(`aborted:${operation}`));
    };
    if (signal.aborted) {
      clearTimeout(t);
      reject(new Error(`aborted:${operation}`));
      return;
    }
    signal.addEventListener("abort", onAbort, { once: true });
  });
}

// ---------- InFlightTracker ----------
class InFlightTracker {
  private inFlight = new Set<Promise<unknown>>();

  track<T>(p: Promise<T>): Promise<T> {
    // TODO: add p, remove in finally
    return p;
  }

  count(): number {
    return this.inFlight.size;
  }

  async drain(deadlineMs: number): Promise<{ drained: boolean; remaining: number }> {
    // TODO:
    // - wait until inFlight == 0 OR time >= deadlineMs
    // - hint: poll with small interval or Promise.race on current snapshot
    return { drained: false, remaining: this.count() };
  }
}

// ---------- ShutdownManager ----------
class ShutdownManager {
  private state: ShutdownState = "running";
  private readonly tracker = new InFlightTracker();
  private readonly controllers = new Set<AbortController>();

  getState(): ShutdownState {
    return this.state;
  }

  // wraps a job so it counts as in-flight and can be aborted
  runJob<T>(name: string, fn: (signal: AbortSignal) => Promise<T>): Promise<T> {
    if (this.state !== "running") {
      return Promise.reject(new Error(`reject:${name}:server_not_running`));
    }

    const controller = new AbortController();
    this.controllers.add(controller);

    const startedAt = performance.now();
    log("info", "job.start", { name, inFlight: this.tracker.count() + 1 });

    const p = fn(controller.signal)
      .then((v) => {
        log("info", "job.ok", { name, ms: Math.round(performance.now() - startedAt) });
        return v;
      })
      .catch((e) => {
        log("warn", "job.fail", { name, ms: Math.round(performance.now() - startedAt), err: String(e) });
        throw e;
      })
      .finally(() => {
        this.controllers.delete(controller);
        log("info", "job.end", { name, inFlight: this.tracker.count() - 1 });
      });

    return this.tracker.track(p);
  }

  async shutdown(opts: { timeoutMs: number }): Promise<{ drained: boolean; remaining: number }> {
    if (this.state === "stopped") return { drained: true, remaining: 0 };

    log("warn", "shutdown.begin", { state: this.state });
    this.state = "draining";

    const deadline = Date.now() + opts.timeoutMs;

    // TODO:
    // - wait drain until deadline
    // - if not drained by deadline: abort all controllers and wait a bit more (or immediate)
    // - set state stopped
    // - return result
    this.state = "stopped";
    log("warn", "shutdown.done", { drained: true });
    return { drained: true, remaining: 0 };
  }

  private abortAll(reason: string) {
    for (const c of this.controllers) c.abort(reason);
  }
}

// ---------- Fake server workload ----------
async function handleOneJob(signal: AbortSignal): Promise<void> {
  // simulate work: db + external
  await sleep(80, signal, "db");
  await sleep(120, signal, "external");
}

// ---------- Demo ----------
async function demo() {
  const sm = new ShutdownManager();

  // generate jobs at ~10/s
  const t = setInterval(() => {
    sm.runJob("job", (signal) => handleOneJob(signal)).catch(() => {});
  }, 100);

  // trigger shutdown after 2s
  setTimeout(() => {
    log("warn", "signal.SIGTERM");
    sm.shutdown({ timeoutMs: 500 }).then((r) => log("warn", "shutdown.result", r));
    // stop accepting new
    clearInterval(t);
  }, 2_000);

  // keep alive enough to observe
  await new Promise<void>((r) => setTimeout(r, 4_000));
}

demo().catch((e) => {
  console.error("DEMO FAILED:", e);
  process.exitCode = 1;
});
```

---

## Tasks (bạn phải làm)

### Task A — Implement `InFlightTracker.track`

* add promise vào set
* remove trong `finally`

### Task B — Implement `drain(deadlineMs)`

Gợi ý 2 cách:

1. Poll mỗi 25ms đến khi `count()==0` hoặc timeout
2. “Wait all snapshot”:

   * lấy snapshot `Array.from(inFlight)`
   * `await Promise.race([Promise.allSettled(snapshot), sleepUntil(deadline)])`
   * loop cho đến hết

### Task C — Implement `ShutdownManager.shutdown`

Phải đúng thứ tự:

1. Set state `draining`
2. `await tracker.drain(deadline)`
3. Nếu chưa drained:

   * `abortAll("shutdown_timeout")`
   * `await tracker.drain(Date.now()+smallGrace)` (vd +200ms)
4. Set state `stopped`
5. Return `{drained, remaining}`

### Task D — Stop accept new

Trong `runJob`, nếu state != running → reject.

---

## Required behaviors

* Sau khi shutdown bắt đầu:

  * job mới bị reject ngay
  * jobs đang chạy vẫn hoàn thành nếu kịp deadline
  * nếu quá deadline → bị abort
* Log thể hiện rõ transition
* Không leak interval/timer (demo kết thúc sạch)

---

## Definition of Done

* State machine rõ ràng
* Draining hoạt động đúng
* Abort works end-to-end (jobs stop ở sleep/db/external)
* Không `any`

---

## Stretch (rất production)

1. **Close resources**: `closeDbPool()`, `stopLagMonitor()` (nối Kata 16)
2. **SIGINT/SIGTERM handling**: idempotent shutdown (gọi nhiều lần không sao)
3. **Health endpoint**: khi draining trả “not ready”
4. **Two-phase shutdown**:

   * phase 1: stop accept new + drain
   * phase 2: hard abort + exit
5. **Per-job deadlines**: nếu job đã gần timeout thì abort sớm