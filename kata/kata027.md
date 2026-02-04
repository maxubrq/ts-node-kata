# Kata 27 — Structured Logging: JSON logs với request_id + user_id

## Goal

1. Mọi log là **JSON một dòng** (machine-friendly), có field chuẩn:

* `timestamp`, `level`, `msg` (hoặc `event`)
* `request_id` (bắt buộc)
* `trace_id` (nếu có, hoặc dùng chung với request_id)
* `user_id` (nếu đã auth, optional)
* `method`, `path`, `status`, `latency_ms`
* `error` object (khi lỗi), không leak secrets

2. **Propagate context** xuyên async boundary bằng `AsyncLocalStorage`:

* gọi `logger.info()` ở deep layer vẫn tự có `request_id`, `user_id`.

3. Có **HTTP request logging** chuẩn:

* 1 log lúc request start (optional)
* 1 log lúc request end (bắt buộc) có latency + status
* 1 log khi error (bắt buộc)

4. **Redaction**:

* không log `authorization`, `cookie`, `password`, `token`, `secret`, etc.
* log body/query chỉ khi debug và có redact

---

## Constraints

* ✅ Không `console.log` trực tiếp (chỉ qua logger).
* ✅ Log JSON (không phải string format).
* ✅ Có AsyncLocalStorage context.
* ✅ Có test verify “log có request_id + user_id”.
* ✅ Có test verify “authorization/password bị redacted”.
* ❌ Không được “bắn log một đống” vô nghĩa; log theo event.

> Gợi ý tooling: `pino` (nhanh, chuẩn JSON). (Đúng như plan repo của bạn.)

---

## Context (bài toán demo)

API có route:

* `POST /v1/orders`

  * header `x-request-id` optional
  * header `authorization: Bearer <token>` optional (giả lập decode ra userId)
  * response `201` nếu OK
  * response `400` nếu validation fail (dùng Kata 25/26)

Bạn cần logging cho:

* request lifecycle
* domain event: `order.created`
* error event: `request.failed`

---

## Deliverables

`kata-27/`

* `src/logger.ts` (pino base + helpers)
* `src/context.ts` (AsyncLocalStorage: set/get context)
* `src/http-middleware.ts` (inject request_id/user_id + request end log)
* `src/app.ts` (demo route + deep function log)
* `test/kata27.test.ts` (verify logs + redaction)

---

## Log Contract (chuẩn hoá fields)

### Request context

```ts
export type RequestContext = {
  request_id: string;
  user_id?: string;
  trace_id?: string;
};
```

### Log events bạn phải có

* `http.request.completed`
* `http.request.failed`
* `order.created`

> Đặt `event` thay vì “msg dài dòng”. `msg` vẫn có thể có nhưng `event` phải là primary.

---

## Starter Skeleton (điền TODO)

### `src/context.ts`

```ts
import { AsyncLocalStorage } from "node:async_hooks";

export type RequestContext = {
  request_id: string;
  user_id?: string;
  trace_id?: string;
};

const als = new AsyncLocalStorage<RequestContext>();

export function runWithContext<T>(ctx: RequestContext, fn: () => T): T {
  return als.run(ctx, fn);
}

export function getContext(): RequestContext | undefined {
  return als.getStore();
}
```

### `src/logger.ts`

```ts
import pino from "pino";
import { getContext } from "./context";

// TODO: configure redact paths (authorization, cookie, password, token, secret)
export const baseLogger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  redact: {
    paths: [
      "req.headers.authorization",
      "req.headers.cookie",
      "*.password",
      "*.token",
      "*.secret",
      "*.authorization",
    ],
    remove: true,
  },
});

// Logger facade: auto-attach context fields
export const logger = {
  info(obj: Record<string, unknown>, msg?: string) {
    const ctx = getContext();
    baseLogger.info({ ...obj, ...(ctx ?? {}) }, msg);
  },
  warn(obj: Record<string, unknown>, msg?: string) {
    const ctx = getContext();
    baseLogger.warn({ ...obj, ...(ctx ?? {}) }, msg);
  },
  error(obj: Record<string, unknown>, msg?: string) {
    const ctx = getContext();
    baseLogger.error({ ...obj, ...(ctx ?? {}) }, msg);
  },
};
```

### `src/http-middleware.ts` (framework-agnostic kiểu “adapter”)

```ts
import { randomUUID } from "node:crypto";
import { runWithContext } from "./context";
import { logger } from "./logger";

export type RawReq = {
  method: string;
  path: string;
  headers: Record<string, string | undefined>;
  body?: unknown;
};

export type RawRes = { status: number; body: unknown };

function nowMs() {
  return Date.now();
}

function getRequestId(req: RawReq): string {
  // prefer incoming header x-request-id; else generate
  return req.headers["x-request-id"] ?? randomUUID();
}

function fakeAuthToUserId(req: RawReq): string | undefined {
  // TODO: if authorization exists like "Bearer usr_..." return user id
  const a = req.headers["authorization"];
  if (!a) return undefined;
  const m = a.match(/^Bearer\s+(.+)$/);
  if (!m) return undefined;
  // demo: token is userId
  return m[1];
}

export async function withHttpLogging(
  req: RawReq,
  handler: (req: RawReq) => Promise<RawRes> | RawRes
): Promise<RawRes> {
  const start = nowMs();
  const request_id = getRequestId(req);
  const user_id = fakeAuthToUserId(req);

  return await runWithContext({ request_id, user_id, trace_id: request_id }, async () => {
    try {
      const res = await handler(req);
      const latency_ms = nowMs() - start;

      logger.info(
        {
          event: "http.request.completed",
          method: req.method,
          path: req.path,
          status: res.status,
          latency_ms,
          // optional: attach req meta but be careful; redact handles secrets
          req: { headers: req.headers },
        },
        "request completed"
      );

      return res;
    } catch (e) {
      const latency_ms = nowMs() - start;

      logger.error(
        {
          event: "http.request.failed",
          method: req.method,
          path: req.path,
          status: 500,
          latency_ms,
          error: {
            name: e instanceof Error ? e.name : "UnknownError",
            message: e instanceof Error ? e.message : String(e),
          },
        },
        "request failed"
      );

      throw e;
    }
  });
}
```

### `src/app.ts` (demo handler + deep layer log)

```ts
import { withHttpLogging, type RawReq, type RawRes } from "./http-middleware";
import { logger } from "./logger";

// deep layer function
function createOrder(userId?: string) {
  // NOTE: logger auto attaches request_id/user_id via ALS
  logger.info({ event: "order.created", user_id: userId ?? "anonymous" }, "order created");
  return { orderId: "ord_deadbeefc0de" };
}

async function handler(req: RawReq): Promise<RawRes> {
  if (req.method === "POST" && req.path === "/v1/orders") {
    // TODO: integrate Kata 25 validation if you want; for kata this stub is enough
    const out = createOrder(req.headers["authorization"]?.replace(/^Bearer\s+/, ""));
    return { status: 201, body: out };
  }
  return { status: 404, body: { ok: false } };
}

// export a function you can test
export async function handle(req: RawReq): Promise<RawRes> {
  return withHttpLogging(req, handler);
}
```

---

## Tests (vitest) — kiểm tra log thật, không nói suông

### Ý tưởng test

* Capture stdout (pino default writes to stdout)
* Parse từng line JSON
* Assert line có `request_id`, có `user_id` khi auth
* Assert không chứa authorization/password

### `test/kata27.test.ts`

```ts
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import { handle } from "../src/app";

function captureStdout() {
  const writes: string[] = [];
  const orig = process.stdout.write.bind(process.stdout);

  // @ts-ignore
  process.stdout.write = (chunk: any) => {
    writes.push(Buffer.isBuffer(chunk) ? chunk.toString("utf8") : String(chunk));
    return true;
  };

  return {
    stop() {
      // @ts-ignore
      process.stdout.write = orig;
      return writes.join("");
    },
  };
}

describe("kata27 - structured logging", () => {
  it("logs request completed with request_id and user_id", async () => {
    const cap = captureStdout();

    await handle({
      method: "POST",
      path: "/v1/orders",
      headers: {
        "x-request-id": "req_abc",
        authorization: "Bearer usr_1a2b3c4d5e6f",
      },
      body: { items: [] },
    });

    const out = cap.stop();
    const lines = out.trim().split("\n").filter(Boolean);
    const parsed = lines.map((l) => JSON.parse(l));

    const completed = parsed.find((x) => x.event === "http.request.completed");
    expect(completed).toBeTruthy();
    expect(completed.request_id).toBe("req_abc");
    expect(completed.user_id).toBe("usr_1a2b3c4d5e6f");

    const domain = parsed.find((x) => x.event === "order.created");
    expect(domain).toBeTruthy();
    expect(domain.request_id).toBe("req_abc"); // proves ALS propagation
  });

  it("redacts authorization from logs", async () => {
    const cap = captureStdout();

    await handle({
      method: "POST",
      path: "/v1/orders",
      headers: { authorization: "Bearer supersecret", "x-request-id": "req_redact" },
    });

    const out = cap.stop();
    expect(out).not.toContain("supersecret");
    expect(out).not.toContain("authorization");
  });
});
```

---

## Checks (Definition of Done)

Bạn pass kata này khi:

1. Log output là JSON lines (mỗi line parse được JSON).
2. Log `http.request.completed` luôn có:

* `request_id`
* `method`, `path`, `status`, `latency_ms`

3. Domain log `order.created` (deep layer) **vẫn tự có request_id/user_id** ⇒ chứng minh ALS hoạt động.
4. Không leak secrets:

* stdout không chứa `authorization` value, `password`, `token`, `cookie`.

---

## Stretch (Senior+ “đụng chỗ đau”)

1. **Sampling**:

* chỉ log `http.request.completed` cho 2xx theo sampling (vd 10%)
* nhưng log **100%** cho `status >= 400` hoặc `latency_ms > threshold`

2. **Error serialization chuẩn**:

* log `{ err: serializeError(e) }` thay vì chỉ message

3. **Request size / response size**:

* thêm `req_bytes`, `res_bytes`

4. **Child logger per module**:

* `logger.info({ component: "orders" ... })` hoặc `baseLogger.child({ component: "orders" })`

5. **Correlate với Problem+JSON** (Kata 26):

* response error luôn include `traceId`
* log line cũng include `traceId` ⇒ 1 click trace