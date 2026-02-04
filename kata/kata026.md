# Kata 26 — Problem+JSON: chuẩn hoá error response

## Goal

1. Tạo một **error contract** ổn định cho toàn API:

* client luôn nhận cùng shape
* dễ debug + dễ alert
* không leak thông tin nhạy cảm

2. Phân loại lỗi production-grade:

* **validation** (4xx)
* **auth/authz** (401/403)
* **not found** (404)
* **conflict** (409)
* **rate limit** (429)
* **upstream/downstream** (502/503/504)
* **internal** (500)

3. Có một **error boundary** duy nhất:

* controller/handler *không tự format error*
* chỉ `throw AppError(...)` hoặc trả `Result` và boundary convert thành Problem+JSON

---

## Constraints

* ✅ Response `Content-Type: application/problem+json` cho error.
* ✅ Không được trả raw `Error.message` cho lỗi internal.
* ✅ Không leak stack trace, env vars, SQL, token, secrets.
* ✅ Luôn có `traceId`/`requestId` trong error body (để correlate logs/traces).
* ✅ Có tests cho mapping + header + shape.
* ❌ Không được “tự chế 10 kiểu error JSON khác nhau”.

---

## Spec: Problem Details (tối thiểu + extension hợp lý)

### Base shape

```ts
export type ProblemJson = {
  type: string;     // URL hoặc URN phân loại lỗi
  title: string;    // nhãn ngắn, ổn định
  status: number;   // HTTP status
  detail?: string;  // mô tả dành cho client (không leak)
  instance?: string;// request path hoặc error instance id

  // extensions (cho production)
  traceId: string;  // bắt buộc
  code?: string;    // stable machine code, ví dụ "E_VALIDATION"
  errors?: Array<{ path: string; message: string; code?: string }>; // cho validation
  retryable?: boolean; // cho upstream/downstream
};
```

### Type URL convention

Bạn chọn 1 convention và dùng xuyên suốt. Ví dụ:

* `https://errors.example.com/validation`
* `https://errors.example.com/unauthorized`
* `https://errors.example.com/forbidden`
* `https://errors.example.com/not-found`
* `https://errors.example.com/conflict`
* `https://errors.example.com/rate-limit`
* `https://errors.example.com/upstream`
* `https://errors.example.com/internal`

> `type` nên **ổn định**, không chứa message động.

---

## Deliverables (repo structure)

`kata-26/`

* `src/problem.ts` (types + builders)
* `src/errors.ts` (AppError + subclasses / factory)
* `src/boundary.ts` (error boundary middleware/handler wrapper)
* `src/app.ts` (demo route)
* `test/kata26.test.ts` (mapping + no leak)

---

## Bài toán demo (đủ để test thật)

API có các routes:

1. `POST /v1/orders`

   * validation fail ⇒ 400 + problem+json + `errors[]`
2. `GET /v1/orders/:id`

   * không tồn tại ⇒ 404
3. `POST /v1/payments`

   * gọi upstream timeout ⇒ 504 + `retryable: true`
4. `GET /v1/admin`

   * thiếu auth ⇒ 401 / có auth nhưng thiếu quyền ⇒ 403
5. Bất kỳ bug runtime (throw Error) ⇒ 500 internal (không leak)

---

## Starter Skeleton (điền TODO)

### `src/problem.ts`

```ts
export type ProblemJson = {
  type: string;
  title: string;
  status: number;
  detail?: string;
  instance?: string;

  traceId: string;
  code?: string;
  errors?: Array<{ path: string; message: string; code?: string }>;
  retryable?: boolean;
};

export function problemBase(input: {
  type: string;
  title: string;
  status: number;
  traceId: string;
  instance?: string;
  detail?: string;
  code?: string;
  errors?: ProblemJson["errors"];
  retryable?: boolean;
}): ProblemJson {
  return {
    type: input.type,
    title: input.title,
    status: input.status,
    traceId: input.traceId,
    instance: input.instance,
    detail: input.detail,
    code: input.code,
    errors: input.errors,
    retryable: input.retryable,
  };
}

export const PROBLEM_TYPES = {
  validation: "https://errors.example.com/validation",
  unauthorized: "https://errors.example.com/unauthorized",
  forbidden: "https://errors.example.com/forbidden",
  notFound: "https://errors.example.com/not-found",
  conflict: "https://errors.example.com/conflict",
  rateLimit: "https://errors.example.com/rate-limit",
  upstream: "https://errors.example.com/upstream",
  internal: "https://errors.example.com/internal",
} as const;
```

### `src/errors.ts`

```ts
export type AppErrorKind =
  | "VALIDATION"
  | "UNAUTHORIZED"
  | "FORBIDDEN"
  | "NOT_FOUND"
  | "CONFLICT"
  | "RATE_LIMIT"
  | "UPSTREAM"
  | "INTERNAL";

export class AppError extends Error {
  readonly kind: AppErrorKind;
  readonly status: number;
  readonly code: string;
  readonly safeDetail?: string;
  readonly data?: unknown;
  readonly retryable?: boolean;

  constructor(input: {
    kind: AppErrorKind;
    message: string;        // internal message (logs)
    status: number;
    code: string;           // stable machine code
    safeDetail?: string;    // safe for client
    data?: unknown;         // optional details (e.g., validation issues)
    retryable?: boolean;
    cause?: unknown;
  }) {
    super(input.message);
    this.kind = input.kind;
    this.status = input.status;
    this.code = input.code;
    this.safeDetail = input.safeDetail;
    this.data = input.data;
    this.retryable = input.retryable;

    // Node 16+ supports Error cause (optional)
    // (but don't rely on it for response)
    // @ts-ignore
    if (input.cause) this.cause = input.cause;
  }
}

// Factories (để codebase dùng thống nhất)
export function errValidation(issues: Array<{ path: string; message: string; code?: string }>) {
  return new AppError({
    kind: "VALIDATION",
    message: "Request validation failed",
    status: 400,
    code: "E_VALIDATION",
    safeDetail: "Request validation failed",
    data: { issues },
  });
}

export function errNotFound(resource: string, id: string) {
  return new AppError({
    kind: "NOT_FOUND",
    message: `Not found: ${resource} ${id}`,
    status: 404,
    code: "E_NOT_FOUND",
    safeDetail: "Resource not found",
    data: { resource },
  });
}

export function errUpstreamTimeout(service: string) {
  return new AppError({
    kind: "UPSTREAM",
    message: `Upstream timeout: ${service}`,
    status: 504,
    code: "E_UPSTREAM_TIMEOUT",
    safeDetail: "Upstream timeout",
    data: { service },
    retryable: true,
  });
}

// TODO: unauthorized/forbidden/conflict/rateLimit/internal factories
```

### `src/boundary.ts`

```ts
import { AppError } from "./errors";
import { problemBase, PROBLEM_TYPES, type ProblemJson } from "./problem";

export type HttpLikeRequest = {
  method: string;
  path: string;
  headers: Record<string, string | undefined>;
  traceId: string; // injected earlier (middleware)
};

export type HttpLikeResponse = {
  status: number;
  headers: Record<string, string>;
  body: unknown;
};

function toProblemJson(req: HttpLikeRequest, err: unknown): ProblemJson {
  // 1) AppError => map by kind
  if (err instanceof AppError) {
    const instance = `${req.method} ${req.path}`;
    switch (err.kind) {
      case "VALIDATION": {
        const issues = (err.data as any)?.issues as Array<{ path: string; message: string; code?: string }> | undefined;
        return problemBase({
          type: PROBLEM_TYPES.validation,
          title: "Validation Error",
          status: 400,
          traceId: req.traceId,
          instance,
          detail: err.safeDetail,
          code: err.code,
          errors: issues ?? [],
        });
      }
      case "NOT_FOUND": {
        return problemBase({
          type: PROBLEM_TYPES.notFound,
          title: "Not Found",
          status: 404,
          traceId: req.traceId,
          instance,
          detail: err.safeDetail,
          code: err.code,
        });
      }
      case "UPSTREAM": {
        return problemBase({
          type: PROBLEM_TYPES.upstream,
          title: "Upstream Error",
          status: err.status,
          traceId: req.traceId,
          instance,
          detail: err.safeDetail,
          code: err.code,
          retryable: err.retryable ?? false,
        });
      }
      default: {
        // TODO: map other AppError kinds
        return problemBase({
          type: PROBLEM_TYPES.internal,
          title: "Internal Server Error",
          status: 500,
          traceId: req.traceId,
          instance,
          detail: "Unexpected error",
          code: "E_INTERNAL",
        });
      }
    }
  }

  // 2) unknown error => internal (no leak)
  return problemBase({
    type: PROBLEM_TYPES.internal,
    title: "Internal Server Error",
    status: 500,
    traceId: req.traceId,
    instance: `${req.method} ${req.path}`,
    detail: "Unexpected error",
    code: "E_INTERNAL",
  });
}

export function withErrorBoundary<TReq>(
  handler: (req: TReq) => Promise<unknown> | unknown,
  mapReq: (raw: any) => TReq
) {
  return async (rawReq: any): Promise<HttpLikeResponse> => {
    const req = mapReq(rawReq);
    try {
      const out = await handler(req);
      return { status: 200, headers: { "content-type": "application/json" }, body: out };
    } catch (e) {
      const p = toProblemJson(rawReq as HttpLikeRequest, e);
      return {
        status: p.status,
        headers: { "content-type": "application/problem+json" },
        body: p,
      };
    }
  };
}
```

---

## Tests (vitest) — kiểm tra “chuẩn hoá” + “no leak”

### `test/kata26.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { withErrorBoundary } from "../src/boundary";
import { errValidation, errNotFound, errUpstreamTimeout, AppError } from "../src/errors";

function mkReq(overrides?: Partial<any>) {
  return {
    method: "GET",
    path: "/v1/orders/ord_x",
    headers: {},
    traceId: "trace_123",
    ...overrides,
  };
}

describe("kata26 - problem+json", () => {
  it("maps validation error to problem+json 400", async () => {
    const h = withErrorBoundary(
      () => { throw errValidation([{ path: "items[0].qty", message: "must be int" }]); },
      (r) => r
    );

    const res = await h(mkReq({ method: "POST", path: "/v1/orders" }));
    expect(res.status).toBe(400);
    expect(res.headers["content-type"]).toBe("application/problem+json");
    expect((res.body as any).type).toContain("/validation");
    expect((res.body as any).errors[0].path).toBe("items[0].qty");
    expect((res.body as any).traceId).toBe("trace_123");
  });

  it("maps not found to 404 without leaking internals", async () => {
    const h = withErrorBoundary(
      () => { throw errNotFound("order", "ord_x"); },
      (r) => r
    );

    const res = await h(mkReq());
    expect(res.status).toBe(404);
    expect((res.body as any).detail).toBe("Resource not found");
    expect(JSON.stringify(res.body)).not.toContain("Not found: order ord_x"); // internal message must not leak
  });

  it("maps upstream timeout to 504 and sets retryable", async () => {
    const h = withErrorBoundary(
      () => { throw errUpstreamTimeout("payment-gw"); },
      (r) => r
    );

    const res = await h(mkReq({ path: "/v1/payments" }));
    expect(res.status).toBe(504);
    expect((res.body as any).retryable).toBe(true);
  });

  it("unknown Error becomes internal 500 and does not leak stack/message", async () => {
    const h = withErrorBoundary(
      () => { throw new Error("DB password=supersecret"); },
      (r) => r
    );

    const res = await h(mkReq());
    expect(res.status).toBe(500);
    const body = res.body as any;
    expect(body.title).toBe("Internal Server Error");
    expect(JSON.stringify(body)).not.toContain("supersecret");
    expect(JSON.stringify(body)).not.toContain("password");
    expect(body.traceId).toBe("trace_123");
  });
});
```

---

## Definition of Done (Checks)

Bạn pass kata này khi:

1. **Tất cả error response** đều:

* `content-type: application/problem+json`
* body có `type`, `title`, `status`, `traceId`

2. Validation error có `errors[]` với `path` dạng `items[0].qty`.
3. Internal 500 **không leak**:

* stack
* raw message
* secrets

4. Mapping nhất quán giữa các kind (401/403/404/409/429/5xx) — ít nhất implement 4 kind trong kata này (validation, not-found, upstream, internal).

---

## Stretch (Senior+ thật)

1. **Attach `instance` = full request URI** + query string (nhưng nhớ không leak secrets trong query).
2. **Security headers** cho error responses:

   * `cache-control: no-store`
3. **Error logging policy**:

   * log internal message + stack + traceId
   * nhưng response chỉ safe
4. **Convert ZodError/AjvError trực tiếp** thành `errValidation(...)` (nối liền Kata 25 → 26).
5. **Rate-limit problem**: include `retryAfterSeconds` extension + set `Retry-After` header.