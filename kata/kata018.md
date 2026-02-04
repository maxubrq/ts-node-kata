# Kata 18 — Unhandled Rejection Policy

**Thiết kế policy xử lý `unhandledRejection` / `uncaughtException`: log + metrics + quyết định “crash hay không”** (production Node standard)

## Goal

1. Xây 1 module `fatalPolicy.ts`:

* attach process handlers:

  * `unhandledRejection`
  * `uncaughtException`
  * (bonus) `warning`

2. Classify lỗi:

* **Operational** (có thể recover / request-scoped) → log + metrics, *có thể không crash*
* **Programmer/Fatal** → log + *crash gracefully* (nối Kata 17)

3. Provide **one switch** để chọn policy:

* `policy: "crash" | "log_only"` (hoặc theo env)

4. Demo rõ ràng 4 case:

* unhandled rejection operational (vd TimeoutError)
* unhandled rejection programmer (TypeError)
* uncaughtException programmer
* rejection handled đúng (không trigger)

---

## Constraints

* ❌ Không nuốt lỗi im lặng
* ✅ Nếu quyết định crash: phải **graceful shutdown** (drain/close) trước khi exit
* ✅ Không leak PII trong logs
* ✅ Không `any`
* ✅ Idempotent: attach handlers nhiều lần không bị duplicate

---

## Deliverables

File `kata18.ts` gồm:

1. Types `AppError` (operational)
2. `isOperationalError(e)` (reuse Kata 11)
3. `installFatalPolicy(opts)`
4. `gracefulExit(code)` hook (stub hoặc reuse Kata 17 shutdown manager)
5. Demo runner (4 scenarios)

---

## Spec

### Operational error union (fixed)

```ts
type AppError =
  | { type: "validation"; field: string; message: string }
  | { type: "timeout"; operation: string; ms: number }
  | { type: "cancelled"; operation: string; reason: "client_disconnect" | "timeout" }
  | { type: "downstream_unavailable"; service: string }
  | { type: "rate_limited"; retryAfterMs: number };
```

### Install API

```ts
type FatalPolicy = "crash" | "log_only";

function installFatalPolicy(opts: {
  policy: FatalPolicy;
  // called before exiting
  onBeforeExit?: (reason: { kind: "unhandledRejection" | "uncaughtException"; error: unknown }) => Promise<void>;
  logger?: (level: "error" | "warn", msg: string, extra: Record<string, unknown>) => void;
}): void;
```

### Decision rules (bạn phải implement)

* `uncaughtException` → **always crash** (log_only vẫn log nhưng exit recommended; kata: crash bắt buộc)
* `unhandledRejection`:

  * nếu `isOperationalError(reason)` và policy = `log_only` → log warn, continue
  * nếu operational nhưng policy = `crash` → crash
  * nếu không operational (unknown/programmer) → crash

> Rationale: unhandled rejection thường là bug hoặc missing await; nhiều org chọn crash để tránh inconsistent state.

---

## Starter skeleton (điền TODO)

```ts
/* kata18.ts */

type AppError =
  | { type: "validation"; field: string; message: string }
  | { type: "timeout"; operation: string; ms: number }
  | { type: "cancelled"; operation: string; reason: "client_disconnect" | "timeout" }
  | { type: "downstream_unavailable"; service: string }
  | { type: "rate_limited"; retryAfterMs: number };

type FatalPolicy = "crash" | "log_only";

function isRecord(x: unknown): x is Record<string, unknown> {
  return typeof x === "object" && x !== null;
}

function isOperationalError(e: unknown): e is AppError {
  if (!isRecord(e)) return false;
  const t = e.type;
  if (typeof t !== "string") return false;
  return (
    t === "validation" ||
    t === "timeout" ||
    t === "cancelled" ||
    t === "downstream_unavailable" ||
    t === "rate_limited"
  );
}

function defaultLogger(level: "error" | "warn", msg: string, extra: Record<string, unknown>) {
  console.log(JSON.stringify({ ts: Date.now(), level, msg, ...extra }));
}

let installed = false;

export function installFatalPolicy(opts: {
  policy: FatalPolicy;
  onBeforeExit?: (reason: { kind: "unhandledRejection" | "uncaughtException"; error: unknown }) => Promise<void>;
  logger?: (level: "error" | "warn", msg: string, extra: Record<string, unknown>) => void;
}): void {
  if (installed) return;
  installed = true;

  const logger = opts.logger ?? defaultLogger;

  async function gracefulExit(code: number, reason: { kind: "unhandledRejection" | "uncaughtException"; error: unknown }) {
    // TODO:
    // - call opts.onBeforeExit (await)
    // - flush logs/metrics if needed (stub)
    // - then process.exit(code)
    try {
      if (opts.onBeforeExit) await opts.onBeforeExit(reason);
    } finally {
      process.exit(code);
    }
  }

  process.on("uncaughtException", (error) => {
    logger("error", "fatal.uncaughtException", {
      errorName: error.name,
      message: error.message,
      stack: error.stack,
    });

    // TODO: always crash
    void gracefulExit(1, { kind: "uncaughtException", error });
  });

  process.on("unhandledRejection", (reason) => {
    const operational = isOperationalError(reason);

    logger(operational ? "warn" : "error", "fatal.unhandledRejection", {
      operational,
      reason: operational ? reason : (reason instanceof Error ? reason.message : String(reason)),
      stack: reason instanceof Error ? reason.stack : undefined,
    });

    // TODO:
    // - if not operational => crash
    // - if operational + policy=crash => crash
    // - if operational + policy=log_only => continue
    if (!operational || opts.policy === "crash") {
      void gracefulExit(1, { kind: "unhandledRejection", error: reason });
    }
  });

  // bonus: warnings
  process.on("warning", (w) => {
    logger("warn", "process.warning", { name: w.name, message: w.message, stack: w.stack });
  });
}

// ---------- Demo ----------
async function demo() {
  installFatalPolicy({
    policy: "log_only",
    onBeforeExit: async (r) => {
      // simulate graceful shutdown (nối Kata 17)
      defaultLogger("warn", "shutdown.begin", { kind: r.kind });
      await new Promise<void>((res) => setTimeout(res, 50));
      defaultLogger("warn", "shutdown.done", {});
    },
  });

  // CASE A: operational unhandled rejection (log_only should NOT crash)
  Promise.reject({ type: "timeout", operation: "db.query", ms: 50 } satisfies AppError);

  // CASE B: handled rejection (should NOT trigger)
  Promise.reject(new Error("handled")).catch(() => {});

  // CASE C: programmer unhandled rejection (should crash even in log_only)
  setTimeout(() => {
    Promise.reject(new TypeError("bug: undefined is not a function"));
  }, 50);

  // keep alive shortly
  await new Promise<void>((res) => setTimeout(res, 500));
}

if (process.argv[1]?.includes("kata18")) {
  demo().catch((e) => {
    console.error("DEMO FAILED:", e);
    process.exitCode = 1;
  });
}
```

---

## Tasks (bạn phải làm)

1. Implement `gracefulExit` đúng (await hook rồi exit)
2. Ensure `uncaughtException` luôn crash
3. Implement rule cho `unhandledRejection` đúng spec
4. Demo: confirm behavior:

   * operational unhandled rejection log warn, không crash (log_only)
   * programmer unhandled rejection crash (exit 1)
5. Idempotent install: gọi `installFatalPolicy` nhiều lần không attach nhiều handler

---

## Required behaviors

* Bạn nhìn log sẽ thấy:

  * `fatal.unhandledRejection` operational=true (không exit ngay ở policy log_only)
  * sau đó `fatal.unhandledRejection` operational=false (TypeError) → chạy shutdown hook → exit
* Nếu đổi policy `"crash"`: operational cũng crash

---

## Definition of Done

* Không `any`
* Không nuốt lỗi
* Có graceful shutdown hook trước exit
* Logic policy rõ, dễ audit

---

## Stretch (rất production)

1. **Per-error rate limit** cho logs (tránh spam)
2. **Exit codes** khác nhau: 1 (bug), 2 (config), 3 (OOM-ish)
3. Tự động dump diagnostics khi fatal:

   * `process.memoryUsage()`
   * event loop lag snapshot (nối Kata 16)
4. “Two-step crash”: stop accept new (Kata 17), drain 2s, rồi hard exit
5. Emit metric counters: `fatal_unhandled_rejection_total{operational=true/false}`