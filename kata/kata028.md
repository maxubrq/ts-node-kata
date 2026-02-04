# Kata 28 — OpenAPI from Types: generate spec, avoid drift

## Goal

1. Có một nguồn chân lý duy nhất cho contract API:

* runtime validation (Kata 25)
* typed request/response (TS inference)
* OpenAPI spec (generated)

2. **Không drift**:

* đổi schema → type & OpenAPI đổi theo
* CI/test fail nếu spec không cập nhật / mismatch

3. Spec đủ dùng cho production:

* paths + methods
* requestBody + responses
* error responses chuẩn Problem+JSON (Kata 26)
* components schemas reusable

---

## Constraints

* ✅ OpenAPI phải được **generate tự động** từ code (không viết YAML tay).
* ✅ TypeScript types phải được infer từ schema (không define type riêng rồi lệch).
* ✅ Có script `pnpm openapi:gen` tạo `openapi.json`.
* ✅ Có test/CI guard: “spec file is up-to-date” (fail nếu quên regen).
* ✅ Tối thiểu 2 endpoints + 2 response codes/endpoints.
* ❌ Không chấp nhận “spec copy-paste” hoặc spec tự viết thủ công.

---

## Context (API demo đủ để luyện)

Bạn có 2 endpoints:

### 1) `POST /v1/orders`

* Request: `CreateOrderRequest`
* Response 201: `CreateOrderResponse`
* Error 400: `ValidationProblem` (Problem+JSON + `errors[]`)

### 2) `GET /v1/orders/{orderId}`

* Response 200: `OrderView`
* Error 404: `NotFoundProblem`

> Bonus: add `401 UnauthorizedProblem` nếu bạn muốn.

---

## Deliverables

`kata-28/`

* `src/schemas/`

  * `problem.schema.ts` (Problem base + specific problems)
  * `order.schema.ts` (CreateOrderRequest/Response, OrderView)
* `src/openapi.ts` (registry + generate function)
* `openapi.json` (generated artifact)
* `package.json` scripts
* `test/kata28.test.ts` (spec guard + sanity checks)

---

## Tooling đề xuất

* `zod`
* `@asteasolutions/zod-to-openapi` (mature, practical)
* `vitest`
* (optional) `openapi-types` để type-check OpenAPI doc

---

## Starter Skeleton (copy vào repo và điền TODO)

### `src/schemas/problem.schema.ts`

```ts
import { z } from "zod";
import { extendZodWithOpenApi } from "@asteasolutions/zod-to-openapi";

extendZodWithOpenApi(z);

// Base Problem (RFC 9457-ish)
export const ProblemSchema = z.object({
  type: z.string().openapi({ example: "https://errors.example.com/validation" }),
  title: z.string().openapi({ example: "Validation Error" }),
  status: z.number().int().openapi({ example: 400 }),
  detail: z.string().optional(),
  instance: z.string().optional(),

  traceId: z.string().openapi({ example: "trace_123" }),
  code: z.string().optional(),
}).openapi("Problem");

// ValidationProblem extends Problem + errors[]
export const ValidationProblemSchema = ProblemSchema.extend({
  errors: z.array(z.object({
    path: z.string().openapi({ example: "items[0].qty" }),
    message: z.string().openapi({ example: "must be int" }),
    code: z.string().optional(),
  })).openapi("ValidationErrors"),
}).openapi("ValidationProblem");

export const NotFoundProblemSchema = ProblemSchema.extend({}).openapi("NotFoundProblem");

// Types inferred (single source of truth)
export type Problem = z.infer<typeof ProblemSchema>;
export type ValidationProblem = z.infer<typeof ValidationProblemSchema>;
export type NotFoundProblem = z.infer<typeof NotFoundProblemSchema>;
```

### `src/schemas/order.schema.ts`

```ts
import { z } from "zod";
import { extendZodWithOpenApi } from "@asteasolutions/zod-to-openapi";

extendZodWithOpenApi(z);

const USER_ID_RE = /^usr_[0-9a-f]{12}$/;
const ORDER_ID_RE = /^ord_[0-9a-f]{12}$/;
const SKU_RE = /^[A-Z0-9_]{3,32}$/;

export const CreateOrderRequestSchema = z.object({
  userId: z.string().regex(USER_ID_RE).openapi({ example: "usr_1a2b3c4d5e6f" }),
  items: z.array(z.object({
    sku: z.string().regex(SKU_RE).openapi({ example: "SKU_ABC123" }),
    qty: z.number().int().min(1).max(999).openapi({ example: 2 }),
  }).strict()).min(1).max(50),
  couponCode: z.string().regex(/^[A-Z0-9]{4,16}$/).optional(),
  note: z.string().trim().max(200).optional(),
}).strict().openapi("CreateOrderRequest");

export const CreateOrderResponseSchema = z.object({
  orderId: z.string().regex(ORDER_ID_RE).openapi({ example: "ord_deadbeefc0de" }),
}).openapi("CreateOrderResponse");

export const OrderViewSchema = z.object({
  orderId: z.string().regex(ORDER_ID_RE),
  userId: z.string().regex(USER_ID_RE),
  status: z.enum(["PENDING", "PAID", "CANCELLED"]),
  itemCount: z.number().int().nonnegative(),
  createdAt: z.string().datetime(), // ISO
}).openapi("OrderView");

export type CreateOrderRequest = z.infer<typeof CreateOrderRequestSchema>;
export type CreateOrderResponse = z.infer<typeof CreateOrderResponseSchema>;
export type OrderView = z.infer<typeof OrderViewSchema>;
```

### `src/openapi.ts`

```ts
import { OpenAPIRegistry, OpenApiGeneratorV3 } from "@asteasolutions/zod-to-openapi";
import { z } from "zod";

import {
  ProblemSchema,
  ValidationProblemSchema,
  NotFoundProblemSchema,
} from "./schemas/problem.schema";

import {
  CreateOrderRequestSchema,
  CreateOrderResponseSchema,
  OrderViewSchema,
} from "./schemas/order.schema";

export function buildOpenApiDoc() {
  const registry = new OpenAPIRegistry();

  // Register component schemas
  registry.register("Problem", ProblemSchema);
  registry.register("ValidationProblem", ValidationProblemSchema);
  registry.register("NotFoundProblem", NotFoundProblemSchema);

  registry.register("CreateOrderRequest", CreateOrderRequestSchema);
  registry.register("CreateOrderResponse", CreateOrderResponseSchema);
  registry.register("OrderView", OrderViewSchema);

  // Path: POST /v1/orders
  registry.registerPath({
    method: "post",
    path: "/v1/orders",
    description: "Create a new order",
    request: {
      body: {
        content: {
          "application/json": { schema: CreateOrderRequestSchema },
        },
      },
    },
    responses: {
      201: {
        description: "Created",
        content: {
          "application/json": { schema: CreateOrderResponseSchema },
        },
      },
      400: {
        description: "Validation Error",
        content: {
          "application/problem+json": { schema: ValidationProblemSchema },
        },
      },
    },
  });

  // Path: GET /v1/orders/{orderId}
  registry.registerPath({
    method: "get",
    path: "/v1/orders/{orderId}",
    description: "Get an order by id",
    request: {
      params: z.object({
        orderId: z.string().regex(/^ord_[0-9a-f]{12}$/),
      }),
    },
    responses: {
      200: {
        description: "OK",
        content: {
          "application/json": { schema: OrderViewSchema },
        },
      },
      404: {
        description: "Not Found",
        content: {
          "application/problem+json": { schema: NotFoundProblemSchema },
        },
      },
    },
  });

  const generator = new OpenApiGeneratorV3(registry.definitions);

  return generator.generateDocument({
    openapi: "3.0.3",
    info: {
      title: "Kata 28 API",
      version: "1.0.0",
    },
    servers: [{ url: "http://localhost:3000" }],
  });
}
```

### `scripts/openapi-gen.ts`

```ts
import { writeFileSync } from "node:fs";
import { buildOpenApiDoc } from "../src/openapi";

const doc = buildOpenApiDoc();
writeFileSync("openapi.json", JSON.stringify(doc, null, 2), "utf8");
console.log("generated openapi.json");
```

### `package.json` scripts

```json
{
  "scripts": {
    "openapi:gen": "tsx scripts/openapi-gen.ts",
    "test": "vitest"
  }
}
```

---

## Tests (guard “no drift”)

### `test/kata28.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { readFileSync } from "node:fs";
import { buildOpenApiDoc } from "../src/openapi";

describe("kata28 - openapi generation", () => {
  it("openapi.json is up-to-date (no drift)", () => {
    const generated = buildOpenApiDoc();
    const file = JSON.parse(readFileSync("openapi.json", "utf8"));

    expect(file).toEqual(generated);
  });

  it("spec contains expected routes and content-types", () => {
    const doc = buildOpenApiDoc();
    const paths = doc.paths;

    expect(paths["/v1/orders"]).toBeTruthy();
    expect(paths["/v1/orders"].post).toBeTruthy();

    const post = paths["/v1/orders"].post;
    expect(post.responses["400"].content["application/problem+json"]).toBeTruthy();

    const get = paths["/v1/orders/{orderId}"].get;
    expect(get.responses["404"].content["application/problem+json"]).toBeTruthy();
  });
});
```

**Cách chạy**

1. `pnpm openapi:gen` tạo `openapi.json`
2. `pnpm test`

* Nếu bạn đổi schema mà quên regen spec → test #1 fail ngay. Drift chết tại chỗ.

---

## Checks (Definition of Done)

Bạn pass kata này khi:

1. `openapi.json` được generate từ code, **không chỉnh tay**.
2. Đổi `CreateOrderRequestSchema` (vd thêm field) → chạy `test` sẽ fail nếu chưa `openapi:gen`.
3. Spec có:

* `/v1/orders` POST có requestBody JSON + 201 + 400 problem+json
* `/v1/orders/{orderId}` GET có param + 200 + 404 problem+json

4. TypeScript types dùng `z.infer<...>` (không có type thủ công bị lệch).

---

## Stretch (Senior+ thật)

1. **Enforce `operationId`** + tag grouping (Orders, Payments…).
2. **Shared error response helper**:

   * `problemResponse(400, ValidationProblemSchema)` để giảm lặp.
3. **Contract test**:

   * spin server, fetch `/openapi.json`, compare với generated doc.
4. **Monorepo contract**:

   * publish `@internal/contracts` chứa schemas + generated openapi, backend & frontend cùng dùng.
5. **Breaking-change guard**:

   * snapshot spec + diff trong PR (tối thiểu: show diff, hoặc fail nếu change mà không bump version).