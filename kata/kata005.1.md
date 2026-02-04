# Bonus Kata 05.1 — Problem+JSON Adapter

## Goal

1. Tạo `ProblemDetails` theo RFC 7807 (Problem+JSON)
2. Map:

* `ValidationResult<T>` (accumulate errors) → HTTP **400** + `invalid_request`
* `AppError` (not_found/conflict/timeout) → 404/409/504

3. Logging structured:

* luôn có `requestId`
* log lỗi kèm `error.type`, `field`, `count`, `status`

---

## Constraints

* ❌ Không throw trong handler (trừ bug thật)
* ❌ Không `any`
* ✅ Response shape stable (client dễ dùng)
* ✅ Không leak PII (đừng log full email)

---

## Deliverables

File `kata05_1.ts` gồm:

1. `ProblemDetails` type
2. `toProblemDetails(...)`
3. `handleCreateUserProfile(req)` giả lập HTTP request/response
4. Demo chạy console cho 3 cases

---

## Spec

### 1) ProblemDetails type (minimal, practical)

Bạn implement:

```ts
type ProblemDetails = {
  type: string;            // URI-ish, e.g. "https://errors.yourapp.com/invalid_request"
  title: string;           // short, human-readable
  status: number;          // HTTP status
  detail?: string;         // optional
  instance?: string;       // request path
  requestId: string;       // correlation id (custom but useful)
  errors?: { field: string; message: string }[]; // for validation
};
```

### 2) Mapping rules

* Validation errors → `status: 400`, `type: .../invalid_request`, `title: "Invalid request"`

  * `errors` là list `{field,message}`
  * `detail` có thể: `"Validation failed"`

* NotFoundError → 404, type `.../not_found`

* ConflictError → 409, type `.../conflict`

* TimeoutError → 504, type `.../timeout`

### 3) Logging rules

Tạo logger đơn giản (console JSON) với format:

* level: `"info"|"warn"|"error"`
* msg
* requestId
* status
* errorType (nếu có)
* errorCount (nếu validation)
* path

Không log raw body.

---

## Starter skeleton (điền TODO)

```ts
/* kata05_1.ts */

// ----- Reuse from kata05 / kata04.2 -----
type ValidationError = { type: "validation"; field: string; message: string };
type ValidationResult<T> =
  | { ok: true; value: T }
  | { ok: false; errors: ValidationError[] };

// App errors (from kata04_bonus)
type NotFoundError = { type: "not_found"; resource: "user" | "order"; id: string };
type ConflictError = { type: "conflict"; message: string };
type TimeoutError = { type: "timeout"; operation: string; ms: number };
type AppError = NotFoundError | ConflictError | TimeoutError;

// Minimal Result for app layer call
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

// ----- Problem Details -----
export type ProblemDetails = {
  type: string;
  title: string;
  status: number;
  detail?: string;
  instance?: string;
  requestId: string;
  errors?: { field: string; message: string }[];
};

// ----- Logger -----
type LogLevel = "info" | "warn" | "error";
function log(level: LogLevel, entry: Record<string, unknown>) {
  const line = { ts: Date.now(), level, ...entry };
  console.log(JSON.stringify(line));
}

// ----- Mapping helpers -----
const ERR_BASE = "https://errors.yourapp.com";

export function validationToProblem(
  requestId: string,
  instance: string,
  vr: { ok: false; errors: ValidationError[] }
): ProblemDetails {
  return {
    type: `${ERR_BASE}/invalid_request`,
    title: "Invalid request",
    status: 400,
    detail: "Validation failed",
    instance,
    requestId,
    errors: vr.errors.map((e) => ({ field: e.field, message: e.message })),
  };
}

export function appErrorToProblem(requestId: string, instance: string, e: AppError): ProblemDetails {
  switch (e.type) {
    case "not_found":
      return {
        type: `${ERR_BASE}/not_found`,
        title: "Not found",
        status: 404,
        detail: `${e.resource} not found`,
        instance,
        requestId,
      };
    case "conflict":
      return {
        type: `${ERR_BASE}/conflict`,
        title: "Conflict",
        status: 409,
        detail: e.message,
        instance,
        requestId,
      };
    case "timeout":
      return {
        type: `${ERR_BASE}/timeout`,
        title: "Timeout",
        status: 504,
        detail: `Operation ${e.operation} timed out after ${e.ms}ms`,
        instance,
        requestId,
      };
    default:
      // exhaustive
      const _x: never = e;
      return _x;
  }
}

// ----- Fake HTTP types -----
type HttpRequest = {
  path: string;
  requestId: string;
  body: unknown;
};

type HttpResponse =
  | { status: 200; json: unknown }
  | { status: number; json: ProblemDetails };

// ----- Domain parser (stub): plug your kata05.parseUserProfile here -----
type UserProfile = { email: string; name: string; tags: string[]; marketingOptIn: boolean; birthYear?: number };

// TODO: Replace with your real parseUserProfile from kata05
function parseUserProfile(_body: unknown): ValidationResult<UserProfile> {
  return { ok: false, errors: [{ type: "validation", field: "email", message: "TODO: plug real parser" }] };
}

// ----- App service (stub): pretend save profile -----
function saveUserProfile(_profile: UserProfile): Result<{ id: string }, AppError> {
  // TODO: for demo: if email includes "404" -> not_found, "409" -> conflict, "timeout" -> timeout
  return { ok: true, value: { id: "u1" } };
}

// ----- Handler -----
export function handleCreateUserProfile(req: HttpRequest): HttpResponse {
  const { requestId, path } = req;

  const parsed = parseUserProfile(req.body);
  if (!parsed.ok) {
    const problem = validationToProblem(requestId, path, parsed);

    log("warn", {
      msg: "request.validation_failed",
      requestId,
      path,
      status: problem.status,
      errorType: "invalid_request",
      errorCount: parsed.errors.length,
    });

    return { status: problem.status, json: problem };
  }

  const saved = saveUserProfile(parsed.value);
  if (!saved.ok) {
    const problem = appErrorToProblem(requestId, path, saved.error);

    log("error", {
      msg: "request.failed",
      requestId,
      path,
      status: problem.status,
      errorType: saved.error.type,
    });

    return { status: problem.status, json: problem };
  }

  log("info", {
    msg: "request.ok",
    requestId,
    path,
    status: 200,
  });

  return { status: 200, json: { id: saved.value.id } };
}

// ----- Demo -----
const demos: HttpRequest[] = [
  { path: "/user-profile", requestId: "req_1", body: { email: "bad", name: "M" } }, // 400
  { path: "/user-profile", requestId: "req_2", body: { email: "user404@example.com", name: "Max" } }, // 404 (when you implement save stub)
  { path: "/user-profile", requestId: "req_3", body: { email: "ok@example.com", name: "Max" } }, // 200
];

for (const r of demos) {
  const res = handleCreateUserProfile(r);
  console.log("RESPONSE:", JSON.stringify(res, null, 2));
}
```

---

## Required behaviors

Khi bạn nối parser thật + implement `saveUserProfile` stub:

1. Invalid body → response 400 + `errors[]` đầy đủ
2. AppError `not_found/conflict/timeout` → 404/409/504 đúng
3. Log JSON ra console có `requestId`, `status`, `errorType` (khi lỗi)
4. Không leak request body vào log

---

## Checks (Definition of Done)

* `appErrorToProblem` exhaustive (thêm error mới mà quên map → TS báo)
* `ProblemDetails` stable shape
* Handler không throw, chỉ return response
* Log “warn” cho validation, “error” cho app failures, “info” cho success

---

## Stretch (rất production)

1. Add `traceId` / `spanId` (nếu bạn dùng tracing sau này)
2. Rate-limit validation logs (sampling) để tránh spam
3. `errors` trả thêm `code` (machine-readable) ngoài message
4. Map `ValidationError.field` thành JSON Pointer (ví dụ `/email`, `/tags/1`)