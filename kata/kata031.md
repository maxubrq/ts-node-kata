# Kata 31 — Transaction Wrapper: Unit of Work với rollback

## Goal

1. Viết **Transaction Wrapper** `withTransaction(...)`:

* mở transaction
* chạy callback
* commit nếu OK
* rollback nếu lỗi
* đảm bảo **cleanup** (release/close) dù thành công hay thất bại

2. Thiết kế **Unit of Work** để:

* repos dùng cùng một transaction context
* code application/service không biết DB cụ thể
* test được behavior commit/rollback

3. Có test chứng minh:

* success ⇒ commit, changes persist
* error ⇒ rollback, changes revert
* nested transaction: policy rõ (reuse outer tx hoặc create savepoint)
* rollback cũng phải xảy ra khi callback throw error sync/async

---

## Constraints

* ✅ Không được “repo tự start transaction”.
* ✅ Transaction scope phải nằm ở **application layer** (service boundary).
* ✅ Repo methods phải nhận `tx` (hoặc `uow`) và chỉ dùng trong scope đó.
* ✅ Có unit tests bằng **in-memory fake DB** (đủ để mô phỏng commit/rollback).
* ❌ Không được dùng global mutable state để “nhớ transaction hiện tại”.
* ✅ Có “error taxonomy” tối thiểu: lỗi domain vs lỗi infra (optional).

---

## Context (scenario demo)

Bạn có nghiệp vụ `placeOrder`:

* Trừ tồn kho cho từng item
* Tạo order record
* Tạo payment record (pending)
* Nếu bất kỳ bước nào fail ⇒ **rollback tất cả**

### Entities (tối thiểu)

* `Inventory(sku, qty)`
* `Order(orderId, userId, total, status)`
* `Payment(paymentId, orderId, status)`

---

## Deliverables

`kata-31/`

* `src/db.ts` (fake transactional DB)
* `src/tx.ts` (Transaction wrapper + UoW)
* `src/repos.ts` (InventoryRepo, OrderRepo, PaymentRepo)
* `src/service.ts` (placeOrder use-case)
* `test/kata31.test.ts` (commit/rollback/nested)

---

## Bảng “nhìn vào là làm được” (Plan → Task → Checks)

| Phần            | Bạn làm gì                                   | Output                    | Checks pass/fail      |
| --------------- | -------------------------------------------- | ------------------------- | --------------------- |
| 1. Fake DB      | Tạo in-memory DB có **transaction snapshot** | `Database`, `Transaction` | rollback restore đúng |
| 2. Tx wrapper   | `withTransaction(db, fn)`                    | auto commit/rollback      | fail thì rollback     |
| 3. Unit of Work | `UnitOfWork` chứa repos bound với tx         | `uow.inventory...`        | repos share same tx   |
| 4. Use-case     | `placeOrder(uowFactory, input)`              | order + payment created   | atomicity             |
| 5. Tests        | test success & failure                       | vitest                    | chứng minh behavior   |

---

## Starter Skeleton (điền TODO)

### `src/db.ts` — Fake transactional DB (snapshot-based)

```ts
export type InventoryRow = { sku: string; qty: number };
export type OrderRow = { orderId: string; userId: string; total: number; status: "PENDING" | "CONFIRMED" };
export type PaymentRow = { paymentId: string; orderId: string; status: "PENDING" | "CAPTURED" };

export type DbState = {
  inventory: Map<string, InventoryRow>;
  orders: Map<string, OrderRow>;
  payments: Map<string, PaymentRow>;
};

function cloneState(s: DbState): DbState {
  // deep-ish clone (maps + objects)
  return {
    inventory: new Map([...s.inventory.entries()].map(([k, v]) => [k, { ...v }])),
    orders: new Map([...s.orders.entries()].map(([k, v]) => [k, { ...v }])),
    payments: new Map([...s.payments.entries()].map(([k, v]) => [k, { ...v }])),
  };
}

export class Database {
  private state: DbState;

  constructor(seed?: Partial<DbState>) {
    this.state = {
      inventory: seed?.inventory ?? new Map(),
      orders: seed?.orders ?? new Map(),
      payments: seed?.payments ?? new Map(),
    };
  }

  // Non-transactional read (for test/debug)
  snapshot(): DbState {
    return cloneState(this.state);
  }

  beginTransaction(): Transaction {
    return new Transaction(this);
  }

  // internal access for tx
  _getStateRef(): DbState {
    return this.state;
  }
  _setStateRef(next: DbState) {
    this.state = next;
  }
}

export class Transaction {
  private active = true;
  private readonly before: DbState;
  private working: DbState;

  // instrumentation for tests
  public commits = 0;
  public rollbacks = 0;

  constructor(private readonly db: Database) {
    const current = db._getStateRef();
    this.before = cloneState(current);
    this.working = cloneState(current);
  }

  assertActive() {
    if (!this.active) throw new Error("TX_NOT_ACTIVE");
  }

  // Provide access to the working state (repos will use this)
  state(): DbState {
    this.assertActive();
    return this.working;
  }

  commit() {
    this.assertActive();
    this.db._setStateRef(this.working);
    this.active = false;
    this.commits++;
  }

  rollback() {
    this.assertActive();
    this.db._setStateRef(this.before);
    this.active = false;
    this.rollbacks++;
  }
}
```

---

### `src/tx.ts` — Transaction wrapper + Unit of Work

```ts
import type { Database, Transaction } from "./db";
import { makeInventoryRepo, makeOrderRepo, makePaymentRepo, type UnitOfWork } from "./repos";

export type TxOptions = {
  // nested handling strategy
  // "reuse": if already in tx, reuse it (no new commit/rollback)
  // "savepoint": optional stretch (not required for snapshot DB)
  nested?: "reuse";
};

export async function withTransaction<T>(
  db: Database,
  fn: (tx: Transaction, uow: UnitOfWork) => Promise<T> | T,
  opts: TxOptions = { nested: "reuse" }
): Promise<T> {
  // For kata simplicity: no ALS; caller passes tx explicitly.
  const tx = db.beginTransaction();
  const uow: UnitOfWork = {
    tx,
    inventory: makeInventoryRepo(tx),
    orders: makeOrderRepo(tx),
    payments: makePaymentRepo(tx),
  };

  try {
    const out = await fn(tx, uow);
    tx.commit();
    return out;
  } catch (e) {
    // rollback must happen even if commit would have happened later
    try { tx.rollback(); } catch { /* ignore */ }
    throw e;
  }
}
```

> Nested transactions (policy) mình để ở phần Stretch; bản core này đủ luyện commit/rollback đúng.

---

### `src/repos.ts` — Repos bound to tx

```ts
import type { Transaction, InventoryRow, OrderRow, PaymentRow } from "./db";

export type UnitOfWork = {
  tx: Transaction;
  inventory: InventoryRepo;
  orders: OrderRepo;
  payments: PaymentRepo;
};

export interface InventoryRepo {
  get(sku: string): Promise<InventoryRow | null>;
  decrease(sku: string, by: number): Promise<void>;
}

export interface OrderRepo {
  insert(row: OrderRow): Promise<void>;
  get(orderId: string): Promise<OrderRow | null>;
}

export interface PaymentRepo {
  insert(row: PaymentRow): Promise<void>;
  get(paymentId: string): Promise<PaymentRow | null>;
}

export function makeInventoryRepo(tx: Transaction): InventoryRepo {
  return {
    async get(sku) {
      const inv = tx.state().inventory.get(sku);
      return inv ? { ...inv } : null;
    },
    async decrease(sku, by) {
      const s = tx.state();
      const row = s.inventory.get(sku);
      if (!row) throw new Error("INVENTORY_NOT_FOUND");
      if (row.qty < by) throw new Error("INSUFFICIENT_STOCK");
      row.qty -= by; // mutate working state
      s.inventory.set(sku, row);
    },
  };
}

export function makeOrderRepo(tx: Transaction): OrderRepo {
  return {
    async insert(row) {
      const s = tx.state();
      if (s.orders.has(row.orderId)) throw new Error("ORDER_DUPLICATE");
      s.orders.set(row.orderId, { ...row });
    },
    async get(orderId) {
      const row = tx.state().orders.get(orderId);
      return row ? { ...row } : null;
    },
  };
}

export function makePaymentRepo(tx: Transaction): PaymentRepo {
  return {
    async insert(row) {
      const s = tx.state();
      if (s.payments.has(row.paymentId)) throw new Error("PAYMENT_DUPLICATE");
      s.payments.set(row.paymentId, { ...row });
    },
    async get(paymentId) {
      const row = tx.state().payments.get(paymentId);
      return row ? { ...row } : null;
    },
  };
}
```

---

### `src/service.ts` — Use-case atomic

```ts
import type { Database } from "./db";
import { withTransaction } from "./tx";

export type PlaceOrderInput = {
  userId: string;
  items: Array<{ sku: string; qty: number; price: number }>;
};

function genId(prefix: string) {
  return `${prefix}_${Math.random().toString(16).slice(2, 14)}`;
}

export async function placeOrder(db: Database, input: PlaceOrderInput) {
  return withTransaction(db, async (_tx, uow) => {
    // 1) decrease inventory
    for (const it of input.items) {
      await uow.inventory.decrease(it.sku, it.qty);
    }

    // 2) create order
    const orderId = genId("ord");
    const total = input.items.reduce((s, it) => s + it.qty * it.price, 0);

    await uow.orders.insert({
      orderId,
      userId: input.userId,
      total,
      status: "PENDING",
    });

    // 3) create payment pending
    const paymentId = genId("pay");
    await uow.payments.insert({
      paymentId,
      orderId,
      status: "PENDING",
    });

    return { orderId, paymentId, total };
  });
}
```

---

## Tests (vitest) — commit/rollback phải chứng minh

### `test/kata31.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { Database } from "../src/db";
import { placeOrder } from "../src/service";
import { withTransaction } from "../src/tx";

describe("kata31 - transaction wrapper", () => {
  it("commits on success", async () => {
    const db = new Database({
      inventory: new Map([["SKU_1", { sku: "SKU_1", qty: 10 }]]),
      orders: new Map(),
      payments: new Map(),
    });

    const out = await placeOrder(db, {
      userId: "usr_1",
      items: [{ sku: "SKU_1", qty: 2, price: 100 }],
    });

    const snap = db.snapshot();
    expect(snap.inventory.get("SKU_1")?.qty).toBe(8);
    expect([...snap.orders.keys()].length).toBe(1);
    expect([...snap.payments.keys()].length).toBe(1);
    expect(out.total).toBe(200);
  });

  it("rolls back on failure (insufficient stock)", async () => {
    const db = new Database({
      inventory: new Map([["SKU_1", { sku: "SKU_1", qty: 1 }]]),
      orders: new Map(),
      payments: new Map(),
    });

    await expect(
      placeOrder(db, {
        userId: "usr_1",
        items: [{ sku: "SKU_1", qty: 2, price: 100 }],
      })
    ).rejects.toThrow("INSUFFICIENT_STOCK");

    const snap = db.snapshot();
    expect(snap.inventory.get("SKU_1")?.qty).toBe(1); // reverted
    expect(snap.orders.size).toBe(0);
    expect(snap.payments.size).toBe(0);
  });

  it("rolls back on later failure (after inventory decreased)", async () => {
    const db = new Database({
      inventory: new Map([["SKU_1", { sku: "SKU_1", qty: 10 }]]),
      orders: new Map(),
      payments: new Map(),
    });

    // Inject failure by creating duplicate order id inside tx
    await expect(
      withTransaction(db, async (_tx, uow) => {
        await uow.inventory.decrease("SKU_1", 2);
        await uow.orders.insert({ orderId: "ord_fixed", userId: "u", total: 1, status: "PENDING" });
        // duplicate insert triggers error
        await uow.orders.insert({ orderId: "ord_fixed", userId: "u", total: 1, status: "PENDING" });
      })
    ).rejects.toThrow("ORDER_DUPLICATE");

    const snap = db.snapshot();
    expect(snap.inventory.get("SKU_1")?.qty).toBe(10); // inventory decrease rolled back
    expect(snap.orders.size).toBe(0);
  });
});
```

---

## Checks (Definition of Done)

Bạn pass kata này khi:

1. `withTransaction` **luôn** commit khi không lỗi, rollback khi lỗi (sync/async).
2. Repo methods dùng chung một `tx` (thể hiện qua atomic commit/rollback).
3. Test chứng minh rollback *đúng* ngay cả khi lỗi xảy ra sau vài thao tác đã “mutate working state”.

---

## Stretch (Senior+)

1. **Nested transaction policy**

* `withTransaction(db, fn)` được gọi bên trong `withTransaction`:

  * **reuse outer tx** (recommended): inner không commit/rollback, chỉ throw lên để outer xử lý
  * hoặc **savepoint** (khó hơn): rollback inner nhưng không rollback outer

2. **“Rollback on commit failure”**

* giả lập `tx.commit()` throw → wrapper phải try rollback (best effort) và throw lỗi rõ.

3. **Domain vs Infra errors**

* `INSUFFICIENT_STOCK` là domain error ⇒ 409
* `TX_NOT_ACTIVE` là programmer/infra error ⇒ 500
  (nối lại Kata 26 mapping Problem+JSON)

4. **No-leak design**

* Service chỉ thấy `UnitOfWork` interface, không biết `Transaction` class cụ thể.