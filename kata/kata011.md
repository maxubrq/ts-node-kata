# Kata 11 — Error Boundaries (Operational vs Programmer)

## Goal

1. Xây **error taxonomy** rõ ràng:

* **Operational errors**: expected trong runtime (timeout, not found, validation, rate limit, downstream unavailable…) → handle, return error response, metrics/log.
* **Programmer errors**: bug (undefined access, invariant violated, impossible state) → **crash fast** hoặc bubble up để process manager restart.

2. Tạo **error boundary** cho HTTP handler:

* Map operational → Problem+JSON + status
* Programmer → log + 500 (hoặc crash, tùy mode)

3. Có **test cases** chứng minh phân loại đúng.

---

## Constraints

* ❌ Không “catch all rồi nuốt”
* ✅ Operational errors phải có:

  * `type` (discriminated union)
  * `message` safe
  * `cause?` optional
* ✅ Programmer errors: không wrap bừa thành operational (đừng che bug)
* ✅ Có `assertNever` cho exhaustive checks

---

## Deliverables

File `kata11.ts` gồm:

1. `AppError` (operational union)
2. Type guard `isOperationalError(e): e is AppError`
3. `toProblemDetails(AppError) -> {status, body}`
4. `withHttpBoundary(handler)` wrapper
5. Demo: 6 scenarios (validation, not_found, timeout, conflict, thrown Error, invariant violation)

---

## Error model (fixed)

Bạn dùng union:

```ts
type AppError =
  | { type: "validation"; field: string; message: string }
  | { type: "not_found"; resource: string; id: string }
  | { type: "conflict"; message: string }
  | { type: "timeout"; operation: string; ms: number }
  | { type: "rate_limited"; retryAfterMs: number }
  | { type: "downstream_unavailable"; service: string };
```

Programmer errors = mọi thứ **không phải** `AppError` (ví dụ `TypeError`, `ReferenceError`, `assertNever` throw, invariant throw).

---

## HTTP boundary spec

### Input/Output

```ts
type HttpRequest = { path: string; requestId: string; body: unknown };
type HttpResponse = { status: number; body: unknown };
type Handler = (req: HttpRequest) => Promise<HttpResponse>;
```

### Rules

* Nếu handler throw **AppError** → convert sang status + Problem+JSON
* Nếu handler throw **Error / unknown**:

  * log level `error`
  * response 500 Problem+JSON (không lộ stack)
  * **Stretch:** nếu `NODE_ENV !== "production"` thì rethrow để crash test/dev

---

## Starter skeleton (điền TODO)

```ts
/* kata11.ts */

// ---------- Types ----------
type HttpRequest = { path: string; requestId: string; body: unknown };
type HttpResponse = { status: number; body: unknown };
type Handler = (req: HttpRequest) => Promise<HttpResponse>;

type ProblemDetails = {
  type: string;
  title: string;
  status: number;
  detail?: string;
  instance?: string;
  requestId: string;
  // optional field errors
  errors?: { field: string; message: string }[];
};

type AppError =
  | { type: "validation"; field: string; message: string }
  | { type: "not_found"; resource: string; id: string }
  | { type: "conflict"; message: string }
  | { type: "timeout"; operation: string; ms: number }
  | { type: "rate_limited"; retryAfterMs: number }
  | { type: "downstream_unavailable"; service: string };

const ERR_BASE = "https://errors.yourapp.com";

// ---------- TODO 1: isOperationalError ----------
function isRecord(x: unknown): x is Record<string, unknown> {
  return typeof x === "object" && x !== null;
}

export function isOperationalError(e: unknown): e is AppError {
  // TODO: narrow by checking object + string type + required fields by case
  return false;
}

// ---------- TODO 2: map to ProblemDetails ----------
function assertNever(x: never): never {
  throw new Error("Unhandled case: " + JSON.stringify(x));
}

export function toProblemDetails(req: HttpRequest, e: AppError): HttpResponse {
  // TODO: map each error type to:
  // - status (validation 400, not_found 404, conflict 409, timeout 504, rate_limited 429, downstream 503)
  // - problem.type URI
  // - title
  // - detail safe
  // - instance req.path + requestId
  switch (e.type) {
    case "validation":
      return {
        status: 400,
        body: {
          type: `${ERR_BASE}/invalid_request`,
          title: "Invalid request",
          status: 400,
          detail: "Validation failed",
          instance: req.path,
          requestId: req.requestId,
          errors: [{ field: e.field, message: e.message }],
        } satisfies ProblemDetails,
      };

    // TODO other cases

    default:
      return assertNever(e);
  }
}

// ---------- Logger ----------
type LogLevel = "info" | "warn" | "error";
function log(level: LogLevel, msg: string, extra: Record<string, unknown>) {
  console.log(JSON.stringify({ ts: Date.now(), level, msg, ...extra }));
}

// ---------- TODO 3: withHttpBoundary ----------
export function withHttpBoundary(handler: Handler): Handler {
  return async (req) => {
    try {
      return await handler(req);
    } catch (e) {
      if (isOperationalError(e)) {
        const res = toProblemDetails(req, e);
        log("warn", "request.operational_error", {
          requestId: req.requestId,
          path: req.path,
          status: res.status,
          errorType: e.type,
        });
        return res;
      }

      // programmer / unknown error
      log("error", "request.unhandled_exception", {
        requestId: req.requestId,
        path: req.path,
        // do NOT include stack in response; ok to include in logs
        errorName: e instanceof Error ? e.name : "Unknown",
        message: e instanceof Error ? e.message : String(e),
        stack: e instanceof Error ? e.stack : undefined,
      });

      const res: HttpResponse = {
        status: 500,
        body: {
          type: `${ERR_BASE}/internal`,
          title: "Internal Server Error",
          status: 500,
          instance: req.path,
          requestId: req.requestId,
        } satisfies ProblemDetails,
      };

      // Stretch option: crash in non-prod
      // if (process.env.NODE_ENV !== "production") throw e;

      return res;
    }
  };
}

// ---------- Demo handlers ----------
const okHandler: Handler = async (_req) => ({ status: 200, body: { ok: true } });

const validationHandler: Handler = async () => {
  throw { type: "validation", field: "email", message: "invalid email" } satisfies AppError;
};

const notFoundHandler: Handler = async () => {
  throw { type: "not_found", resource: "user", id: "u1" } satisfies AppError;
};

const timeoutHandler: Handler = async () => {
  throw { type: "timeout", operation: "charge", ms: 50 } satisfies AppError;
};

const programmerBugHandler: Handler = async () => {
  // Programmer error (TypeError)
  const x: any = null;
  return { status: 200, body: x.foo }; // will throw at runtime
};

const invariantViolationHandler: Handler = async () => {
  // programmer error (explicit invariant)
  throw new Error("Invariant violated: impossible state");
};

// ---------- Run demos ----------
async function run() {
  const req: HttpRequest = { path: "/demo", requestId: "req_1", body: {} };

  for (const h of [
    okHandler,
    validationHandler,
    notFoundHandler,
    timeoutHandler,
    programmerBugHandler,
    invariantViolationHandler,
  ]) {
    const wrapped = withHttpBoundary(h);
    const res = await wrapped(req);
    console.log("RESPONSE:", res.status, JSON.stringify(res.body));
  }
}

run().catch((e) => console.error("UNEXPECTED:", e));
```

> Trong `programmerBugHandler` có `any` để cố tình tạo bug runtime. Trong bài làm “sạch”, bạn có thể bỏ `any` và tạo bug kiểu khác, nhưng mục tiêu ở đây là **boundary xử lý đúng**.

---

## Required behaviors (phải đạt)

1. Validation → **400** + `errors[]`
2. Not found → **404**
3. Timeout → **504**
4. Rate limited → **429** (bạn phải implement mapping)
5. Downstream unavailable → **503**
6. Programmer errors → **500** (và log level error, có stack trong log, không trả stack cho client)

---

## Definition of Done

* `isOperationalError` phân loại đúng (không quá lỏng; object random không được coi là AppError)
* `toProblemDetails` exhaustive (thêm type mới mà quên map → TS báo)
* Boundary không nuốt bug một cách im lặng
* Response body không leak stack/PII

---

## Stretch (đúng production)

1. `cause` chain: operational errors có `cause?: unknown`, log root cause
2. “Crash-on-programmer-error” trong production (tùy hệ):

   * return 500 rồi `process.exit(1)` để restart clean (cẩn thận inflight)
3. Error metrics: `counter{error_type,status}` + latency buckets
4. Integrate với AsyncLocalStorage context (Kata 08.1) để auto attach requestId