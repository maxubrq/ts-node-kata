# Bonus Kata 08.1 — Immutable Context (AsyncLocalStorage)

## Goal

1. Tạo `RequestContext` immutable:

* type-level: `DeepReadonly<RequestContext>`
* runtime: `freezeDeep(ctx)`

2. Dùng `AsyncLocalStorage` để:

* `withContext(ctx, fn)` chạy fn trong scope có context
* `getContext()` lấy context ở bất kỳ layer nào (logger, repo, service)

3. Logger tự động đính `requestId` vào log mà không cần truyền tay
4. Bonus: `deadlineMs` + helper `timeLeft()` (nối với Kata 04.1 timeout)

---

## Constraints

* ❌ Không truyền context bằng parameter xuyên mọi function (đó là anti-goal)
* ❌ Không mutate context sau khi set
* ✅ `getContext()` phải typed, không `any`
* ✅ Nếu gọi `getContext()` ngoài scope → throw rõ ràng (programmer error)

---

## Deliverables

File `kata08_1.ts` gồm:

1. `DeepReadonly` + `freezeDeep` (có thể reuse từ Kata 08)
2. `RequestContext` type
3. `ContextStore` module (withContext/getContext)
4. `log()` tự gắn context
5. Demo chạy được (console)

---

## RequestContext spec

```ts
type RequestContext = {
  requestId: string;
  userId?: string;
  tenantId?: string;
  deadlineMs?: number; // epoch ms (optional)
  tags?: string[];     // debugging tags (optional)
};
```

---

## Starter skeleton (điền TODO)

```ts
/* kata08_1.ts */

import { AsyncLocalStorage } from "node:async_hooks";

// ---------- DeepReadonly + freezeDeep (reuse from kata08) ----------
export type DeepReadonly<T> =
  T extends (...args: any[]) => any ? T :
  T extends ReadonlyArray<infer U> ? ReadonlyArray<DeepReadonly<U>> :
  T extends Array<infer U> ? ReadonlyArray<DeepReadonly<U>> :
  T extends Map<infer K, infer V> ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>> :
  T extends Set<infer U> ? ReadonlySet<DeepReadonly<U>> :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
  T;

// NOTE: Above contains `any` in function type branch only; replace it if you want ultra-pure.
// Acceptable for kata? If you want strict: use unknown[] instead of any[] (task below).

// TODO: make this branch any-free
type AnyFreeFn = (...args: unknown[]) => unknown;

export type DeepReadonly2<T> =
  T extends AnyFreeFn ? T :
  T extends ReadonlyArray<infer U> ? ReadonlyArray<DeepReadonly2<U>> :
  T extends Array<infer U> ? ReadonlyArray<DeepReadonly2<U>> :
  T extends Map<infer K, infer V> ? ReadonlyMap<DeepReadonly2<K>, DeepReadonly2<V>> :
  T extends Set<infer U> ? ReadonlySet<DeepReadonly2<U>> :
  T extends object ? { readonly [K in keyof T]: DeepReadonly2<T[K]> } :
  T;

export function freezeDeep<T>(value: T): DeepReadonly2<T> {
  // TODO: deep freeze objects/arrays
  // Tip: if value is object, iterate own props and recurse before Object.freeze
  return value as any;
}

// ---------- Context ----------
export type RequestContext = {
  requestId: string;
  userId?: string;
  tenantId?: string;
  deadlineMs?: number;
  tags?: string[];
};

type Ctx = DeepReadonly2<RequestContext>;

const als = new AsyncLocalStorage<Ctx>();

export function withContext<T>(ctx: RequestContext, fn: () => T): T {
  // TODO:
  // - freezeDeep(ctx)
  // - als.run(frozen, fn)
  return fn();
}

export function getContext(): Ctx {
  const ctx = als.getStore();
  if (!ctx) throw new Error("No RequestContext in AsyncLocalStorage scope");
  return ctx;
}

// ---------- Helpers ----------
export function timeLeftMs(now: number = Date.now()): number | undefined {
  const { deadlineMs } = getContext();
  if (deadlineMs === undefined) return undefined;
  return Math.max(0, deadlineMs - now);
}

// ---------- Logger ----------
type LogLevel = "info" | "warn" | "error";

export function log(level: LogLevel, msg: string, extra: Record<string, unknown> = {}) {
  const ctx = getContext();
  // don't log PII beyond ids
  const line = {
    ts: Date.now(),
    level,
    msg,
    requestId: ctx.requestId,
    userId: ctx.userId,
    tenantId: ctx.tenantId,
    timeLeftMs: ctx.deadlineMs ? timeLeftMs() : undefined,
    ...extra,
  };
  console.log(JSON.stringify(line));
}

// ---------- Demo "layers" ----------
async function fakeDbCall() {
  await new Promise((r) => setTimeout(r, 20));
  log("info", "db.query.ok");
}

async function service() {
  log("info", "service.start");
  await fakeDbCall();

  // TODO: prove immutability (compile-time + runtime)
  // @ts-expect-error
  getContext().requestId = "hacked";

  log("info", "service.end");
}

async function handler() {
  const ctx: RequestContext = {
    requestId: "req_123",
    userId: "usr_1a2b3c4d5e6f",
    tenantId: "t1",
    deadlineMs: Date.now() + 50,
    tags: ["demo"],
  };

  return withContext(ctx, () => service());
}

// ---------- Run ----------
handler()
  .then(() => console.log("done"))
  .catch((e) => console.error("unexpected:", e));
```

---

## Required behaviors

1. Trong `service()` gọi `log()` vẫn có `requestId` mà không truyền parameter
2. `getContext().requestId = ...` bị chặn compile-time
3. Nếu bạn cố tình cast để mutate, runtime nên throw nếu `freezeDeep` chuẩn
4. `getContext()` ngoài scope (gọi trực tiếp ở top-level) phải throw rõ ràng

---

## Definition of Done

* Không dùng `any` trong `DeepReadonly2` (đã chuyển sang `unknown[]`)
* `freezeDeep` deep-freeze được object/array lồng nhau
* Async boundary (await/setTimeout) vẫn giữ context
* Logs là JSON, có requestId

---

## Stretch (rất production)

1. `withContextAsync(ctx, fn: () => Promise<T>)` overload để dùng tiện hơn
2. `runWithDeadline(msFromNow, fn)` tự set deadline & propagate
3. `withTag("payment", fn)` push tag **không mutate** (trả context mới trong nested scope)
4. Integrate với `withTimeout` của Kata 04.1: timeout = min(userTimeout, timeLeft)