# Kata 13 — Timeouts Everywhere (timeout + abort + deadline)

## Goal

1. Viết wrapper `withTimeout(...)`:

* nhận `signal` (AbortSignal) từ bên ngoài (để propagate cancel)
* tự tạo timeout nội bộ
* khi timeout → abort + trả lỗi typed
* luôn cleanup timer

2. Viết helper `withDeadline(...)` / `timeLeftMs()` (nối Kata 12/08.1)
3. Áp dụng vào 3 loại async:

* Promise thuần (không abort được) → reject đúng, cleanup
* Fetch-like (có signal) → abort thật
* Custom “DB call” mô phỏng có thể cancel

---

## Constraints

* ❌ Không `any`
* ❌ Không swallow errors
* ✅ Nếu upstream signal abort → wrapper phải abort và return error `{type:"cancelled"}` hoặc `{type:"timeout"}` tùy nguyên nhân
* ✅ Timeout error typed, không string
* ✅ Không leak timer / listener (must remove event listeners)

---

## Deliverables

File `kata13.ts` gồm:

1. Error union: `TimeoutError | CancelledError`
2. `withTimeout` (core)
3. `withTimeoutResult` (trả `Result<T, Timeout|Cancelled|Unknown>` — optional but recommended)
4. Demo 5 cases:

   * resolves before timeout
   * times out (abort happens)
   * upstream abort
   * nested timeout with propagated deadline
   * ensuring cleanup (no double resolve/reject)

---

## Error model (fixed)

```ts
type TimeoutError = { type: "timeout"; operation: string; ms: number };
type CancelledError = { type: "cancelled"; operation: string; reason: "upstream_abort" | "timeout" };
type TimeoutishError = TimeoutError | CancelledError;
```

---

## API spec bạn phải implement

### Core wrapper

```ts
function withTimeout<T>(
  op: {
    operation: string;
    ms: number;
    signal?: AbortSignal;
    // run receives a signal that WILL be aborted on timeout/upstream abort
    run: (signal: AbortSignal) => Promise<T>;
  }
): Promise<T>;
```

Rules:

* Create internal `AbortController`
* If upstream `signal` aborts → abort internal controller + reject `CancelledError{upstream_abort}`
* If timeout hits → abort internal controller + reject `TimeoutError` (hoặc CancelledError reason timeout; nhưng spec yêu cầu TimeoutError riêng)
* If `run` resolves/rejects first → clear timer, detach listener, return result
* Ensure no “double settle”

### Result wrapper (optional but recommended)

```ts
type Result<T, E> = { ok:true; value:T } | { ok:false; error:E };

function withTimeoutResult<T>(op: ...): Promise<Result<T, TimeoutishError | { type:"unknown"; message:string }>>;
```

---

## Starter skeleton (điền TODO)

```ts
/* kata13.ts */

type TimeoutError = { type: "timeout"; operation: string; ms: number };
type CancelledError = { type: "cancelled"; operation: string; reason: "upstream_abort" | "timeout" };
type TimeoutishError = TimeoutError | CancelledError;

type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
const err = <E>(error: E): Result<never, E> => ({ ok: false, error });

// TODO: helper to make AbortSignal reason typed (Node supports AbortSignal.reason)
function abort(controller: AbortController, reason: unknown) {
  // Node 18+ supports controller.abort(reason)
  controller.abort(reason);
}

export async function withTimeout<T>(op: {
  operation: string;
  ms: number;
  signal?: AbortSignal;
  run: (signal: AbortSignal) => Promise<T>;
}): Promise<T> {
  const controller = new AbortController();

  // TODO: wire upstream abort -> abort internal
  // TODO: start timer -> abort internal
  // TODO: run(op.run(controller.signal)) with cleanup in finally
  // TODO: reject with typed TimeoutError / CancelledError (upstream)
  throw new Error("TODO");
}

export async function withTimeoutResult<T>(op: {
  operation: string;
  ms: number;
  signal?: AbortSignal;
  run: (signal: AbortSignal) => Promise<T>;
}): Promise<Result<T, TimeoutishError | { type: "unknown"; message: string }>> {
  try {
    const v = await withTimeout(op);
    return ok(v);
  } catch (e) {
    // TODO: map thrown value to typed errors
    return err({ type: "unknown", message: e instanceof Error ? e.message : String(e) });
  }
}

// ---------- Demo helpers ----------
function sleep(ms: number, signal?: AbortSignal): Promise<void> {
  return new Promise((resolve, reject) => {
    const t = setTimeout(resolve, ms);

    const onAbort = () => {
      clearTimeout(t);
      reject(new Error("aborted"));
    };

    if (signal) {
      if (signal.aborted) {
        clearTimeout(t);
        reject(new Error("aborted"));
        return;
      }
      signal.addEventListener("abort", onAbort, { once: true });
    }
  });
}

// Simulate an abortable DB call
async function fakeDb(signal: AbortSignal) {
  await sleep(80, signal);
  return { rowCount: 1 };
}

// Simulate a non-abortable call (ignores signal)
async function nonAbortable() {
  await sleep(80);
  return "ok";
}

// ---------- Run cases ----------
async function run() {
  // 1) resolves
  console.log(
    "case1",
    await withTimeoutResult({
      operation: "db.fast",
      ms: 200,
      run: fakeDb,
    })
  );

  // 2) times out (should abort)
  console.log(
    "case2",
    await withTimeoutResult({
      operation: "db.slow",
      ms: 30,
      run: fakeDb,
    })
  );

  // 3) upstream abort
  const upstream = new AbortController();
  setTimeout(() => upstream.abort("client disconnected"), 20);

  console.log(
    "case3",
    await withTimeoutResult({
      operation: "db.cancelled",
      ms: 200,
      signal: upstream.signal,
      run: fakeDb,
    })
  );

  // 4) non-abortable still times out from wrapper perspective
  console.log(
    "case4",
    await withTimeoutResult({
      operation: "legacy.call",
      ms: 10,
      run: () => nonAbortable(),
    })
  );
}

run().catch((e) => console.error("UNEXPECTED:", e));
```

---

## Required behaviors

* case1 → ok
* case2 → timeout error typed (`type:"timeout"`, correct operation, ms)
* case3 → cancelled typed (`reason:"upstream_abort"`)
* case4 → timeout typed (dù underlying vẫn chạy, wrapper must settle + cleanup)

---

## Definition of Done

* Không `any`
* Cleanup đúng: timer cleared, abort listener removed
* Không double-settle (resolve/reject không bị gọi hai lần)
* `withTimeoutResult` map error rõ ràng, không leak raw Error ra ngoài domain

---

## Stretch (Senior+/Lead)

1. **Deadline propagation**: `ms = min(ms, timeLeftFromContext())` (nối Kata 12/08.1)
2. `raceWithSignal`: reject ngay nếu signal aborted trước khi run
3. “Budgeted retries”: retry on timeout N lần nhưng tổng deadline không vượt
4. Instrumentation: log `operation`, `ms`, `elapsed`, `timedOut`, `aborted`