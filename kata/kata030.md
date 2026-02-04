# Kata 30 — Multi-tenant Guards: Tenant isolation bằng middleware

## Goal

1. Implement tenant isolation chuẩn:

* Mọi request phải có **tenant context** (từ auth token / header / subdomain)
* Mọi DB access phải **bị ép** đi kèm `tenant_id`
* Không có chuyện “quên where tenant_id = ?”

2. Thiết kế middleware + context propagation (ALS) để:

* handler/service/repo ở deep layer vẫn biết tenant
* log & error có `tenant_id` để correlate (nối Kata 27)

3. Có test chứng minh:

* request thiếu tenant ⇒ 401/400
* user thuộc tenant A không đọc được data tenant B
* repo/query không thể compile nếu thiếu tenantId (stretch) hoặc runtime guard bắt được

---

## Constraints

* ✅ Tenant ID **không lấy trực tiếp từ body/query** (client input dễ fake).
  Tenant phải đến từ **auth context** (hoặc mapping từ API key).
* ✅ Có middleware `requireTenant()` set `tenant_id` vào context.
* ✅ Có “query guard”: mọi repo function bắt buộc nhận `TenantId`.
* ✅ Có test “attempt cross-tenant access” fail.
* ❌ Không dùng global mutable singleton để giữ tenant (phải ALS hoặc passing explicit).
* ✅ Error response chuẩn Problem+JSON (reuse Kata 26 style nếu bạn muốn).

---

## Context (scenario demo)

Bạn có service `GET /v1/orders/:orderId`

* Auth token decode ra:

  * `user_id`
  * `tenant_id`
* Data store có order thuộc nhiều tenant.

Yêu cầu:

* Nếu user thuộc tenant A gọi order thuộc tenant B ⇒ **404** (recommended) hoặc **403** (tuỳ policy).
  (Production thường trả 404 để tránh enumeration.)

---

## Data model (in-memory fake DB cho kata)

Order:

```ts
type Order = {
  tenantId: TenantId;
  orderId: OrderId;
  userId: UserId;
  total: number;
};
```

---

## Deliverables

`kata-30/`

* `src/types.ts` (Brand + TenantId/UserId/OrderId)
* `src/context.ts` (ALS context: request_id, user_id, tenant_id)
* `src/auth.ts` (fake auth decode → claims)
* `src/middleware.ts` (requireAuth + requireTenant + attachContext)
* `src/repo.ts` (OrderRepo: tenant-scoped queries)
* `src/app.ts` (routes using middleware)
* `test/kata30.test.ts` (isolation tests)

---

## Starter Skeleton (điền TODO)

### `src/types.ts`

```ts
declare const __brand: unique symbol;
export type Brand<T, Name extends string> = T & { readonly [__brand]: Name };

export type TenantId = Brand<string, "TenantId">;
export type UserId = Brand<string, "UserId">;
export type OrderId = Brand<string, "OrderId">;

export function asTenantId(x: string): TenantId { return x as TenantId; }
export function asUserId(x: string): UserId { return x as UserId; }
export function asOrderId(x: string): OrderId { return x as OrderId; }
```

### `src/context.ts`

```ts
import { AsyncLocalStorage } from "node:async_hooks";
import type { TenantId, UserId } from "./types";

export type RequestContext = {
  request_id: string;
  user_id?: UserId;
  tenant_id?: TenantId;
};

const als = new AsyncLocalStorage<RequestContext>();

export function runWithContext<T>(ctx: RequestContext, fn: () => T): T {
  return als.run(ctx, fn);
}
export function getContext(): RequestContext | undefined {
  return als.getStore();
}

// hard requirement accessor (guard rails)
export function requireTenantId(): TenantId {
  const t = als.getStore()?.tenant_id;
  if (!t) throw new Error("TENANT_CONTEXT_MISSING");
  return t;
}

export function requireUserId(): UserId {
  const u = als.getStore()?.user_id;
  if (!u) throw new Error("USER_CONTEXT_MISSING");
  return u;
}
```

### `src/auth.ts`

```ts
import type { TenantId, UserId } from "./types";
import { asTenantId, asUserId } from "./types";

export type Claims = {
  user_id: UserId;
  tenant_id: TenantId;
};

// Fake auth: Authorization: Bearer tenantA|user123
export function decodeAuthHeader(auth?: string): Claims | null {
  if (!auth) return null;
  const m = auth.match(/^Bearer\s+(.+)$/);
  if (!m) return null;

  const token = m[1];
  const parts = token.split("|");
  if (parts.length !== 2) return null;

  const [tenantRaw, userRaw] = parts;
  if (!tenantRaw || !userRaw) return null;

  return { tenant_id: asTenantId(tenantRaw), user_id: asUserId(userRaw) };
}
```

### `src/middleware.ts`

```ts
import { randomUUID } from "node:crypto";
import type { RawReq, RawRes, Handler } from "./runtime";
import { runWithContext } from "./context";
import { decodeAuthHeader } from "./auth";

export function withRequestContext(handler: Handler): Handler {
  return async (req) => {
    const request_id = req.headers["x-request-id"] ?? randomUUID();
    return runWithContext({ request_id }, () => handler(req));
  };
}

export function requireAuth(handler: Handler): Handler {
  return async (req) => {
    const claims = decodeAuthHeader(req.headers["authorization"]);
    if (!claims) return { status: 401, body: { ok: false, error: "unauthorized" } };

    // merge into ALS context
    return runWithContext(
      { ...(req.context ?? {}), request_id: req.context?.request_id ?? "missing", user_id: claims.user_id, tenant_id: claims.tenant_id },
      () => handler(req)
    );
  };
}
```

> Đoạn trên có “hack”: `req.context` chưa tồn tại. Ta sẽ làm runtime wrapper gọn hơn ngay dưới đây.

### `src/runtime.ts` (mini framework để chain middleware clean)

```ts
export type RawReq = {
  method: string;
  path: string;
  headers: Record<string, string | undefined>;
  body?: unknown;
};

export type RawRes = { status: number; body: unknown };

export type Handler = (req: RawReq) => Promise<RawRes> | RawRes;

export function compose(...mws: Array<(h: Handler) => Handler>) {
  return (h: Handler) => mws.reduceRight((acc, mw) => mw(acc), h);
}
```

### `src/middleware.ts` (clean version, không cần req.context)

```ts
import { randomUUID } from "node:crypto";
import type { Handler } from "./runtime";
import { runWithContext, getContext } from "./context";
import { decodeAuthHeader } from "./auth";

export function withRequestContext(): (h: Handler) => Handler {
  return (h) => async (req) => {
    const request_id = req.headers["x-request-id"] ?? randomUUID();
    return runWithContext({ request_id }, () => h(req));
  };
}

export function withAuthContext(): (h: Handler) => Handler {
  return (h) => async (req) => {
    const claims = decodeAuthHeader(req.headers["authorization"]);
    if (!claims) return { status: 401, body: { ok: false, error: "unauthorized" } };

    const base = getContext(); // should contain request_id
    return runWithContext(
      { request_id: base?.request_id ?? "missing", user_id: claims.user_id, tenant_id: claims.tenant_id },
      () => h(req)
    );
  };
}
```

### `src/repo.ts`

```ts
import type { Order, OrderRepo } from "./repo.types";
import type { TenantId, OrderId } from "./types";

export type Order = {
  tenantId: TenantId;
  orderId: OrderId;
  userId: any;
  total: number;
};

export interface OrderRepo {
  getById(tenantId: TenantId, orderId: OrderId): Promise<Order | null>;
  list(tenantId: TenantId): Promise<Order[]>;
}

export function makeInMemoryOrderRepo(seed: Order[]): OrderRepo {
  return {
    async getById(tenantId, orderId) {
      // ✅ mandatory tenant filter
      return seed.find((o) => o.tenantId === tenantId && o.orderId === orderId) ?? null;
    },
    async list(tenantId) {
      return seed.filter((o) => o.tenantId === tenantId);
    },
  };
}
```

### `src/app.ts`

```ts
import { compose, type RawReq } from "./runtime";
import { withRequestContext, withAuthContext } from "./middleware";
import { requireTenantId } from "./context";
import { asOrderId, asTenantId, asUserId } from "./types";
import { makeInMemoryOrderRepo } from "./repo";

const repo = makeInMemoryOrderRepo([
  { tenantId: asTenantId("tenantA"), orderId: asOrderId("ord_a"), userId: asUserId("user1"), total: 100 },
  { tenantId: asTenantId("tenantB"), orderId: asOrderId("ord_b"), userId: asUserId("user9"), total: 999 },
]);

async function getOrderHandler(req: RawReq) {
  // parse orderId from path: /v1/orders/:id
  const m = req.path.match(/^\/v1\/orders\/([^/]+)$/);
  if (!m) return { status: 404, body: { ok: false } };

  const orderId = asOrderId(m[1]);
  const tenantId = requireTenantId();

  const order = await repo.getById(tenantId, orderId);

  // recommended: 404 on cross-tenant access
  if (!order) return { status: 404, body: { ok: false, message: "Not Found" } };

  return { status: 200, body: { orderId: order.orderId, total: order.total } };
}

export const handle = compose(
  withRequestContext(),
  withAuthContext(),
)(getOrderHandler);
```

---

## Tests (vitest) — isolation or it didn’t happen

### `test/kata30.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { handle } from "../src/app";

describe("kata30 - multi tenant guards", () => {
  it("rejects when no auth (no tenant context)", async () => {
    const res = await handle({
      method: "GET",
      path: "/v1/orders/ord_a",
      headers: {},
    });

    expect(res.status).toBe(401);
  });

  it("tenantA can read its own order", async () => {
    const res = await handle({
      method: "GET",
      path: "/v1/orders/ord_a",
      headers: { authorization: "Bearer tenantA|user1" },
    });

    expect(res.status).toBe(200);
    expect((res.body as any).orderId).toBe("ord_a");
  });

  it("tenantA cannot read tenantB order (should be 404)", async () => {
    const res = await handle({
      method: "GET",
      path: "/v1/orders/ord_b",
      headers: { authorization: "Bearer tenantA|user1" },
    });

    expect(res.status).toBe(404);
  });
});
```

---

## Checks (Definition of Done)

Bạn pass kata này khi:

1. Tenant context **được set bởi middleware**, không lấy từ request body/query.
2. Repo API **bắt buộc** `tenantId` ở mọi method.
3. Cross-tenant access bị chặn (404 hoặc 403, nhưng phải nhất quán).
4. Có test chứng minh 3 case: no auth, same-tenant ok, cross-tenant fail.

---

## Stretch (Senior+ “đúng chất production”)

1. **Compile-time guard mạnh hơn**: không cho gọi repo nếu thiếu tenant:

   * tạo `TenantScopedRepo` chỉ tạo được từ `requireTenantId()`
   * ví dụ:

     ```ts
     const scoped = repo.scope(requireTenantId());
     scoped.getById(orderId);
     ```

   → giảm nguy cơ dev quên truyền tenantId.

2. **Row-level security mindset**:

   * nếu dùng Postgres: enforce `tenant_id` bằng RLS policy (mô tả trong README)
   * app vẫn giữ guard để defense-in-depth.

3. **Tenant from API key**:

   * header `x-api-key` → lookup tenant in store
   * rotate keys, audit logs.

4. **Observability**:

   * structured logging luôn include `tenant_id` (nối Kata 27)
   * metrics per tenant: `requests_total{tenant_id=...}` (cẩn thận cardinality)

5. **Prevent tenant spoofing**:

   * nếu có `x-tenant-id` header: chỉ dùng cho internal trusted gateway, còn public ignore.