## Kata 49 — Fan-out Control: parallel calls có cap + cancel

## Goal

1. Build utility **fan-out executor**: chạy N lời gọi downstream song song nhưng có **cap** (`maxConcurrency`) + **cancel** (AbortSignal) + **deadline**.
2. Support 3 modes production-grade:

   * **All-or-cancel**: một fail/timeout ⇒ cancel phần còn lại
   * **Quorum**: lấy đủ K kết quả ⇒ cancel phần còn lại
   * **Best-effort**: chạy hết, gom successes + errors (không cancel)
3. Có **result shaping** rõ ràng: successes/errors + per-call latency + aborted count.
4. Có **metrics**: started, completed, canceled, timedOut, maxInflightObserved.
5. Tests deterministic: cap invariant, cancel propagation, early-exit behavior.

---

## Context

Fan-out hay gặp trong:

* aggregate API: gọi profile + orders + recommendations + inventory
* search across shards/replicas
* “parallel calls” trong microservices

Nếu không kiểm soát:

* bắn 100 requests một lúc → pool cạn, tail latency nổ
* call kéo dài vô hạn → request bị treo
* đã đủ kết quả vẫn tiếp tục chạy → lãng phí + tăng load

---

## Constraints

* ❌ Không dùng thư viện `p-limit` / async pool có sẵn.
* ✅ Bắt buộc dùng `AbortController` để cancel.
* ✅ Bắt buộc có `deadlineMs` (timeout tổng cho whole fan-out).
* ✅ Hard cap concurrency: không vượt `maxConcurrency` trong bất kỳ thời điểm nào.
* ✅ Không leak: listener abort phải cleanup, promise settle đúng 1 lần.
* ✅ Có test cho “racey” cases: fail sớm, timeout sớm, quorum đạt sớm.

---

## Deliverables

1. `src/fanout.ts` — implementation
2. `src/errors.ts` — `FanoutTimeoutError`, `FanoutAbortedError`
3. `src/downstream.ts` — fake call hỗ trợ AbortSignal + delay/fail
4. `test/fanout.test.ts` — unit tests (vitest)
5. `README.md` — semantics + trade-offs

---

## API bạn sẽ build

```ts
export type FanoutMode =
  | { kind: "ALL_OR_CANCEL" }                 // any failure => cancel all
  | { kind: "QUORUM"; need: number }          // stop when got K successes
  | { kind: "BEST_EFFORT" };                  // never cancel due to failures

export type FanoutOptions = {
  maxConcurrency: number;
  deadlineMs: number;                         // total deadline for the whole fan-out
  signal?: AbortSignal;                       // caller cancellation
  mode: FanoutMode;
};

export type CallSpec<T> = {
  id: string;
  run: (ctx: { signal: AbortSignal }) => Promise<T>;
};

export type FanoutResult<T> = {
  ok: boolean;
  reason?: "deadline" | "aborted" | "all_or_cancel_failed" | "quorum_unreachable";

  successes: Array<{ id: string; value: T; ms: number }>;
  errors: Array<{ id: string; error: string; ms: number }>;

  stats: {
    started: number;
    completed: number;
    canceled: number;
    timedOut: number;
    maxInflightObserved: number;
    durationMs: number;
  };
};

export async function fanout<T>(
  calls: CallSpec<T>[],
  opts: FanoutOptions,
  deps?: { now?: () => number; rand?: () => number }
): Promise<FanoutResult<T>>;
```

---

## Semantics bắt buộc

### Concurrency cap

* Luôn chỉ chạy tối đa `maxConcurrency` calls đang “inflight”.
* Khi một call settle (success/fail/abort/timeout), scheduler start call tiếp theo (nếu còn).

### Deadline (global)

* Sau `deadlineMs`, cancel tất cả inflight + stop start calls mới.
* Kết quả `ok=false reason=deadline` trừ khi trước đó đã đạt quorum.

### Cancel propagation

* Nếu `opts.signal` aborted ⇒ cancel tất cả, stop scheduling, return `reason=aborted`.

### Modes

* **ALL_OR_CANCEL**: bất kỳ error/timeout nào ⇒ cancel còn lại, return `ok=false reason=all_or_cancel_failed` (nhưng vẫn trả collected results).
* **QUORUM(need=K)**: khi đủ K successes ⇒ cancel còn lại và `ok=true`.

  * Nếu hết call mà successes < K ⇒ `ok=false reason=quorum_unreachable`.
* **BEST_EFFORT**: chạy tới khi hết hoặc deadline/abort, luôn gom kết quả.

---

## Starter skeleton (nhìn vào làm)

```ts
// src/fanout.ts
export type FanoutMode =
  | { kind: "ALL_OR_CANCEL" }
  | { kind: "QUORUM"; need: number }
  | { kind: "BEST_EFFORT" };

export type FanoutOptions = {
  maxConcurrency: number;
  deadlineMs: number;
  signal?: AbortSignal;
  mode: FanoutMode;
};

export type CallSpec<T> = {
  id: string;
  run: (ctx: { signal: AbortSignal }) => Promise<T>;
};

export type FanoutResult<T> = {
  ok: boolean;
  reason?: "deadline" | "aborted" | "all_or_cancel_failed" | "quorum_unreachable";
  successes: Array<{ id: string; value: T; ms: number }>;
  errors: Array<{ id: string; error: string; ms: number }>;
  stats: {
    started: number;
    completed: number;
    canceled: number;
    timedOut: number;
    maxInflightObserved: number;
    durationMs: number;
  };
};

export async function fanout<T>(
  calls: CallSpec<T>[],
  opts: FanoutOptions,
  deps: { now?: () => number } = {}
): Promise<FanoutResult<T>> {
  const now = deps.now ?? (() => Date.now());

  if (opts.maxConcurrency <= 0) throw new Error("maxConcurrency must be > 0");
  if (opts.deadlineMs <= 0) throw new Error("deadlineMs must be > 0");

  const startAt = now();
  const successes: FanoutResult<T>["successes"] = [];
  const errors: FanoutResult<T>["errors"] = [];

  let started = 0;
  let completed = 0;
  let canceled = 0;
  let timedOut = 0;

  let inflight = 0;
  let maxInflightObserved = 0;

  let reason: FanoutResult<T>["reason"] | undefined;
  let ok = false;

  const globalAC = new AbortController();
  const onParentAbort = () => globalAC.abort();
  if (opts.signal) {
    if (opts.signal.aborted) globalAC.abort();
    else opts.signal.addEventListener("abort", onParentAbort, { once: true });
  }

  let deadlineId: NodeJS.Timeout | undefined;
  const deadlinePromise = new Promise<"deadline">((resolve) => {
    deadlineId = setTimeout(() => resolve("deadline"), opts.deadlineMs);
  });

  const pending = calls.slice(); // queue
  const running = new Map<string, AbortController>(); // per-call controllers

  const stopAll = (why: "deadline" | "aborted" | "all_or_cancel_failed") => {
    reason = why;
    globalAC.abort();
    for (const [, ac] of running) ac.abort();
  };

  const quorumNeed = opts.mode.kind === "QUORUM" ? opts.mode.need : undefined;

  const shouldStopScheduling = () => {
    if (globalAC.signal.aborted) return true;
    if (opts.mode.kind === "QUORUM" && quorumNeed !== undefined) {
      if (successes.length >= quorumNeed) return true;
    }
    if (opts.mode.kind === "ALL_OR_CANCEL" && reason === "all_or_cancel_failed") return true;
    return false;
  };

  const spawnOne = () => {
    if (pending.length === 0) return;
    if (shouldStopScheduling()) return;

    const spec = pending.shift()!;
    const ac = new AbortController();

    // link global -> per-call
    const onGlobalAbort = () => ac.abort();
    if (globalAC.signal.aborted) ac.abort();
    else globalAC.signal.addEventListener("abort", onGlobalAbort, { once: true });

    running.set(spec.id, ac);
    inflight++;
    maxInflightObserved = Math.max(maxInflightObserved, inflight);
    started++;

    const t0 = now();

    spec.run({ signal: ac.signal })
      .then((value) => {
        const ms = now() - t0;
        successes.push({ id: spec.id, value, ms });
      })
      .catch((e) => {
        const ms = now() - t0;
        const name = e instanceof Error ? e.name : "Error";
        const msg = e instanceof Error ? e.message : String(e);
        errors.push({ id: spec.id, error: `${name}:${msg}`, ms });

        // if aborted due to our cancellation, count as canceled, not "business error"
        if (ac.signal.aborted) {
          canceled++;
        }

        if (opts.mode.kind === "ALL_OR_CANCEL" && !globalAC.signal.aborted) {
          stopAll("all_or_cancel_failed");
        }
      })
      .finally(() => {
        running.delete(spec.id);
        globalAC.signal.removeEventListener("abort", onGlobalAbort);
        inflight--;
        completed++;

        // keep filling slots
        while (inflight < opts.maxConcurrency && pending.length > 0 && !shouldStopScheduling()) {
          spawnOne();
        }
      });
  };

  // start initial batch
  while (inflight < opts.maxConcurrency && pending.length > 0 && !shouldStopScheduling()) {
    spawnOne();
  }

  // wait until done OR deadline OR parent abort OR quorum
  const donePromise = new Promise<"done">((resolve) => {
    const tick = () => {
      if (opts.mode.kind === "QUORUM" && quorumNeed !== undefined && successes.length >= quorumNeed) {
        resolve("done");
        return;
      }
      if (pending.length === 0 && inflight === 0) {
        resolve("done");
        return;
      }
      if (globalAC.signal.aborted) {
        resolve("done");
        return;
      }
      setTimeout(tick, 0);
    };
    tick();
  });

  const raced = await Promise.race([
    deadlinePromise,
    donePromise,
    new Promise<"aborted">((resolve) => {
      if (globalAC.signal.aborted) resolve("aborted");
      else globalAC.signal.addEventListener("abort", () => resolve("aborted"), { once: true });
    }),
  ]);

  if (raced === "deadline") {
    timedOut++;
    stopAll("deadline");
  }

  if (opts.signal?.aborted) {
    stopAll("aborted");
  }

  // If quorum achieved, stop remaining
  if (opts.mode.kind === "QUORUM" && quorumNeed !== undefined && successes.length >= quorumNeed) {
    // cancel remaining inflight and ignore pending
    for (const [, ac] of running) ac.abort();
    globalAC.abort();
    ok = true;
    reason = undefined;
  }

  // wait for inflight to settle quickly after cancel
  while (inflight > 0) await new Promise(r => setTimeout(r, 0));

  if (!ok) {
    if (reason === "deadline") ok = false;
    else if (reason === "aborted") ok = false;
    else if (opts.mode.kind === "ALL_OR_CANCEL" && reason === "all_or_cancel_failed") ok = false;
    else if (opts.mode.kind === "QUORUM") {
      ok = successes.length >= (quorumNeed ?? Infinity);
      if (!ok) reason = "quorum_unreachable";
    } else {
      // best effort: ok=true if at least one success? your choice; define clearly.
      ok = true;
      reason = undefined;
    }
  }

  // cleanup
  if (deadlineId) clearTimeout(deadlineId);
  if (opts.signal) opts.signal.removeEventListener("abort", onParentAbort);

  const durationMs = now() - startAt;

  return {
    ok,
    reason,
    successes,
    errors,
    stats: {
      started,
      completed,
      canceled,
      timedOut,
      maxInflightObserved,
      durationMs,
    },
  };
}
```

> Bài này intentionally “căng” vì fan-out dễ dính race. Bạn sẽ phải chỉnh phần “done loop” để tránh polling thô (stretch: dùng Promise counters). Nhưng skeleton đủ để bạn implement sạch.

---

## Fake downstream helper (để test cap/cancel/quorum)

```ts
// src/downstream.ts
export async function fakeCall(
  ms: number,
  opts: { signal: AbortSignal; fail?: boolean }
): Promise<string> {
  await new Promise<void>((resolve, reject) => {
    const t = setTimeout(resolve, ms);
    const onAbort = () => { clearTimeout(t); reject(Object.assign(new Error("aborted"), { name: "AbortError" })); };
    if (opts.signal.aborted) return onAbort();
    opts.signal.addEventListener("abort", onAbort, { once: true });
  });
  if (opts.fail) throw new Error("fail");
  return "ok";
}
```

---

## Test plan bắt buộc (vitest)

### Test 1 — Concurrency cap invariant

* maxConcurrency=3, tạo 20 calls mỗi call delay 50ms
* trong `run()` tăng `running++`, assert `running<=3`, finally giảm
* expect `maxInflightObserved===3`

### Test 2 — ALL_OR_CANCEL cancels remaining on first failure

* 1 call fail nhanh (10ms), 9 calls chậm (200ms)
* mode ALL_OR_CANCEL
* expect result.ok=false reason=all_or_cancel_failed
* expect `canceled > 0` và số successes nhỏ

### Test 3 — QUORUM stops early

* need=3, có 10 calls, 3 calls nhanh success, còn lại rất chậm
* expect ok=true
* expect tổng started có thể >3 (do concurrency), nhưng durationMs << thời gian chạy hết
* expect nhiều calls bị canceled

### Test 4 — Deadline cancels all

* deadlineMs=50, các calls delay 200
* expect reason=deadline, ok=false, canceled tăng

### Test 5 — Parent abort cancels

* abort signal sau 30ms
* expect reason=aborted

### Test 6 — BEST_EFFORT collects errors and successes

* mix fail/success
* expect ok=true (theo định nghĩa bạn chọn) và errors/successes đúng số

---

## Checks (Definition of Done)

* Không vượt `maxConcurrency` (assert).
* Deadline và AbortSignal cancel thật (downstream nhận signal và dừng).
* Mode semantics đúng:

  * ALL_OR_CANCEL fail sớm, cancel phần còn lại
  * QUORUM đủ K thì stop sớm
  * BEST_EFFORT gom hết (trừ deadline/abort)
* Metrics snapshot hợp lý (started/completed/canceled).
* Không leak event listeners.

---

## README bắt buộc (10 dòng, ops-grade)

1. Fan-out gây overload kiểu gì (pool, retries, tail latency)
2. Cap concurrency cứu gì
3. Deadline vs per-call timeout (khác nhau)
4. Cancel propagation và vì sao bắt buộc
5. 3 modes và khi nào dùng (all-or-cancel/quorum/best-effort)
6. Trade-offs: quorum có thể trả “partial truth”
7. Failure modes: hung call, never-resolving promise, listener leak
8. Metrics cần có để tune maxConcurrency
9. Integration với Bulkhead (44) + Load Shedding (46) + Circuit Breaker (43)
10. Stretch ideas

---

## Stretch (Senior+ thật)

Chọn 1:

1. **Per-call timeout** + deadline propagation: mỗi call có `timeoutMs`, nhưng global deadline vẫn là hard cap.
2. **Budget-based fanout**: stop nếu tổng cost vượt budget (vd shard cost).
3. **Hedged requests**: sau T ms chưa xong thì launch replica call, dùng cái về trước, cancel cái còn lại.
4. **Structured concurrency**: implement bằng `TaskGroup` style (JS) để không polling.
5. **Result ranking**: với quorum, chọn “best” theo latency/score.