# Kata 08 — Enforce Immutability at Boundaries

## Goal

1. Tạo type utilities để “đóng băng” dữ liệu ở boundary:

* `DeepReadonly<T>`
* `DeepImmutable<T>` (tùy bạn: alias hoặc khác chút)

2. Viết `freezeDeep(value)` runtime (optional nhưng recommended) để:

* dev mutate → crash ngay (dev mode)
* tránh bug “lỡ tay sửa request object”

3. Thiết kế API contract:

* Handler nhận `DeepReadonly<Input>`
* Domain/service không thể mutate input compile-time

4. Viết 1 flow nhỏ chứng minh:

* mutation bị chặn compile-time
* mutation bị chặn runtime (nếu bật freeze)

---

## Context

Trong Node/TS, request body thường được chuyền qua nhiều layer. Bug phổ biến:

* Một middleware sửa `req.body`
* Service sửa input object để “tận dụng”, làm side-effects khó trace
* Trong async chain, mutate gây race-ish behavior

Bạn muốn: **mọi input từ bên ngoài** (HTTP, queue message, config) phải được treat như immutable.

---

## Constraints

* ❌ Không `any`
* ❌ Không “lách” bằng type assertion để mutate
* ✅ Deep readonly phải xử lý:

  * object lồng nhau
  * arrays / readonly arrays
  * tuples
  * Map/Set (basic handling)
  * function giữ nguyên
* ✅ Có compile-time checks `@ts-expect-error`

---

## Deliverables

File `kata08.ts` gồm:

1. `DeepReadonly<T>` type
2. `freezeDeep<T>(value: T): DeepReadonly<T>` runtime helper
3. `handleRequest(input: DeepReadonly<CreateOrderInput>)`
4. `service(input: DeepReadonly<CreateOrderInput>)`
5. Checks

---

## Domain spec (simple)

```ts
type CreateOrderInput = {
  userId: string;
  items: { sku: string; qty: number }[];
  meta?: {
    coupon?: string;
    tags?: string[];
  };
};
```

---

## Starter skeleton (điền TODO)

```ts
/* kata08.ts */

// ---------- TODO 1: DeepReadonly<T> ----------
export type DeepReadonly<T> =
  // TODO: handle primitives, functions, arrays/tuples, maps/sets, objects
  never;

// ---------- TODO 2: freezeDeep runtime ----------
export function freezeDeep<T>(value: T): DeepReadonly<T> {
  // TODO:
  // - recursively Object.freeze
  // - handle arrays
  // - handle plain objects
  // - Map/Set optional (can freeze container, not entries)
  // - return typed DeepReadonly
  return value as any;
}

// ---------- Example domain ----------
type CreateOrderInput = {
  userId: string;
  items: { sku: string; qty: number }[];
  meta?: {
    coupon?: string;
    tags?: string[];
  };
};

// boundary: handler receives immutable input
export function handleCreateOrder(input: DeepReadonly<CreateOrderInput>) {
  // pass through layers
  return createOrderService(input);
}

export function createOrderService(input: DeepReadonly<CreateOrderInput>) {
  // TODO: should NOT be able to mutate input
  // e.g. input.userId = "x" should be compile error
  // input.items.push(...) should be compile error
  // input.items[0].qty++ should be compile error

  // Instead, derive a new object
  const totalQty = input.items.reduce((sum, it) => sum + it.qty, 0);
  return { ok: true as const, totalQty };
}

// ---------- Compile-time checks ----------
const raw: CreateOrderInput = {
  userId: "u1",
  items: [{ sku: "A", qty: 1 }],
  meta: { coupon: "OFF10", tags: ["vip"] },
};

const immutable = freezeDeep(raw);

// @ts-expect-error - cannot assign
immutable.userId = "u2";

// @ts-expect-error - cannot push
immutable.items.push({ sku: "B", qty: 1 });

// @ts-expect-error - nested mutation forbidden
immutable.items[0].qty = 99;

// @ts-expect-error - optional nested array mutation forbidden
immutable.meta?.tags?.push("new");

// should work: reading
immutable.items[0].sku;

// boundary call
handleCreateOrder(immutable);

// ---------- Runtime check (optional) ----------
try {
  (immutable as unknown as { userId: string }).userId = "hacked";
  console.log("runtime mutation succeeded (should not if frozen)");
} catch (e) {
  console.log("runtime mutation blocked (good):", String(e));
}
```

> `freezeDeep` hiện đang `as any` placeholder. Bài làm của bạn: **xóa any**.

---

## Required behaviors

1. Tất cả các dòng `@ts-expect-error` phải thật sự error
2. `createOrderService` chỉ đọc input, không mutate
3. `freezeDeep` trả về type `DeepReadonly<T>`
4. Nếu bạn implement freeze runtime chuẩn: mutation runtime phải throw (trong strict mode)

---

## Definition of Done

* Không `any`
* `DeepReadonly` hoạt động đúng cho nested object/array
* Boundary dùng `DeepReadonly` (không chỉ “tự giác”)
* Có 1 ví dụ “derive new object instead of mutate”

---

## Stretch (Senior+ thật)

1. `DeepReadonlyExcept<T, Keys>`: cho phép mutate một số field nhất định (rare but useful)
2. `DeepClone<T>(value: DeepReadonly<T>): T` để tạo copy mutable ở nơi cần
3. `Immutable Update Helpers` kiểu `setIn(obj, path, value)` / `produce` style (không dùng immer)