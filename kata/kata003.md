# Kata 03 — Branded IDs (Nominal typing in TS)

## Goal

1. Tạo **branded types** cho các ID dạng string để:

* `UserId` không thể truyền nhầm sang chỗ cần `OrderId`
* Không bị “string-itis” lan khắp codebase

2. Có **factory + parser** để biến `unknown/string` thành branded ID một cách an toàn.

3. Enforce rule:

* Từ runtime input → **parse** → domain ID
* Trong domain code: không còn `string` trần cho ID

---

## Context

Bạn có service:

* `getUser(userId: UserId)`
* `getOrder(orderId: OrderId)`
* `createOrder(userId: UserId): OrderId`

Dev hay lỡ tay gọi `getOrder(userId)` và TypeScript vẫn cho nếu cả 2 là `string`. Bạn phải chặn.

---

## Constraints

* ❌ Không dùng `any`
* ❌ Không `as any`
* ⚠️ Hạn chế tối đa type assertion (chỉ cho phép ở **1 chỗ**: trong factory function)
* ✅ Có unit tests hoặc compile-time checks (`@ts-expect-error`)
* ✅ Có `isUserId` / `parseUserId` (type guard / parser)

---

## Deliverables

File `kata03.ts` gồm:

1. `Brand<T, Name>`
2. `UserId`, `OrderId`
3. Factory: `makeUserId`, `makeOrderId`
4. Parser: `parseUserId`, `parseOrderId` (từ `unknown`)
5. Ví dụ function signatures + compile-time checks

---

## Domain rules

* UserId format: `"usr_" + 12 hex` (vd: `usr_1a2b3c4d5e6f`)
* OrderId format: `"ord_" + 12 hex` (vd: `ord_deadbeefc0de`)
* Hex: `[0-9a-f]` lowercase

---

## Starter skeleton (điền TODO)

```ts
/* kata03.ts */

// ---------- Brand utility ----------
declare const __brand: unique symbol;

export type Brand<T, Name extends string> = T & {
  readonly [__brand]: Name;
};

// ---------- Branded IDs ----------
export type UserId = Brand<string, "UserId">;
export type OrderId = Brand<string, "OrderId">;

// ---------- Runtime validators ----------
const USER_ID_RE = /^usr_[0-9a-f]{12}$/;
const ORDER_ID_RE = /^ord_[0-9a-f]{12}$/;

// TODO: type guard
export function isUserId(x: unknown): x is UserId {
  // TODO
  return false;
}

export function isOrderId(x: unknown): x is OrderId {
  // TODO
  return false;
}

// ---------- Factories (ONLY PLACE you may assert) ----------
export function makeUserId(raw: string): UserId {
  // TODO: validate with regex; throw if invalid
  // TODO: return branded
  throw new Error("TODO");
}

export function makeOrderId(raw: string): OrderId {
  // TODO
  throw new Error("TODO");
}

// ---------- Parsers (safe, no throw) ----------
export type ParseResult<T> =
  | { ok: true; value: T }
  | { ok: false; error: string };

export function parseUserId(x: unknown): ParseResult<UserId> {
  // TODO: if string and matches, ok true
  // else ok false with reason
  return { ok: false, error: "TODO" };
}

export function parseOrderId(x: unknown): ParseResult<OrderId> {
  // TODO
  return { ok: false, error: "TODO" };
}

// ---------- Example usage ----------
export function getUser(userId: UserId) {
  return { userId };
}

export function getOrder(orderId: OrderId) {
  return { orderId };
}

export function createOrder(userId: UserId): OrderId {
  // pretend generate
  return makeOrderId("ord_deadbeefc0de");
}

// ---------- Compile-time checks ----------
const u1 = makeUserId("usr_1a2b3c4d5e6f");
const o1 = makeOrderId("ord_deadbeefc0de");

getUser(u1);
getOrder(o1);

// @ts-expect-error - cannot use OrderId where UserId is required
getUser(o1);

// @ts-expect-error - cannot use UserId where OrderId is required
getOrder(u1);

// @ts-expect-error - raw string is not allowed
getUser("usr_1a2b3c4d5e6f");

// ---------- Parser checks ----------
const p1 = parseUserId("usr_1a2b3c4d5e6f");
if (p1.ok) {
  getUser(p1.value);
}

const p2 = parseOrderId("usr_1a2b3c4d5e6f");
// p2 should be ok: false
```

---

## Tests bạn nên có (runtime)

Nếu bạn dùng `vitest`, tối thiểu:

* `makeUserId`:

  * accept đúng format
  * throw với `"usr_"` nhưng sai length
  * throw với uppercase hex
* `parseUserId`:

  * ok true cho valid
  * ok false cho non-string / invalid
* `isUserId`:

  * true cho valid string
  * false cho invalid

---

## Checks (Definition of Done)

* `UserId` và `OrderId` không assignable qua lại (compile-time)
* Không thể gọi function nhận `UserId` bằng string trần
* Parser/factory chạy đúng rule format
* Type assertion chỉ xuất hiện trong `makeUserId/makeOrderId` (một nơi, có kiểm soát)

---

## Stretch (Senior+ real-world)

1. **Generic ID module**: tạo `createBrandedId(prefix, regex, brandName)` sinh ra `make/is/parse` cho mọi ID.
2. Add `toString(id)` helper + JSON serialization strategy (để log/trace không mất brand).
3. Integrate với Zod (nếu bạn dùng): `z.string().regex(...).transform(makeUserId)` nhưng vẫn giữ branded type.