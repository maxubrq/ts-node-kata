# Kata 25 — Request Validation: schema → typed request body

## Goal

1. Thiết kế một pipeline chuẩn production:

* `req.body` / `req.query` / `req.params` / `req.headers` (runtime) ⇒ **validate + parse** ⇒ `TypedRequest`
* Handler chỉ nhận **typed input**, không còn `any`/`unknown` rải rác.

2. Chuẩn hóa lỗi validate theo **Problem+JSON** (khớp Kata 26, nhưng ở đây bạn implement tối thiểu).

3. Có **test** cho cả:

* valid request (pass)
* invalid request (fail có error shape đúng)
* edge cases (coerce, missing, extra keys, etc.)

---

## Context

Bạn đang làm API `POST /v1/orders`.

* Body gửi lên là JSON “bẩn” từ user/client.
* Bạn cần parse thành request type chuẩn, đảm bảo:

  * format hợp lệ
  * default/coerce đúng
  * không lọt field thừa (anti footgun)
  * error trả về ổn định để frontend/consumer xử lý.

---

## Constraints

* ✅ Must: **Schema-first validation** (không viết if-else thủ công cho từng field).
* ✅ Must: **Handler không được đọc `req.body` trực tiếp** (chỉ dùng typed output từ validator).
* ✅ Must: tách rõ 2 pha:

  * **validate request** (boundary)
  * **business logic** (domain/service)
* ❌ Không dùng `any`
* ❌ Không “cast bừa” kiểu `as CreateOrderRequest`
* ✅ Có unit test
* ✅ Có cơ chế **strip/forbid unknown keys** (chọn 1 và test rõ ràng)

> Gợi ý thư viện: `zod` (đúng như plan). Nếu bạn thích Ajv cũng được, nhưng dưới đây mình viết theo Zod để “đi thẳng”.

---

## Spec API (đủ đau để đáng luyện)

### Endpoint

`POST /v1/orders`

### Request body (runtime)

```json
{
  "userId": "usr_1a2b3c4d5e6f",
  "items": [
    { "sku": "SKU_ABC123", "qty": 2 }
  ],
  "couponCode": "TET2026",
  "note": "deliver after 6pm",
  "metadata": { "source": "ios" }
}
```

### Validation rules

**userId**

* string
* format `usr_` + 12 hex lowercase (dùng branded type từ Kata 03 nếu muốn)
* required

**items**

* array length: 1..50
* mỗi item:

  * `sku`: uppercase letters/digits/underscore, 3..32 ký tự
  * `qty`: integer 1..999
* required

**couponCode**

* optional
* nếu có: uppercase/digits, 4..16

**note**

* optional
* nếu có: string trimmed, 0..200

**metadata**

* optional
* object JSON “phẳng” (key string, value string|number|boolean), max 20 keys

**Unknown keys**

* chọn **forbid** (khuyên dùng cho request boundary): nếu có key lạ ⇒ 400

---

## Deliverables

Tạo package `kata-25/` với:

1. `src/types.ts`

   * `Brand` + `UserId` (nếu bạn reuse từ Kata 03 thì import)
   * `CreateOrderRequest` (typed output của schema)
2. `src/schema.ts`

   * `CreateOrderSchema` (Zod schema)
   * `parseCreateOrderRequest(input: unknown): Result<CreateOrderRequest, ValidationProblem>`
3. `src/http.ts`

   * `makeJsonHandler(validate, handler)` hoặc middleware tương đương
   * `ProblemJson` shape + `toProblemJson(err)`
4. `src/app.ts`

   * minimal HTTP server (Express/Fastify/native `http` đều được)
   * route `POST /v1/orders`
5. `test/kata25.test.ts`

   * test validate pass/fail + error shape

---

## Output Types (chuẩn hóa)

### Result

```ts
export type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };
```

### Problem+JSON (tối thiểu)

```ts
export type ProblemJson = {
  type: "https://errors.example.com/validation";
  title: "Validation Error";
  status: 400;
  detail: string;
  errors: Array<{
    path: string;        // "items[0].qty"
    message: string;     // "Expected number, received string"
    code?: string;       // optional
  }>;
};
```

---

## Starter Skeleton (điền TODO)

### `src/types.ts`

```ts
declare const __brand: unique symbol;

export type Brand<T, Name extends string> = T & { readonly [__brand]: Name };

export type UserId = Brand<string, "UserId">;

export type CreateOrderRequest = {
  userId: UserId;
  items: Array<{ sku: string; qty: number }>;
  couponCode?: string;
  note?: string;
  metadata?: Record<string, string | number | boolean>;
};

export type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

export type ProblemJson = {
  type: "https://errors.example.com/validation";
  title: "Validation Error";
  status: 400;
  detail: string;
  errors: Array<{ path: string; message: string; code?: string }>;
};
```

### `src/schema.ts`

```ts
import { z } from "zod";
import type { CreateOrderRequest, Result, ProblemJson, UserId } from "./types";

const USER_ID_RE = /^usr_[0-9a-f]{12}$/;
const SKU_RE = /^[A-Z0-9_]{3,32}$/;
const COUPON_RE = /^[A-Z0-9]{4,16}$/;

function asUserId(raw: string): UserId {
  // TODO: validate USER_ID_RE, throw if invalid (controlled assertion)
  return raw as UserId;
}

const MetadataSchema = z
  .record(z.union([z.string(), z.number(), z.boolean()]))
  .refine((obj) => Object.keys(obj).length <= 20, "metadata too large");

export const CreateOrderSchema = z
  .object({
    userId: z.string().refine((s) => USER_ID_RE.test(s), "invalid userId format")
      .transform((s) => asUserId(s)),
    items: z.array(
      z.object({
        sku: z.string().regex(SKU_RE, "invalid sku"),
        qty: z.number().int().min(1).max(999),
      }).strict()
    ).min(1).max(50),
    couponCode: z.string().regex(COUPON_RE, "invalid coupon").optional(),
    note: z.string().trim().max(200).optional(),
    metadata: MetadataSchema.optional(),
  })
  .strict(); // forbid unknown keys

function zodToProblemJson(err: z.ZodError): ProblemJson {
  // TODO: map err.issues to ProblemJson.errors
  // path should become "items[0].qty" style
  return {
    type: "https://errors.example.com/validation",
    title: "Validation Error",
    status: 400,
    detail: "Request validation failed",
    errors: err.issues.map((i) => ({
      path: i.path
        .map((p) => (typeof p === "number" ? `[${p}]` : p))
        .join(".")
        .replace(".[", "["),
      message: i.message,
      code: i.code,
    })),
  };
}

export function parseCreateOrderRequest(input: unknown): Result<CreateOrderRequest, ProblemJson> {
  // TODO: safeParse input
  const r = CreateOrderSchema.safeParse(input);
  if (!r.success) return { ok: false, error: zodToProblemJson(r.error) };
  return { ok: true, value: r.data };
}
```

### `src/http.ts`

```ts
import type { ProblemJson, Result } from "./types";

export type JsonRequest = { body: unknown };
export type JsonResponse =
  | { status: 200 | 201; json: unknown }
  | { status: 400; json: ProblemJson };

export function makeJsonHandler<TReq>(
  validate: (body: unknown) => Result<TReq, ProblemJson>,
  handler: (req: TReq) => Promise<unknown> | unknown
) {
  return async (req: JsonRequest): Promise<JsonResponse> => {
    const v = validate(req.body);
    if (!v.ok) return { status: 400, json: v.error };

    const out = await handler(v.value);
    return { status: 201, json: out };
  };
}
```

### `src/app.ts` (minimal, “đủ chạy”)

Bạn có thể dùng Express/Fastify. Skeleton dưới đây viết kiểu pseudo để bạn gắn framework nào cũng được:

```ts
import { makeJsonHandler } from "./http";
import { parseCreateOrderRequest } from "./schema";

const createOrder = makeJsonHandler(parseCreateOrderRequest, async (req) => {
  // req is fully typed: CreateOrderRequest
  // TODO: implement business stub
  return { orderId: "ord_deadbeefc0de", userId: req.userId, itemCount: req.items.length };
});

// TODO: wire to your HTTP framework:
// - parse JSON body -> pass as { body }
// - return status + json
```

---

## Tests (vitest) — “Definition of Done” phải rõ

### `test/kata25.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { parseCreateOrderRequest } from "../src/schema";

describe("kata25 - request validation", () => {
  it("accepts a valid request", () => {
    const r = parseCreateOrderRequest({
      userId: "usr_1a2b3c4d5e6f",
      items: [{ sku: "SKU_ABC123", qty: 2 }],
      couponCode: "TET2026",
      note: "  deliver after 6pm  ",
      metadata: { source: "ios", ab: true, v: 1 },
    });

    expect(r.ok).toBe(true);
    if (r.ok) {
      expect(r.value.userId).toBe("usr_1a2b3c4d5e6f");
      expect(r.value.note).toBe("deliver after 6pm"); // trimmed
    }
  });

  it("rejects unknown keys (strict)", () => {
    const r = parseCreateOrderRequest({
      userId: "usr_1a2b3c4d5e6f",
      items: [{ sku: "SKU_ABC123", qty: 2 }],
      hacker: "lol",
    });

    expect(r.ok).toBe(false);
    if (!r.ok) {
      expect(r.error.status).toBe(400);
      expect(r.error.errors.some(e => e.path.includes("hacker"))).toBe(true);
    }
  });

  it("rejects invalid userId format", () => {
    const r = parseCreateOrderRequest({
      userId: "USR_1A2B3C4D5E6F",
      items: [{ sku: "SKU_ABC123", qty: 2 }],
    });

    expect(r.ok).toBe(false);
    if (!r.ok) {
      expect(r.error.errors[0]?.path).toBe("userId");
    }
  });

  it("rejects items qty not int", () => {
    const r = parseCreateOrderRequest({
      userId: "usr_1a2b3c4d5e6f",
      items: [{ sku: "SKU_ABC123", qty: 1.5 }],
    });

    expect(r.ok).toBe(false);
    if (!r.ok) {
      expect(r.error.errors.some(e => e.path === "items[0].qty")).toBe(true);
    }
  });

  it("rejects metadata > 20 keys", () => {
    const metadata: Record<string, string> = {};
    for (let i = 0; i < 21; i++) metadata["k" + i] = "v";
    const r = parseCreateOrderRequest({
      userId: "usr_1a2b3c4d5e6f",
      items: [{ sku: "SKU_ABC123", qty: 1 }],
      metadata,
    });

    expect(r.ok).toBe(false);
  });
});
```

---

## Checks (Pass/Fail rõ ràng)

Bạn coi kata này “pass” khi:

1. **Không còn chỗ nào trong handler đọc `req.body` trực tiếp**.
2. `parseCreateOrderRequest(unknown)` trả về:

   * `{ ok: true, value: CreateOrderRequest }` với dữ liệu đã trim/transform
   * hoặc `{ ok: false, error: ProblemJson }` với `errors[].path` chuẩn
3. Unknown keys bị chặn (strict) và có test chứng minh.
4. TypeScript suy ra đúng kiểu cho `req` trong handler (gợi ý: hover type, hoặc viết `req.items[0].qty.toFixed()` phải compile OK).
5. **Chỉ có 1 chỗ “assert brand”** (`asUserId`) và nó được bảo vệ bởi regex.

---

## Stretch (Senior+ nâng level thật)

1. **Validate cả query/params/headers**:

   * `POST /v1/orders?dryRun=true` (`dryRun` coerce boolean)
   * `Idempotency-Key` header (string length 16..128)
2. **Coerce numbers**: cho phép `"qty": "2"` → parse thành number (cẩn thận: phải test `"2x"` fail).
3. **Error taxonomy**:

   * validation error (400)
   * malformed JSON (400, khác detail)
   * internal (500)
4. **Make a reusable boundary**:

   * `makeValidatedRoute({ body, query, params, headers }, handler)`
   * tất cả schema tách riêng, handler typed full `Req`.