# Kata 29 — API Versioning Strategy: v1/v2 + deprecation plan

## Goal

1. Thiết kế và implement **một chiến lược versioning** rõ ràng cho HTTP API:

* hỗ trợ **v1** và **v2** chạy song song
* tránh “đẻ version vô tội vạ”
* consumer upgrade có đường đi

2. Có **deprecation plan** chuẩn:

* response headers (Deprecation / Sunset / Link)
* endpoint deprecation registry
* observability: ai đang dùng v1, bao nhiêu, route nào

3. Có **compatibility strategy**:

* backward-compatible changes được phép
* breaking changes phải đi qua v2
* migration tooling (shim/adapter) để giảm chi phí vận hành

---

## Constraints

* ✅ v1/v2 chạy song song trong cùng app (hoặc cùng gateway) và có routing rõ.
* ✅ Có **policy** văn bản ngắn (1 trang) mô tả:

  * khi nào bump version
  * timeline deprecate
  * communication plan
* ✅ Có **headers deprecation** tự động khi gọi endpoint deprecated.
* ✅ Có test chứng minh:

  * v1 và v2 trả khác nhau theo spec
  * deprecation headers có mặt
  * “unknown version” xử lý đúng
* ❌ Không chấp nhận “copy/paste controller v1 sang v2” mà không có plan.

---

## Context (bài toán demo đủ thực tế)

Bạn có endpoint cũ:

### v1: `POST /v1/orders`

Request v1:

```json
{
  "userId": "usr_1a2b3c4d5e6f",
  "items": [{ "sku": "SKU_ABC123", "qty": 2 }],
  "couponCode": "TET2026"
}
```

Response v1 (201):

```json
{ "orderId": "ord_deadbeefc0de" }
```

Bạn muốn thay đổi contract theo business:

* `couponCode` đổi tên thành `promotionCode`
* thêm `currency` và `total` vào response
* v2 chuẩn hoá `idempotencyKey` (header bắt buộc cho create order)

### v2: `POST /v2/orders`

Request v2:

```json
{
  "userId": "usr_1a2b3c4d5e6f",
  "items": [{ "sku": "SKU_ABC123", "qty": 2 }],
  "promotionCode": "TET2026"
}
```

Response v2 (201):

```json
{ "orderId": "ord_deadbeefc0de", "currency": "VND", "total": 123000 }
```

---

## Versioning options (chọn 1 và implement)

Bạn phải implement **ít nhất 1**, nhưng mình khuyên làm **path-based** để kata rõ ràng.

### Option A — Path-based (khuyên dùng cho kata)

* `/v1/...`, `/v2/...`
* Ưu: đơn giản, cache/CDN friendly
* Nhược: nhiều routes, nhưng quản được

### Option B — Header-based

* `Accept: application/vnd.acme.orders+json;version=2`
* Ưu: URL sạch
* Nhược: tooling/debug phức tạp hơn

> Kata này dùng Option A.

---

## Deliverables

`kata-29/`

* `src/policy.md` (versioning + deprecation policy 1 trang)
* `src/v1/orders.ts` (schema + handler)
* `src/v2/orders.ts` (schema + handler)
* `src/versioning.ts` (router + deprecation registry + headers)
* `src/app.ts` (wire routing)
* `test/kata29.test.ts` (behavior + headers)

---

## Deprecation spec (đúng kiểu production)

Nếu endpoint bị deprecate, response phải có:

* `Deprecation: true` (hoặc date)
* `Sunset: <HTTP-date>` (ngày tắt hẳn)
* `Link: <...>; rel="deprecation"` (link policy/migration guide)
* (optional) `Warning: 299 - "Deprecated API, migrate to /v2"` (cẩn thận spam)

> Bạn không cần hoàn hảo 100% RFC trong kata, nhưng phải **nhất quán**.

---

## Starter Skeleton (điền TODO)

### `src/versioning.ts`

```ts
export type RawReq = {
  method: string;
  path: string;
  headers: Record<string, string | undefined>;
  body?: unknown;
};

export type RawRes = {
  status: number;
  headers?: Record<string, string>;
  body: unknown;
};

type Handler = (req: RawReq) => Promise<RawRes> | RawRes;

type DeprecationInfo = {
  sunset: string; // HTTP-date, e.g. "Wed, 01 Oct 2026 00:00:00 GMT"
  link: string;   // URL to migration guide
  message?: string;
};

// registry: routeKey -> deprecation info
const deprecatedRoutes: Record<string, DeprecationInfo> = {
  // TODO: mark v1 create order as deprecated
  // "POST /v1/orders": { sunset: "...", link: "...", message: "Use /v2/orders" },
};

function routeKey(req: RawReq) {
  return `${req.method.toUpperCase()} ${req.path}`;
}

export function applyDeprecationHeaders(req: RawReq, res: RawRes): RawRes {
  const info = deprecatedRoutes[routeKey(req)];
  if (!info) return res;

  const headers = { ...(res.headers ?? {}) };

  headers["deprecation"] = "true";
  headers["sunset"] = info.sunset;
  headers["link"] = `<${info.link}>; rel="deprecation"`;

  if (info.message) headers["warning"] = `299 - "${info.message}"`;

  return { ...res, headers };
}

export function makeRouter(routes: Record<string, Handler>): Handler {
  return async (req) => {
    const key = routeKey(req);
    const h = routes[key];
    if (!h) return { status: 404, body: { ok: false, message: "Not Found" } };

    const res = await h(req);
    return applyDeprecationHeaders(req, res);
  };
}
```

### `src/v1/orders.ts`

```ts
import { z } from "zod";

const USER_ID_RE = /^usr_[0-9a-f]{12}$/;
const SKU_RE = /^[A-Z0-9_]{3,32}$/;

export const CreateOrderV1Schema = z.object({
  userId: z.string().regex(USER_ID_RE),
  items: z.array(z.object({
    sku: z.string().regex(SKU_RE),
    qty: z.number().int().min(1).max(999),
  }).strict()).min(1).max(50),
  couponCode: z.string().regex(/^[A-Z0-9]{4,16}$/).optional(),
}).strict();

export type CreateOrderV1 = z.infer<typeof CreateOrderV1Schema>;

export async function createOrderV1(reqBody: unknown) {
  const parsed = CreateOrderV1Schema.safeParse(reqBody);
  if (!parsed.success) {
    return { status: 400, body: { ok: false, error: "validation" } };
  }

  // TODO: stub create
  return { status: 201, body: { orderId: "ord_deadbeefc0de" } };
}
```

### `src/v2/orders.ts`

```ts
import { z } from "zod";

const USER_ID_RE = /^usr_[0-9a-f]{12}$/;
const SKU_RE = /^[A-Z0-9_]{3,32}$/;

export const CreateOrderV2Schema = z.object({
  userId: z.string().regex(USER_ID_RE),
  items: z.array(z.object({
    sku: z.string().regex(SKU_RE),
    qty: z.number().int().min(1).max(999),
  }).strict()).min(1).max(50),
  promotionCode: z.string().regex(/^[A-Z0-9]{4,16}$/).optional(),
}).strict();

export type CreateOrderV2 = z.infer<typeof CreateOrderV2Schema>;

function requireIdempotencyKey(headers: Record<string, string | undefined>) {
  const k = headers["idempotency-key"];
  return typeof k === "string" && k.length >= 16 && k.length <= 128;
}

export async function createOrderV2(req: { headers: Record<string, string | undefined>; body: unknown }) {
  // new contract requirement
  if (!requireIdempotencyKey(req.headers)) {
    return { status: 400, body: { ok: false, error: "missing idempotency-key" } };
  }

  const parsed = CreateOrderV2Schema.safeParse(req.body);
  if (!parsed.success) {
    return { status: 400, body: { ok: false, error: "validation" } };
  }

  // TODO: stub create
  return {
    status: 201,
    body: { orderId: "ord_deadbeefc0de", currency: "VND", total: 123000 },
  };
}
```

### `src/app.ts`

```ts
import { makeRouter, type RawReq } from "./versioning";
import { createOrderV1 } from "./v1/orders";
import { createOrderV2 } from "./v2/orders";

const routes = {
  "POST /v1/orders": async (req: RawReq) => createOrderV1(req.body),
  "POST /v2/orders": async (req: RawReq) => createOrderV2({ headers: req.headers, body: req.body }),
};

export const handle = makeRouter(routes);
```

---

## Test suite (vitest)

### `test/kata29.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { handle } from "../src/app";

describe("kata29 - api versioning", () => {
  it("v1 create order works and returns legacy response shape", async () => {
    const res = await handle({
      method: "POST",
      path: "/v1/orders",
      headers: { "x-request-id": "req_1" },
      body: { userId: "usr_1a2b3c4d5e6f", items: [{ sku: "SKU_ABC123", qty: 2 }] },
    });

    expect(res.status).toBe(201);
    expect((res.body as any).orderId).toBeTruthy();
    expect((res.body as any).total).toBeUndefined(); // v1 has no total
  });

  it("v2 create order requires idempotency-key and returns new response shape", async () => {
    const res = await handle({
      method: "POST",
      path: "/v2/orders",
      headers: { "idempotency-key": "idem_1234567890abcdef" },
      body: { userId: "usr_1a2b3c4d5e6f", items: [{ sku: "SKU_ABC123", qty: 2 }], promotionCode: "TET2026" },
    });

    expect(res.status).toBe(201);
    expect((res.body as any).total).toBeTypeOf("number");
    expect((res.body as any).currency).toBe("VND");
  });

  it("v2 rejects missing idempotency-key", async () => {
    const res = await handle({
      method: "POST",
      path: "/v2/orders",
      headers: {},
      body: { userId: "usr_1a2b3c4d5e6f", items: [{ sku: "SKU_ABC123", qty: 2 }] },
    });

    expect(res.status).toBe(400);
  });

  it("deprecated v1 endpoints include deprecation headers", async () => {
    const res = await handle({
      method: "POST",
      path: "/v1/orders",
      headers: {},
      body: { userId: "usr_1a2b3c4d5e6f", items: [{ sku: "SKU_ABC123", qty: 2 }] },
    });

    // TODO: once you mark it deprecated in registry, these should pass
    expect(res.headers?.deprecation).toBe("true");
    expect(res.headers?.sunset).toBeTruthy();
    expect(res.headers?.link).toContain('rel="deprecation"');
  });
});
```

> Lưu ý: test “deprecated headers” sẽ fail cho tới khi bạn điền `deprecatedRoutes` trong `versioning.ts`. Đó là chủ ý: kata ép bạn làm deprecation đúng, không nói mồm.

---

## Policy 1 trang (Deliverable bắt buộc) — `src/policy.md`

Copy/paste template này và điền:

```md
# API Versioning & Deprecation Policy

## Versioning rule
- Backward-compatible changes:
  - Add optional fields
  - Add new endpoints
  - Relax validation
- Breaking changes (require new major version):
  - Rename/remove fields
  - Change semantics
  - Tighten validation in a way that breaks existing clients
  - Change error shapes

## Supported versions
- Current: v2
- Previous: v1 (deprecated)

## Deprecation timeline
- Deprecation announced: <DATE>
- Sunset date: <DATE> (v1 disabled)
- Minimum support window: 90 days

## Deprecation signals
- Responses from deprecated endpoints include:
  - Deprecation: true
  - Sunset: <HTTP-date>
  - Link: <migration guide>; rel="deprecation"
  - Warning: 299 "<message>" (optional)

## Migration guidance
- v1 couponCode => v2 promotionCode
- v2 requires Idempotency-Key header
- v2 response includes total/currency

## Telemetry
- Track usage by:
  - version (v1 vs v2)
  - route
  - client (user-agent / api-key / client-id if available)
- Weekly report until sunset.
```

---

## Definition of Done (Checks)

Bạn pass kata này khi:

1. v1 và v2 cùng chạy, contract khác nhau rõ ràng, có tests.
2. Có `policy.md` mô tả rule + timeline + signals.
3. Endpoint v1 bị deprecate có headers đầy đủ (`Deprecation`, `Sunset`, `Link`, optional `Warning`).
4. Có chiến lược migration rõ (mapping field + new requirement).
5. Không drift: code structure cho phép maintain song song mà không “copy/paste bừa”.

---

## Stretch (Senior+)

1. **Shim adapter**: implement `/v1/orders` bằng cách:

* parse v1 → transform sang v2 request
* gọi v2 handler nội bộ
* transform v2 response → v1 response
  => giảm duplicate logic, tăng consistency.

2. **Client discovery**: thêm header `X-Client-Id` và thống kê ai chưa migrate.

3. **Kill switch**:

* feature flag để tắt v1 theo % / theo client id trước ngày Sunset.

4. **OpenAPI multi-version** (Kata 28 nối tiếp):

* generate `openapi.v1.json` và `openapi.v2.json`
* route `/openapi/v1.json`, `/openapi/v2.json`