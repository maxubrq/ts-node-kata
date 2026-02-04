# Bonus Kata 04.1 — ResultAsync (Promise<Result<…>>) + Timeout

## Goal

1. Tạo `ResultAsync<T,E> = Promise<Result<T,E>>`
2. Helpers:

* `okAsync / errAsync`
* `mapAsync / mapErrAsync`
* `andThenAsync` (flatMap async)
* `fromPromise` (wrap promise → ResultAsync)
* `withTimeout` (timeout + abort)

3. Refactor flow `createOrderAsync`:

* parse input (sync Result)
* fetch user (async)
* call payment gateway (async, có timeout)
* return `ResultAsync<{orderId}, AppError>`

---

## Constraints

* ❌ Không throw trong domain/app flow (adapter layer có thể throw nhưng phải được wrap)
* ✅ Timeout phải tạo **typed error** (ví dụ `conflict` hoặc thêm `timeout` error type)
* ✅ Không dùng `any`

---

## Deliverables

File `kata04_bonus.ts` gồm:

1. `Result`, `ResultAsync` + helpers
2. `withTimeout`
3. `createOrderAsync(body)` chạy demo được (console log)

---

## Starter skeleton (điền TODO)

```ts
/* kata04_bonus.ts */

// ---------- Result ----------
export type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

export const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
export const err = <E>(error: E): Result<never, E> => ({ ok: false, error });

// ---------- ResultAsync ----------
export type ResultAsync<T, E> = Promise<Result<T, E>>;

export const okAsync = <T>(value: T): ResultAsync<T, never> => Promise.resolve(ok(value));
export const errAsync = <E>(error: E): ResultAsync<never, E> => Promise.resolve(err(error));

// TODO: mapAsync
export async function mapAsync<T, E, U>(
  ra: ResultAsync<T, E>,
  fn: (v: T) => U | Promise<U>
): ResultAsync<U, E> {
  // TODO
  return err(null as any);
}

// TODO: mapErrAsync
export async function mapErrAsync<T, E, F>(
  ra: ResultAsync<T, E>,
  fn: (e: E) => F | Promise<F>
): ResultAsync<T, F> {
  // TODO
  return err(null as any);
}

// TODO: andThenAsync
export async function andThenAsync<T, E, U, F>(
  ra: ResultAsync<T, E>,
  fn: (v: T) => ResultAsync<U, F>
): ResultAsync<U, E | F> {
  // TODO
  return err(null as any);
}

// TODO: fromPromise
export async function fromPromise<T, E>(
  p: Promise<T>,
  mapError: (cause: unknown) => E
): ResultAsync<T, E> {
  // TODO: try/catch ONLY here, convert to Result
  return err(mapError("TODO"));
}

// ---------- Errors ----------
export type ValidationError = { type: "validation"; field: string; message: string };
export type NotFoundError = { type: "not_found"; resource: "user" | "order"; id: string };
export type ConflictError = { type: "conflict"; message: string };
export type TimeoutError = { type: "timeout"; operation: string; ms: number };

export type AppError = ValidationError | NotFoundError | ConflictError | TimeoutError;

// ---------- IDs (minimal) ----------
declare const __brand: unique symbol;
type Brand<T, Name extends string> = T & { readonly [__brand]: Name };
export type UserId = Brand<string, "UserId">;
export type OrderId = Brand<string, "OrderId">;

const USER_ID_RE = /^usr_[0-9a-f]{12}$/;
const ORDER_ID_RE = /^ord_[0-9a-f]{12}$/;

export function parseUserId(x: unknown): Result<UserId, ValidationError> {
  if (typeof x !== "string") return err({ type: "validation", field: "userId", message: "must be string" });
  if (!USER_ID_RE.test(x)) return err({ type: "validation", field: "userId", message: "invalid format" });
  return ok(x as UserId); // allowed assertion spot (or keep in a factory)
}

export function makeOrderId(raw: string): Result<OrderId, ValidationError> {
  if (!ORDER_ID_RE.test(raw)) return err({ type: "validation", field: "orderId", message: "invalid format" });
  return ok(raw as OrderId);
}

// ---------- Timeout helper ----------
export function withTimeout<T>(
  p: Promise<T>,
  ms: number,
  onTimeout: () => void = () => {}
): Promise<T> {
  // TODO: reject on timeout; ensure timer cleanup
  return p;
}

// ---------- Domain/app flow ----------
type CreateOrderInput = { userId: UserId; amount: number };

export function parseCreateOrderInput(body: unknown): Result<CreateOrderInput, ValidationError> {
  if (typeof body !== "object" || body === null) {
    return err({ type: "validation", field: "body", message: "must be object" });
  }
  const b = body as Record<string, unknown>;

  const u = parseUserId(b.userId);
  if (!u.ok) return u;

  const amount = b.amount;
  if (typeof amount !== "number" || !Number.isFinite(amount) || amount <= 0) {
    return err({ type: "validation", field: "amount", message: "must be number > 0" });
  }

  return ok({ userId: u.value, amount });
}

// Simulated async dependencies:
export function getUserAsync(userId: UserId): ResultAsync<{ userId: UserId }, NotFoundError> {
  const raw = userId as unknown as string;
  return fromPromise(
    new Promise((resolve, reject) => {
      setTimeout(() => {
        if (raw.endsWith("000000000000")) reject(new Error("not found"));
        else resolve({ userId });
      }, 30);
    }),
    () => ({ type: "not_found", resource: "user", id: raw })
  );
}

// Payment gateway simulation (sometimes slow/fails)
export function chargeAsync(_userId: UserId, _amount: number): Promise<{ receiptId: string }> {
  return new Promise((resolve) => {
    // simulate slow call
    setTimeout(() => resolve({ receiptId: "rcpt_1" }), 120);
  });
}

export async function createOrderAsync(body: unknown): ResultAsync<{ orderId: OrderId }, AppError> {
  // TODO:
  // 1) parseCreateOrderInput (sync Result) -> okAsync/errAsync
  // 2) andThenAsync getUserAsync
  // 3) andThenAsync charge (wrap withTimeout + fromPromise)
  // 4) map to orderId (makeOrderId)
  return errAsync({ type: "conflict", message: "TODO" });
}

// ---------- Demo ----------
async function demo() {
  const good = { userId: "usr_1a2b3c4d5e6f", amount: 100 };
  const notFound = { userId: "usr_000000000000", amount: 100 };

  console.log("good:", await createOrderAsync(good));
  console.log("notFound:", await createOrderAsync(notFound));

  // Timeout scenario (payment takes 120ms; set timeout lower to force)
  // You should implement this in createOrderAsync with withTimeout(..., 50)
  console.log("timeout:", await createOrderAsync(good));
}

demo().catch((e) => console.error("UNEXPECTED THROW:", e));
```

---

## Required behaviors

Bạn hoàn thành sao cho:

1. `createOrderAsync(good)` → ok với `orderId`
2. `createOrderAsync(notFound)` → err `{type:"not_found"...}`
3. Payment timeout → err `{type:"timeout", operation:"charge", ms: 50}` (hoặc ms bạn chọn)
4. `demo()` không được in ra `UNEXPECTED THROW`

---

## Checks (Definition of Done)

* Không còn `any` (xóa các `as any` placeholder trong TODO)
* `fromPromise` là nơi duy nhất dùng try/catch
* `withTimeout` cleanup timer đúng (không leak)
* `andThenAsync` cho phép compose pipeline sạch

---

## Stretch (đúng kiểu production)

1. **Deadline propagation**: thay `ms` bằng `{ deadlineMs: number }` và trừ dần qua từng call
2. **AbortController**: `withTimeout` abort được (nối sang Kata 11–14)
3. **Parallel combineAsync**: `combineAsync(ra1, ra2)` chạy song song, fail-fast