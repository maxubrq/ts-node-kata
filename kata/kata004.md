# Kata 04 — Result/Either Pattern (No-throw Domain)

## Goal

1. Tạo `Result<T, E>` (hoặc `Either<E, T>`) dùng thống nhất cho domain/application layer
2. Viết helpers để:

* `ok / err`
* `map / mapErr`
* `andThen` (flatMap)
* `match`
* `combine` (gộp nhiều Result)

3. Refactor một flow nhỏ: **parse input → validate → execute** mà **không throw**

---

## Context

Bạn có API handler nhận payload `unknown`. Bạn muốn:

* Parse & validate input
* Nếu lỗi: trả về error typed (đủ info để map sang HTTP 400/404/409…)
* Nếu ok: xử lý business

Mục tiêu là “control flow rõ ràng”, không bị try/catch rải khắp nơi.

---

## Constraints

* ❌ Không dùng `any`
* ❌ Không throw trong domain functions (trừ “edge adapters” nếu muốn)
* ✅ Errors phải là **typed union** (discriminated union)
* ✅ Có compile-time checks + vài test runtime (tối thiểu bằng node chạy file cũng được)

---

## Deliverables

File `kata04.ts` gồm:

1. `Result<T,E>` type
2. Constructors + helpers
3. Error model (union)
4. 1 flow thực tế: **CreateOrder** (dùng branded ID kata 03 nếu muốn)

---

## Spec chi tiết

### 1) Result type

Bạn phải dùng discriminated union:

* `{ ok: true; value: T }`
* `{ ok: false; error: E }`

### 2) Error model

Tạo error union kiểu:

* `ValidationError` `{ type:"validation"; field: string; message: string }`
* `NotFoundError` `{ type:"not_found"; resource: "user" | "order"; id: string }`
* `ConflictError` `{ type:"conflict"; message: string }`

Gộp thành `AppError = ValidationError | NotFoundError | ConflictError`

### 3) Flow: CreateOrder

Input `unknown` (giả lập HTTP body). Rule:

* body phải là object có `userId` (string dạng `usr_` + 12 hex) và `amount` (number > 0)
* parse userId thành `UserId` (branded) bằng `parseUserId` kiểu Result
* giả lập `getUser(userId)`:

  * nếu userId endsWith `"000000000000"` thì not found
  * else ok
* tạo order id `"ord_deadbeefc0de"` (branded `OrderId`)
* trả `Result<{ orderId: OrderId }, AppError>`

Không throw ở bất kỳ bước domain nào.

---

## Starter skeleton (điền TODO)

```ts
/* kata04.ts */

// ---------- Result ----------
export type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

export const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
export const err = <E>(error: E): Result<never, E> => ({ ok: false, error });

// TODO: map
export function map<T, E, U>(r: Result<T, E>, fn: (v: T) => U): Result<U, E> {
  // TODO
  return r as any;
}

// TODO: mapErr
export function mapErr<T, E, F>(r: Result<T, E>, fn: (e: E) => F): Result<T, F> {
  // TODO
  return r as any;
}

// TODO: andThen (flatMap)
export function andThen<T, E, U, F>(
  r: Result<T, E>,
  fn: (v: T) => Result<U, F>
): Result<U, E | F> {
  // TODO
  return r as any;
}

// TODO: match
export function match<T, E, R>(
  r: Result<T, E>,
  arms: { ok: (v: T) => R; err: (e: E) => R }
): R {
  // TODO
  return r as any;
}

// TODO: combine (all-or-first-error)
export function combine<T extends readonly Result<any, any>[]>(
  ...results: T
): Result<
  { [K in keyof T]: T[K] extends Result<infer V, any> ? V : never },
  T[number] extends Result<any, infer E> ? E : never
> {
  // TODO
  return err("TODO" as any);
}

// ---------- Errors ----------
export type ValidationError = { type: "validation"; field: string; message: string };
export type NotFoundError = { type: "not_found"; resource: "user" | "order"; id: string };
export type ConflictError = { type: "conflict"; message: string };
export type AppError = ValidationError | NotFoundError | ConflictError;

// ---------- Branded IDs (minimal, reuse kata03 style) ----------
declare const __brand: unique symbol;
type Brand<T, Name extends string> = T & { readonly [__brand]: Name };

export type UserId = Brand<string, "UserId">;
export type OrderId = Brand<string, "OrderId">;

const USER_ID_RE = /^usr_[0-9a-f]{12}$/;
const ORDER_ID_RE = /^ord_[0-9a-f]{12}$/;

// TODO: parseUserId returns Result
export function parseUserId(x: unknown): Result<UserId, ValidationError> {
  // TODO: no throw
  return err({ type: "validation", field: "userId", message: "TODO" });
}

export function makeOrderId(raw: string): Result<OrderId, ValidationError> {
  // TODO: validate + brand, no throw
  return err({ type: "validation", field: "orderId", message: "TODO" });
}

// ---------- Domain functions (no throw) ----------
type CreateOrderInput = { userId: UserId; amount: number };

export function parseCreateOrderInput(body: unknown): Result<CreateOrderInput, ValidationError> {
  // TODO:
  // - body must be object
  // - userId must be valid via parseUserId
  // - amount must be number > 0
  return err({ type: "validation", field: "body", message: "TODO" });
}

export function getUser(userId: UserId): Result<{ userId: UserId }, NotFoundError> {
  // rule: endsWith 000000000000 => not found
  const raw = userId as unknown as string; // NOTE: you should avoid this if you can (stretch)
  if (raw.endsWith("000000000000")) {
    return err({ type: "not_found", resource: "user", id: raw });
  }
  return ok({ userId });
}

export function createOrder(body: unknown): Result<{ orderId: OrderId }, AppError> {
  // TODO: compose parseCreateOrderInput -> getUser -> makeOrderId
  // Use andThen/map/mapErr etc.
  return err({ type: "conflict", message: "TODO" });
}

// ---------- Compile-time / runtime checks ----------
const goodBody = { userId: "usr_1a2b3c4d5e6f", amount: 100 };
const badBody1 = { userId: "usr_NOTHEX", amount: 100 };
const badBody2 = { userId: "usr_1a2b3c4d5e6f", amount: -5 };

console.log("good:", createOrder(goodBody));
console.log("bad1:", createOrder(badBody1));
console.log("bad2:", createOrder(badBody2));
```

> Lưu ý: trong skeleton mình có chỗ `return r as any;` và `as unknown as string` chỉ để “cắm cờ TODO”. Trong bài làm của bạn: **xóa sạch `any`**. Với `userId` raw string, bạn xử lý bằng cách khác (gợi ý ở Stretch).

---

## Required behaviors (phải đạt)

1. `createOrder(goodBody)` trả `{ ok:true, value:{orderId: ...} }`
2. invalid body trả `{ ok:false, error:{ type:"validation", ... } }`
3. user not found trả `{ ok:false, error:{ type:"not_found", ... } }`
4. Không throw

---

## Checks (Definition of Done)

* Không còn `any` trong file
* Không dùng throw trong parse/validate/domain
* Helpers hoạt động đúng (map/andThen/…)
* `combine(ok(1), ok("x"))` trả ok `[1,"x"]`, nếu có 1 err thì trả err đầu tiên

---

## Stretch

1. **Type-safe unwrap string**: tạo helper `toString(id: Brand<string, X>): string` để không cần `as unknown as string`
2. `collectErrors`: thay vì fail-fast, gom nhiều validation errors thành mảng
3. `ResultAsync` (Promise<Result<...>>) + helper `andThenAsync` để compose IO