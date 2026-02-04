# Kata 04.2 — ValidationResult (Accumulate errors)

## Goal

1. Tạo `ValidationResult<T>` dạng:

* Success: `{ ok:true; value:T }`
* Failure: `{ ok:false; errors: ValidationError[] }` (nhiều lỗi)

2. Helpers để compose validations mà **gom lỗi**:

* `valid / invalid`
* `mapV`
* `andThenV` (fail-fast theo bước phụ thuộc)
* `combineV` (run độc lập, gom lỗi)

3. Refactor `parseCreateOrderInput` (từ Kata 04) để:

* validate `userId`, `amount`, (bonus: `couponCode?`) **cùng lúc**
* trả về danh sách lỗi đầy đủ

---

## Constraints

* ❌ Không throw
* ❌ Không `any`
* ✅ Phân biệt:

  * **independent validations** → dùng `combineV` để gom lỗi
  * **dependent step** (cần data đã parse) → dùng `andThenV`

---

## Error model

Dùng lại:

```ts
export type ValidationError = { type: "validation"; field: string; message: string };
```

---

## Starter skeleton (điền TODO)

```ts
/* kata04_2.ts */

export type ValidationError = { type: "validation"; field: string; message: string };

export type ValidationResult<T> =
  | { ok: true; value: T }
  | { ok: false; errors: ValidationError[] };

export const valid = <T>(value: T): ValidationResult<T> => ({ ok: true, value });
export const invalid = (...errors: ValidationError[]): ValidationResult<never> => ({
  ok: false,
  errors,
});

// TODO: mapV
export function mapV<T, U>(vr: ValidationResult<T>, fn: (v: T) => U): ValidationResult<U> {
  // TODO
  return vr as any;
}

// TODO: andThenV (dependent, fail-fast when already invalid)
export function andThenV<T, U>(vr: ValidationResult<T>, fn: (v: T) => ValidationResult<U>): ValidationResult<U> {
  // TODO
  return vr as any;
}

// TODO: combineV (independent, accumulate all errors)
export function combineV<T extends readonly ValidationResult<any>[]>(
  ...results: T
): ValidationResult<{ [K in keyof T]: T[K] extends ValidationResult<infer V> ? V : never }> {
  // TODO: if all ok -> tuple of values
  // else -> merge all errors into one array
  return invalid({ type: "validation", field: "TODO", message: "TODO" });
}

// ---------- IDs (minimal branded) ----------
declare const __brand: unique symbol;
type Brand<T, Name extends string> = T & { readonly [__brand]: Name };
export type UserId = Brand<string, "UserId">;

const USER_ID_RE = /^usr_[0-9a-f]{12}$/;

export function validateUserId(x: unknown): ValidationResult<UserId> {
  if (typeof x !== "string") return invalid({ type: "validation", field: "userId", message: "must be string" });
  if (!USER_ID_RE.test(x)) return invalid({ type: "validation", field: "userId", message: "invalid format" });
  return valid(x as UserId);
}

export function validateAmount(x: unknown): ValidationResult<number> {
  if (typeof x !== "number" || !Number.isFinite(x)) {
    return invalid({ type: "validation", field: "amount", message: "must be a finite number" });
  }
  if (x <= 0) return invalid({ type: "validation", field: "amount", message: "must be > 0" });
  return valid(x);
}

// Bonus optional field
export type CouponCode = Brand<string, "CouponCode">;
const COUPON_RE = /^[A-Z0-9]{6}$/;

export function validateCoupon(x: unknown): ValidationResult<CouponCode | undefined> {
  if (x === undefined) return valid(undefined);
  if (typeof x !== "string") return invalid({ type: "validation", field: "couponCode", message: "must be string" });
  if (!COUPON_RE.test(x)) return invalid({ type: "validation", field: "couponCode", message: "must be 6 chars A-Z0-9" });
  return valid(x as CouponCode);
}

// ---------- Main parse ----------
export type CreateOrderInput = {
  userId: UserId;
  amount: number;
  couponCode?: CouponCode;
};

export function parseCreateOrderInput(body: unknown): ValidationResult<CreateOrderInput> {
  if (typeof body !== "object" || body === null) {
    return invalid({ type: "validation", field: "body", message: "must be object" });
  }

  const b = body as Record<string, unknown>;

  // TODO: validate userId, amount, coupon independently and combine
  // const combined = combineV(validateUserId(b.userId), validateAmount(b.amount), validateCoupon(b.couponCode))
  // then mapV to shape CreateOrderInput

  return invalid({ type: "validation", field: "TODO", message: "TODO" });
}

// ---------- Demo checks ----------
const good = { userId: "usr_1a2b3c4d5e6f", amount: 100, couponCode: "ABC123" };
const bad = { userId: "usr_NOTHEX", amount: -5, couponCode: "nope" };

console.log("good:", parseCreateOrderInput(good));
console.log("bad:", parseCreateOrderInput(bad));
// bad should include 3 errors: userId, amount, couponCode
```

> Trong skeleton có `as any` placeholder. Khi bạn làm xong: **xóa sạch**.

---

## Required behaviors

* `parseCreateOrderInput(bad)` trả `{ ok:false, errors:[...] }` với **đủ 3 lỗi**
* `parseCreateOrderInput(good)` trả `{ ok:true, value:{...} }`
* `combineV(valid(1), invalid(e1), invalid(e2))` → invalid `[e1,e2]` (không mất lỗi)

---

## Checks (Definition of Done)

* Không `any`
* `combineV` preserve thứ tự lỗi theo thứ tự arguments (để predictable)
* `mapV` không thay đổi lỗi, chỉ map value khi ok
* `andThenV` chỉ chạy fn khi ok

---

## Stretch (rất đáng làm)

1. `combineVObj({ userId: ..., amount: ..., couponCode: ... })` trả về object thay vì tuple
2. `path` cho nested fields: `"shipping.address.city"`
3. Add `warnings` (soft validation) song song với errors