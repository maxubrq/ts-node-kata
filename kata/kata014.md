## Kata 14 — AbortController End-to-End

**Cancel request từ HTTP → service → repo → DB call** (propagate signal + cleanup + classify error)

### Goal

1. “Request cancelled” ở HTTP layer phải **abort** toàn bộ chain (service/repo/db)
2. Không leak timer/listener, không chạy tiếp side effects sau khi abort
3. Error boundary phân loại:

* cancel/timeout = operational
* bug = programmer

---

## Constraints

* ❌ Không `any`
* ✅ Mọi IO nhận `AbortSignal`
* ✅ Repo/service không biết HTTP, chỉ nhận `signal`
* ✅ DB call mô phỏng phải thực sự stop khi abort
* ✅ Có demo: client disconnect hoặc timeout làm huỷ request

---

## Deliverables

File `kata14.ts` gồm:

1. `HttpRequest` có `signal`
2. `withRequestAbort(req, handler)` (wire “client disconnect” + server timeout)
3. `createOrderHandler(req)` → service → repo → db
4. `withTimeout` (reuse Kata 13) để server-side timeout
5. Demo 4 cases:

* ok
* client abort giữa chừng
* server timeout
* bug (programmer error) để thấy boundary khác

---

## Spec

### Types

```ts
type HttpRequest = {
  path: string;
  requestId: string;
  body: unknown;
  signal: AbortSignal;        // comes from HTTP server / client disconnect
};

type HttpResponse = { status: number; body: unknown };

type CancelledError = { type: "cancelled"; reason: "client_disconnect" | "timeout"; operation: string };
type TimeoutError = { type: "timeout"; ms: number; operation: string };

type AppError =
  | { type: "validation"; field: string; message: string }
  | CancelledError
  | TimeoutError
  | { type: "downstream_unavailable"; service: string };
```

### Rules

* Handler **không throw** AppError raw string; throw object typed hoặc return Result (tuỳ bạn). Kata này dùng `throw AppError` để boundary bắt.
* Khi `req.signal` abort → propagate abort xuống `dbQuery(signal)` ngay lập tức.
* Server timeout (ví dụ 80ms) phải abort request dù client không abort.

---

## Starter skeleton (điền TODO)

```ts
/* kata14.ts */
import { AsyncLocalStorage } from "node:async_hooks";

// ---------- Result (optional) ----------
type HttpResponse = { status: number; body: unknown };

type HttpRequest = {
  path: string;
  requestId: string;
  body: unknown;
  signal: AbortSignal;
};

// ---------- Errors ----------
type CancelledError = { type: "cancelled"; reason: "client_disconnect" | "timeout"; operation: string };
type TimeoutError = { type: "timeout"; ms: number; operation: string };

type AppError =
  | { type: "validation"; field: string; message: string }
  | CancelledError
  | TimeoutError
  | { type: "downstream_unavailable"; service: string };

const ERR_BASE = "https://errors.yourapp.com";

// ---------- Context (optional, from kata12) ----------
type Ctx = Readonly<{ requestId: string }>;
const als = new AsyncLocalStorage<Ctx>();

function withContext<T>(requestId: string, fn: () => T): T {
  return als.run(Object.freeze({ requestId }), fn);
}
function getRequestId(): string {
  return als.getStore()?.requestId ?? "no-ctx";
}

// ---------- Utilities ----------
function isRecord(x: unknown): x is Record<string, unknown> {
  return typeof x === "object" && x !== null;
}

function log(level: "info" | "warn" | "error", msg: string, extra: Record<string, unknown> = {}) {
  console.log(JSON.stringify({ ts: Date.now(), level, msg, requestId: getRequestId(), ...extra }));
}

// ---------- Timeout wrapper (reuse Kata 13 style) ----------
async function withTimeout<T>(op: {
  operation: string;
  ms: number;
  signal?: AbortSignal;
  run: (signal: AbortSignal) => Promise<T>;
}): Promise<T> {
  const controller = new AbortController();

  // TODO: link upstream abort to internal abort
  // TODO: timer abort after ms
  // TODO: cleanup listener + timer
  // TODO: throw TimeoutError / CancelledError accordingly
  return op.run(controller.signal);
}

// ---------- Abortable sleep ----------
function sleep(ms: number, signal: AbortSignal, operation: string): Promise<void> {
  return new Promise((resolve, reject) => {
    const t = setTimeout(resolve, ms);

    const onAbort = () => {
      clearTimeout(t);
      reject({ type: "cancelled", reason: "client_disconnect", operation } satisfies CancelledError);
    };

    if (signal.aborted) {
      clearTimeout(t);
      reject({ type: "cancelled", reason: "client_disconnect", operation } satisfies CancelledError);
      return;
    }

    signal.addEventListener("abort", onAbort, { once: true });
  });
}

// ---------- DB layer (must accept signal) ----------
async function dbQueryUser(userId: string, signal: AbortSignal) {
  log("info", "db.query.start", { userId });
  await sleep(60, signal, "dbQueryUser");
  log("info", "db.query.ok", { userId });
  return { userId, name: "Max" };
}

async function dbInsertOrder(userId: string, amount: number, signal: AbortSignal) {
  log("info", "db.insert.start", { userId, amount });
  await sleep(60, signal, "dbInsertOrder");
  log("info", "db.insert.ok", { userId, amount });
  return { orderId: "ord_deadbeefc0de" };
}

// ---------- Repo ----------
class OrderRepo {
  async createOrder(userId: string, amount: number, signal: AbortSignal) {
    // TODO: call dbInsertOrder with same signal
    return dbInsertOrder(userId, amount, signal);
  }
}

// ---------- Service ----------
class OrderService {
  constructor(private repo: OrderRepo) {}

  async createOrder(body: unknown, signal: AbortSignal) {
    // TODO: parse body (minimal validation)
    if (!isRecord(body)) throw { type: "validation", field: "body", message: "must be object" } satisfies AppError;

    const userId = body.userId;
    const amount = body.amount;

    if (typeof userId !== "string" || userId.length === 0)
      throw { type: "validation", field: "userId", message: "must be string" } satisfies AppError;

    if (typeof amount !== "number" || !Number.isFinite(amount) || amount <= 0)
      throw { type: "validation", field: "amount", message: "must be number > 0" } satisfies AppError;

    // TODO: propagate signal into DB calls
    const user = await dbQueryUser(userId, signal);

    // TODO: simulate programmer bug if body.bug === true (for demo)
    if (body.bug === true) {
      // programmer error
      (null as unknown as { x: number }).x;
    }

    const created = await this.repo.createOrder(user.userId, amount, signal);
    return { orderId: created.orderId };
  }
}

// ---------- HTTP boundary (maps AppError vs programmer errors) ----------
function isAppError(e: unknown): e is AppError {
  if (!isRecord(e)) return false;
  const t = e.type;
  if (typeof t !== "string") return false;
  return (
    t === "validation" ||
    t === "cancelled" ||
    t === "timeout" ||
    t === "downstream_unavailable"
  );
}

function toResponse(req: HttpRequest, e: AppError): HttpResponse {
  switch (e.type) {
    case "validation":
      return {
        status: 400,
        body: {
          type: `${ERR_BASE}/invalid_request`,
          title: "Invalid request",
          status: 400,
          instance: req.path,
          requestId: req.requestId,
          errors: [{ field: e.field, message: e.message }],
        },
      };
    case "cancelled":
      // many teams use 499 (nginx) for client closed request; in pure HTTP use 499-ish internally or 408/400.
      return {
        status: 499,
        body: {
          type: `${ERR_BASE}/cancelled`,
          title: "Request cancelled",
          status: 499,
          instance: req.path,
          requestId: req.requestId,
          detail: `${e.reason} during ${e.operation}`,
        },
      };
    case "timeout":
      return {
        status: 504,
        body: {
          type: `${ERR_BASE}/timeout`,
          title: "Timeout",
          status: 504,
          instance: req.path,
          requestId: req.requestId,
          detail: `Timed out after ${e.ms}ms at ${e.operation}`,
        },
      };
    case "downstream_unavailable":
      return {
        status: 503,
        body: {
          type: `${ERR_BASE}/downstream_unavailable`,
          title: "Downstream unavailable",
          status: 503,
          instance: req.path,
          requestId: req.requestId,
          detail: `Service ${e.service} unavailable`,
        },
      };
    default: {
      const _x: never = e;
      return _x;
    }
  }
}

type Handler = (req: HttpRequest) => Promise<HttpResponse>;

function withHttpBoundary(handler: Handler): Handler {
  return async (req) => {
    return withContext(req.requestId, async () => {
      try {
        return await handler(req);
      } catch (e) {
        if (isAppError(e)) {
          const res = toResponse(req, e);
          log("warn", "request.operational_error", { status: res.status, errorType: e.type });
          return res;
        }
        log("error", "request.programmer_error", {
          errorName: e instanceof Error ? e.name : "Unknown",
          message: e instanceof Error ? e.message : String(e),
          stack: e instanceof Error ? e.stack : undefined,
        });
        return {
          status: 500,
          body: {
            type: `${ERR_BASE}/internal`,
            title: "Internal Server Error",
            status: 500,
            instance: req.path,
            requestId: req.requestId,
          },
        };
      }
    });
  };
}

// ---------- HTTP handler ----------
const repo = new OrderRepo();
const svc = new OrderService(repo);

const createOrderHandler: Handler = async (req) => {
  // TODO: wrap whole operation with server timeout using withTimeout(...)
  const result = await svc.createOrder(req.body, req.signal);
  return { status: 200, body: result };
};

// ---------- Demo: simulate HTTP server signals ----------
function makeRequest(body: unknown, opts: { abortAfterMs?: number } = {}): HttpRequest {
  const controller = new AbortController();
  if (opts.abortAfterMs !== undefined) {
    setTimeout(() => controller.abort("client disconnected"), opts.abortAfterMs);
  }
  return { path: "/orders", requestId: "req_" + Math.random().toString(16).slice(2), body, signal: controller.signal };
}

async function run() {
  const h = withHttpBoundary(createOrderHandler);

  // 1) OK (should succeed; but note db calls total 120ms, so only ok if no server timeout)
  console.log("CASE1 OK:", await h(makeRequest({ userId: "u1", amount: 100 })));

  // 2) Client abort before db returns
  console.log("CASE2 client abort:", await h(makeRequest({ userId: "u1", amount: 100 }, { abortAfterMs: 30 })));

  // 3) Server timeout (you implement in createOrderHandler with withTimeout ms < 120)
  console.log("CASE3 server timeout:", await h(makeRequest({ userId: "u1", amount: 100 })));

  // 4) Programmer bug
  console.log("CASE4 bug:", await h(makeRequest({ userId: "u1", amount: 100, bug: true })));
}

run().catch((e) => console.error("UNEXPECTED:", e));
```

---

## Tasks (cụ thể bạn phải làm)

### Task A — Implement `withTimeout`

* Link `req.signal` (upstream) → internal controller
* Timeout hits → abort internal controller + throw `TimeoutError`
* Upstream abort → throw `CancelledError` reason `"client_disconnect"`
* Cleanup timer + listener

### Task B — Wrap handler end-to-end

Trong `createOrderHandler`:

* gọi `withTimeout({ operation:"http.POST /orders", ms: 80, signal: req.signal, run: (signal) => svc.createOrder(req.body, signal) })`
* ms bạn chọn phải đảm bảo CASE3 xảy ra (vì db tổng 120ms)

### Task C — Ensure DB stops

* Khi abort/timeout xảy ra, `sleep` phải reject sớm, db logs phải không in “ok” sau khi abort.

### Task D — Boundary mapping

* Cancelled → status 499 (ok) hoặc 408 (tuỳ bạn). Kata dùng 499 để rõ “client closed”.

---

## Required behaviors (phải đạt)

* CASE2: trả 499 cancelled, và db không log “ok” sau đó
* CASE3: trả 504 timeout, và db/insert không hoàn thành
* CASE4: trả 500 internal, log error stack (programmer error)
* Không leak timers/listeners (chạy nhiều lần không tăng listener count mãi)

---

## Stretch (đúng production)

1. **Deadline propagation**: nếu context có `deadlineMs`, timeout ms = min(80, timeLeft)
2. **Different cancellation reasons**:

   * client disconnect
   * timeout
   * manual cancel (admin)
3. **Partial work prevention**: nếu user loaded xong nhưng insert chưa xong mà abort → không insert
4. **Metrics**: count cancelled/timeout per operation