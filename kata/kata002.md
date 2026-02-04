# Kata 02 — Discriminated Union FSM (Order Checkout)

## Goal

1. Mô hình hóa một **Finite State Machine** bằng **discriminated unions**
2. Viết `transition(state, event) -> newState` sao cho:

   * chỉ cho phép **transition hợp lệ**
   * sai là fail compile hoặc fail test rõ ràng
3. Dùng **exhaustive check** để đảm bảo:

   * thêm state/event mới mà quên handle → TypeScript báo lỗi

---

## Context

Bạn đang làm checkout flow. Một order đi qua các trạng thái:

* `cart` → `address` → `payment_pending` → `paid`
* Có thể `cancelled` ở nhiều điểm
* Payment có thể `failed` rồi retry

Bạn phải encode logic này sao cho dev khác không “đẩy order sang paid” bằng nhầm code.

---

## Constraints

* ❌ Không dùng `any`
* ❌ Không `as unknown as ...` để lách
* ✅ Dùng discriminated union với field `type`
* ✅ Transition dùng `switch` + `assertNever`
* ✅ Có unit tests cho các đường đi chính

---

## Deliverables

File `kata02.ts` gồm:

1. Types: `State`, `Event`
2. `transition(state, event): State`
3. `assertNever(x: never): never`
4. Compile-time checks (`@ts-expect-error`) + runtime tests (nếu bạn dùng vitest/jest)

---

## State & Event spec

### States

Bạn phải dùng dạng union:

* `Cart`:

  * type: `"cart"`
  * items: `{ sku: string; qty: number }[]`

* `Address`:

  * type: `"address"`
  * items (carry over)
  * shippingAddress: `{ line1: string; city: string; country: string }`

* `PaymentPending`:

  * type: `"payment_pending"`
  * paymentIntentId: string
  * amount: number

* `Paid`:

  * type: `"paid"`
  * receiptId: string
  * paidAt: number (epoch ms)

* `Cancelled`:

  * type: `"cancelled"`
  * reason: `"user" | "out_of_stock" | "fraud"`

* `PaymentFailed`:

  * type: `"payment_failed"`
  * paymentIntentId: string
  * errorCode: `"DECLINED" | "TIMEOUT" | "GATEWAY_DOWN"`

> Lưu ý: `PaymentFailed` **khác** `Cancelled`.

---

### Events

* `AddItem` `{ type:"add_item"; sku; qty }`
* `RemoveItem` `{ type:"remove_item"; sku }`
* `SubmitAddress` `{ type:"submit_address"; address }`
* `StartPayment` `{ type:"start_payment"; paymentIntentId; amount }`
* `PaymentSucceeded` `{ type:"payment_succeeded"; receiptId; paidAt }`
* `PaymentFailed` `{ type:"payment_failed"; errorCode }`
* `RetryPayment` `{ type:"retry_payment"; paymentIntentId; amount }`
* `Cancel` `{ type:"cancel"; reason }`

---

## Transition rules (bảng luật)

Bạn implement đúng các luật này:

### From `cart`

* `add_item` ✅ stays `cart` (update items)
* `remove_item` ✅ stays `cart`
* `submit_address` ✅ → `address`
* `start_payment` ❌ (chưa có address)
* `cancel` ✅ → `cancelled`

### From `address`

* `add_item` ✅ back to `cart` (vì đổi cart phải quay lại bước trước)
* `remove_item` ✅ back to `cart`
* `start_payment` ✅ → `payment_pending`
* `submit_address` ✅ stays `address` (update address)
* `cancel` ✅ → `cancelled`

### From `payment_pending`

* `payment_succeeded` ✅ → `paid`
* `payment_failed` ✅ → `payment_failed` (state)
* `cancel` ✅ → `cancelled`
* `add_item/remove_item/submit_address/start_payment` ❌

### From `payment_failed` (state)

* `retry_payment` ✅ → `payment_pending`
* `cancel` ✅ → `cancelled`
* `payment_succeeded` ❌ (phải qua pending)
* `payment_failed` ✅ stays `payment_failed` (update errorCode)

### From `paid`

* mọi event ❌ (immutable terminal)

### From `cancelled`

* mọi event ❌ (terminal)

---

## Starter skeleton (điền TODO)

```ts
/* kata02.ts */

// ---------- Helpers ----------
export function assertNever(x: never): never {
  throw new Error("Unhandled case: " + JSON.stringify(x));
}

// ---------- Domain types ----------
type Item = { sku: string; qty: number };
type Address = { line1: string; city: string; country: string };

type CartState = { type: "cart"; items: Item[] };
type AddressState = { type: "address"; items: Item[]; shippingAddress: Address };
type PaymentPendingState = {
  type: "payment_pending";
  items: Item[];
  shippingAddress: Address;
  paymentIntentId: string;
  amount: number;
};
type PaidState = {
  type: "paid";
  items: Item[];
  shippingAddress: Address;
  receiptId: string;
  paidAt: number;
};
type CancelledState = { type: "cancelled"; reason: "user" | "out_of_stock" | "fraud" };
type PaymentFailedState = {
  type: "payment_failed";
  items: Item[];
  shippingAddress: Address;
  paymentIntentId: string;
  errorCode: "DECLINED" | "TIMEOUT" | "GATEWAY_DOWN";
};

export type State =
  | CartState
  | AddressState
  | PaymentPendingState
  | PaymentFailedState
  | PaidState
  | CancelledState;

// ---------- Events ----------
type AddItem = { type: "add_item"; sku: string; qty: number };
type RemoveItem = { type: "remove_item"; sku: string };
type SubmitAddress = { type: "submit_address"; address: Address };
type StartPayment = { type: "start_payment"; paymentIntentId: string; amount: number };
type PaymentSucceeded = { type: "payment_succeeded"; receiptId: string; paidAt: number };
type PaymentFailed = { type: "payment_failed"; errorCode: "DECLINED" | "TIMEOUT" | "GATEWAY_DOWN" };
type RetryPayment = { type: "retry_payment"; paymentIntentId: string; amount: number };
type Cancel = { type: "cancel"; reason: "user" | "out_of_stock" | "fraud" };

export type Event =
  | AddItem
  | RemoveItem
  | SubmitAddress
  | StartPayment
  | PaymentSucceeded
  | PaymentFailed
  | RetryPayment
  | Cancel;

// ---------- TODO: transition ----------
export function transition(state: State, event: Event): State {
  switch (state.type) {
    case "cart": {
      // TODO handle allowed events for cart
      // Use switch(event.type) with assertNever for exhaustiveness
      return state;
    }

    case "address": {
      // TODO
      return state;
    }

    case "payment_pending": {
      // TODO
      return state;
    }

    case "payment_failed": {
      // TODO
      return state;
    }

    case "paid": {
      // TODO: all events invalid
      return state;
    }

    case "cancelled": {
      // TODO: all events invalid
      return state;
    }

    default:
      return assertNever(state);
  }
}
```

---

## Compile-time checks (bắt buộc có)

Bạn thêm cuối file:

```ts
const init: State = { type: "cart", items: [] };

const s1 = transition(init, { type: "add_item", sku: "A", qty: 1 });
const s2 = transition(s1, { type: "submit_address", address: { line1: "x", city: "hcm", country: "vn" } });
const s3 = transition(s2, { type: "start_payment", paymentIntentId: "pi_1", amount: 100 });
const s4 = transition(s3, { type: "payment_failed", errorCode: "DECLINED" });
const s5 = transition(s4, { type: "retry_payment", paymentIntentId: "pi_2", amount: 100 });
const s6 = transition(s5, { type: "payment_succeeded", receiptId: "r1", paidAt: Date.now() });

// @ts-expect-error - start_payment not allowed from cart
transition(init, { type: "start_payment", paymentIntentId: "pi_x", amount: 100 });

// @ts-expect-error - paid is terminal (your implementation should throw or keep paid, but we want it invalid)
transition(s6, { type: "cancel", reason: "user" });
```

> Gợi ý: bạn sẽ cần quyết định “invalid event” làm gì:

* throw Error (rõ ràng)
* hoặc return state unchanged + log (nhưng bài này ưu tiên throw)

---

## Checks (Definition of Done)

* `transition` dùng **2-level switch** (`state.type` rồi `event.type`)
* Có `assertNever` ở chỗ cần để:

  * thêm event mới mà quên handle → compiler complain (trong một branch)
* Có ít nhất 8 test cases cover:

  * happy path cart→paid
  * cancel ở cart/address/pending/failed
  * failed→retry→pending
  * invalid transitions phải throw

---

## Stretch (nâng cấp “Senior+” thật)

1. **Typed transition table**: tạo `AllowedEventsByState` để compiler giới hạn event theo state (compile-time, không chỉ runtime).
2. `transition<S extends State>(state: S, event: AllowedEvent<S>): State` (event type phụ thuộc state)
3. Generate diagram (Mermaid) từ transition table.