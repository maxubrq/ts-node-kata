# Kata 12 — Async Context Trace (AsyncLocalStorage)

## Goal

1. Dùng `AsyncLocalStorage` để propagate `RequestContext` xuyên qua:

* `await`
* `setTimeout`
* nested function calls

2. Xây “async trace” dạng breadcrumb:

* `trace.push("service.createOrder")`
* `trace.push("db.query users")`
* khi error xảy ra → log ra trace này

3. Viết wrapper:

* `withContext(ctx, fn)`
* `withSpan(name, fn)` (push/pop trace)

4. Error boundary (Kata 11 style) log:

* requestId
* trace (list)
* original error stack (nếu là programmer error)

---

## Constraints

* ❌ Không truyền trace thủ công qua params
* ❌ Không mutate context bừa bãi (context nên immutable; trace update phải theo “copy on write” hoặc controlled)
* ✅ Không `any`
* ✅ `withSpan` phải auto-pop dù fn throw (try/finally)

---

## Deliverables

File `kata12.ts` gồm:

1. `RequestContext` + `TraceContext` typed
2. `withContext`, `getContext`
3. `withSpan`
4. `simulateRequest()` tạo lỗi ở deep async và chứng minh log có trace

---

## Context spec

```ts
type TraceFrame = { name: string; at: number };

type RequestContext = {
  requestId: string;
  trace: readonly TraceFrame[];
};
```

Rule:

* `trace` là readonly array
* mỗi `withSpan(name, fn)` tạo **context mới** với trace appended (immutability)
* `getContext()` luôn trả context hiện tại

---

## Starter skeleton (điền TODO)

```ts
/* kata12.ts */

import { AsyncLocalStorage } from "node:async_hooks";

// ---------- Types ----------
type TraceFrame = { name: string; at: number };

type RequestContext = {
  requestId: string;
  trace: readonly TraceFrame[];
};

type Ctx = Readonly<RequestContext>;

const als = new AsyncLocalStorage<Ctx>();

// ---------- Context helpers ----------
function withContext<T>(ctx: { requestId: string }, fn: () => T): T {
  const base: Ctx = Object.freeze({ requestId: ctx.requestId, trace: Object.freeze([] as TraceFrame[]) });
  // TODO: als.run(base, fn)
  return fn();
}

function getContext(): Ctx {
  const ctx = als.getStore();
  if (!ctx) throw new Error("No context in scope");
  return ctx;
}

// ---------- Span helper ----------
async function withSpan<T>(name: string, fn: () => Promise<T>): Promise<T> {
  const ctx = getContext();
  const nextTrace = Object.freeze([...ctx.trace, { name, at: Date.now() }]);
  const next: Ctx = Object.freeze({ requestId: ctx.requestId, trace: nextTrace });

  // TODO: run next context inside fn, ensure no leak and proper nesting
  // Tip: return await als.run(next, fn)
  return fn();
}

// ---------- Logger ----------
function log(level: "info" | "warn" | "error", msg: string, extra: Record<string, unknown> = {}) {
  const ctx = getContext();
  console.log(
    JSON.stringify({
      ts: Date.now(),
      level,
      msg,
      requestId: ctx.requestId,
      trace: ctx.trace.map((f) => f.name),
      ...extra,
    })
  );
}

// ---------- Simulated layers ----------
async function dbQuery() {
  await new Promise((r) => setTimeout(r, 10));
  log("info", "db.query.ok");
}

async function paymentCall() {
  await new Promise((r) => setTimeout(r, 10));
  // simulate bug
  throw new Error("Gateway response parsing bug");
}

async function serviceCreateOrder() {
  return withSpan("service.createOrder", async () => {
    await withSpan("db.loadUser", dbQuery);
    await withSpan("payment.charge", paymentCall);
    return { ok: true };
  });
}

async function handler() {
  return withSpan("http.POST /orders", async () => {
    return await serviceCreateOrder();
  });
}

// ---------- Boundary ----------
async function runOne() {
  return withContext({ requestId: "req_123" }, async () => {
    try {
      const res = await handler();
      log("info", "request.ok", { res });
    } catch (e) {
      log("error", "request.failed", {
        errorName: e instanceof Error ? e.name : "Unknown",
        message: e instanceof Error ? e.message : String(e),
        stack: e instanceof Error ? e.stack : undefined,
      });
    }
  });
}

runOne().catch((e) => console.error("UNEXPECTED:", e));
```

---

## Required behaviors

Khi chạy, log error phải có:

* `requestId: "req_123"`
* `trace` chứa đúng thứ tự:

  1. `"http.POST /orders"`
  2. `"service.createOrder"`
  3. `"db.loadUser"`
  4. `"payment.charge"`
* stack vẫn có (programmer error), nhưng trace giúp bạn hiểu “async call path”

---

## Definition of Done

* `withSpan` dùng `AsyncLocalStorage.run` để set context cho scope
* `trace` immutable (readonly + frozen)
* nesting đúng (span A chứa span B, trace append đúng)
* không `any`

---

## Stretch (rất production)

1. **Span duration**: mỗi frame có `endAt` hoặc log duration khi span kết thúc (try/finally)
2. **Error annotation**: `wrapError(e)` attach `requestId` + trace snapshot vào error object (không mutate context)
3. **Sampling**: chỉ giữ trace cho slow/error requests
4. **Integrate with Kata 11**: boundary tự gọi `toProblemDetails`, logs kèm trace