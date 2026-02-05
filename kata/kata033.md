# Kata 33 — Outbox Pattern: DB write + event atomic

## Goal

1. Khi xử lý use-case (ví dụ `placeOrder`), bạn phải:

* **write domain state** (Order/Payment/…) **và**
* **enqueue event** vào **Outbox table**
* **trong cùng transaction**
  => commit thì cả hai cùng tồn tại, rollback thì cả hai biến mất.

2. Tách publish ra khỏi request path:

* một **Outbox Dispatcher** chạy nền (hoặc cron/worker)
* đọc các outbox chưa gửi
* publish sang message bus (fake)
* đánh dấu `SENT` (hoặc `SENDING → SENT`)
* có retry/backoff

3. Có tests chứng minh:

* atomicity (DB + outbox)
* publish xảy ra “ít nhất một lần”
* consumer/publisher side có **idempotency** (dedup theo `eventId`)
* dispatcher crash giữa chừng không làm mất event

---

## Constraints

* ✅ Tuyệt đối không publish event trực tiếp trong transaction rồi mới commit (đó là bug kinh điển).
* ✅ Outbox record phải có `eventId` ổn định (UUID) + `aggregateId` + `type` + `payload`.
* ✅ Dispatcher phải “claim” record (tránh 2 worker gửi trùng) — tối thiểu `SENDING` + optimistic update.
* ✅ Có test mô phỏng dispatcher crash: publish xong nhưng chưa mark SENT, chạy lại không tạo “double effect” (dedup).
* ❌ Không “đặt event vào memory queue” thay outbox.

---

## Context (scenario demo)

Use-case: `placeOrder`

* tạo `Order`
* ghi outbox event: `OrderCreated { orderId, userId, total }`

Dispatcher sẽ publish sang bus `FakeBus.publish(event)`.

---

## Deliverables

`kata-33/`

* `src/db.ts` (DB state + transaction snapshot)
* `src/tx.ts` (withTransaction + UoW)
* `src/repos.ts` (OrderRepo + OutboxRepo)
* `src/service.ts` (placeOrder writes Order + Outbox atomically)
* `src/dispatcher.ts` (poll + claim + publish + mark sent)
* `test/kata33.test.ts` (atomicity + dispatcher + crash + dedup)

---

## “Nhìn vào làm được” table

| Phần | Bạn làm gì       | Output                                               | Checks                          |
| ---- | ---------------- | ---------------------------------------------------- | ------------------------------- |
| 1    | Add Outbox table | `outbox` Map                                         | record persisted only on commit |
| 2    | Outbox repo      | `enqueue`, `listPending`, `markSending`, `markSent`  | claim prevents double-send      |
| 3    | Use-case         | `placeOrder` writes Order + enqueue event in same tx | rollback removes both           |
| 4    | Dispatcher       | poll pending → claim → publish → markSent            | crash-safe + retryable          |
| 5    | Tests            | atomicity + crash + dedup                            | prove correctness               |

---

## Starter Skeleton (điền TODO)

### `src/db.ts`

```ts
export type OrderRow = {
  orderId: string;
  userId: string;
  total: number;
  status: "PENDING" | "CONFIRMED";
};

// Outbox status lifecycle
export type OutboxStatus = "PENDING" | "SENDING" | "SENT" | "FAILED";

export type OutboxRow = {
  eventId: string;          // UUID
  type: string;             // "order.created"
  aggregateId: string;      // orderId
  payload: unknown;         // JSON
  status: OutboxStatus;
  attempts: number;
  nextAttemptAt: number;    // epoch ms, for backoff
  createdAt: number;
  lastError?: string;
};

export type DbState = {
  orders: Map<string, OrderRow>;
  outbox: Map<string, OutboxRow>; // key = eventId
};

function cloneState(s: DbState): DbState {
  return {
    orders: new Map([...s.orders.entries()].map(([k, v]) => [k, { ...v }])),
    outbox: new Map([...s.outbox.entries()].map(([k, v]) => [k, { ...v }])),
  };
}

export class Database {
  private state: DbState;

  constructor(seed?: Partial<DbState>) {
    this.state = {
      orders: seed?.orders ?? new Map(),
      outbox: seed?.outbox ?? new Map(),
    };
  }

  snapshot(): DbState {
    return cloneState(this.state);
  }

  beginTransaction(): Transaction {
    return new Transaction(this);
  }

  _getStateRef(): DbState { return this.state; }
  _setStateRef(next: DbState) { this.state = next; }
}

export class Transaction {
  private active = true;
  private readonly before: DbState;
  private working: DbState;

  constructor(private readonly db: Database) {
    const current = db._getStateRef();
    this.before = cloneState(current);
    this.working = cloneState(current);
  }

  state(): DbState {
    if (!this.active) throw new Error("TX_NOT_ACTIVE");
    return this.working;
  }

  commit() {
    if (!this.active) throw new Error("TX_NOT_ACTIVE");
    this.db._setStateRef(this.working);
    this.active = false;
  }

  rollback() {
    if (!this.active) throw new Error("TX_NOT_ACTIVE");
    this.db._setStateRef(this.before);
    this.active = false;
  }
}
```

### `src/repos.ts`

```ts
import type { Transaction, OrderRow, OutboxRow } from "./db";

export type UnitOfWork = {
  tx: Transaction;
  orders: OrderRepo;
  outbox: OutboxRepo;
};

export interface OrderRepo {
  insert(row: OrderRow): Promise<void>;
  get(orderId: string): Promise<OrderRow | null>;
}

export interface OutboxRepo {
  enqueue(row: Omit<OutboxRow, "status" | "attempts" | "nextAttemptAt" | "createdAt">): Promise<void>;

  // dispatcher operations
  listDuePending(now: number, limit: number): Promise<OutboxRow[]>;
  markSending(eventId: string): Promise<boolean>; // claim; returns false if not claimable
  markSent(eventId: string): Promise<void>;
  markFailed(eventId: string, errMsg: string, now: number): Promise<void>;
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

export function makeOutboxRepo(tx: Transaction): OutboxRepo {
  return {
    async enqueue(row) {
      const s = tx.state();
      if (s.outbox.has(row.eventId)) throw new Error("OUTBOX_DUPLICATE");
      const now = Date.now();
      const full: OutboxRow = {
        ...row,
        status: "PENDING",
        attempts: 0,
        nextAttemptAt: now,
        createdAt: now,
      };
      s.outbox.set(full.eventId, full);
    },

    async listDuePending(now, limit) {
      const s = tx.state();
      const rows = [...s.outbox.values()]
        .filter((r) => r.status === "PENDING" && r.nextAttemptAt <= now)
        .sort((a, b) => a.createdAt - b.createdAt)
        .slice(0, limit)
        .map((r) => ({ ...r }));
      return rows;
    },

    async markSending(eventId) {
      const s = tx.state();
      const r = s.outbox.get(eventId);
      if (!r) throw new Error("OUTBOX_NOT_FOUND");
      if (r.status !== "PENDING") return false;

      // TODO: mark SENDING (claim)
      // increment attempts
      // persist
      throw new Error("TODO");
    },

    async markSent(eventId) {
      const s = tx.state();
      const r = s.outbox.get(eventId);
      if (!r) throw new Error("OUTBOX_NOT_FOUND");
      r.status = "SENT";
      s.outbox.set(eventId, r);
    },

    async markFailed(eventId, errMsg, now) {
      const s = tx.state();
      const r = s.outbox.get(eventId);
      if (!r) throw new Error("OUTBOX_NOT_FOUND");

      // backoff policy: nextAttemptAt = now + min(1000 * 2^attempts, 60_000)
      const delay = Math.min(1000 * Math.pow(2, r.attempts), 60_000);
      r.status = "PENDING";          // keep retrying as PENDING (simple)
      r.nextAttemptAt = now + delay;
      r.lastError = errMsg.slice(0, 200);
      s.outbox.set(eventId, r);
    },
  };
}
```

### `src/tx.ts`

```ts
import type { Database, Transaction } from "./db";
import { makeOrderRepo, makeOutboxRepo, type UnitOfWork } from "./repos";

export async function withTransaction<T>(
  db: Database,
  fn: (tx: Transaction, uow: UnitOfWork) => Promise<T> | T
): Promise<T> {
  const tx = db.beginTransaction();
  const uow: UnitOfWork = { tx, orders: makeOrderRepo(tx), outbox: makeOutboxRepo(tx) };

  try {
    const out = await fn(tx, uow);
    tx.commit();
    return out;
  } catch (e) {
    try { tx.rollback(); } catch {}
    throw e;
  }
}
```

### `src/service.ts`

```ts
import { randomUUID } from "node:crypto";
import type { Database } from "./db";
import { withTransaction } from "./tx";

export async function placeOrder(db: Database, input: { userId: string; total: number }) {
  return withTransaction(db, async (_tx, uow) => {
    const orderId = "ord_" + randomUUID().slice(0, 12);

    await uow.orders.insert({
      orderId,
      userId: input.userId,
      total: input.total,
      status: "PENDING",
    });

    // ✅ enqueue event IN SAME TX
    await uow.outbox.enqueue({
      eventId: randomUUID(),
      type: "order.created",
      aggregateId: orderId,
      payload: { orderId, userId: input.userId, total: input.total },
    });

    return { orderId };
  });
}
```

### `src/dispatcher.ts`

```ts
import type { Database, OutboxRow } from "./db";
import { withTransaction } from "./tx";

// Fake bus interface
export interface Bus {
  publish(evt: Pick<OutboxRow, "eventId" | "type" | "payload" | "aggregateId">): Promise<void>;
}

// Simple idempotent bus wrapper (dedup by eventId)
export class DedupBus implements Bus {
  private seen = new Set<string>();
  constructor(private readonly inner: Bus) {}

  async publish(evt: any) {
    if (this.seen.has(evt.eventId)) return;
    await this.inner.publish(evt);
    this.seen.add(evt.eventId);
  }
}

export type DispatchOptions = {
  batchSize: number;  // e.g. 50
  now: () => number;  // injectable clock for tests
};

// One “tick” of dispatcher (poll once)
export async function dispatchOutboxOnce(db: Database, bus: Bus, opts: DispatchOptions) {
  const now = opts.now();

  // 1) load due events (transactional read)
  const due = await withTransaction(db, async (_tx, uow) => {
    return uow.outbox.listDuePending(now, opts.batchSize);
  });

  // 2) process each (claim -> publish -> markSent)
  for (const evt of due) {
    // claim in its own tx so multiple workers can compete safely
    const claimed = await withTransaction(db, async (_tx, uow) => {
      return uow.outbox.markSending(evt.eventId);
    });
    if (!claimed) continue;

    try {
      await bus.publish({
        eventId: evt.eventId,
        type: evt.type,
        aggregateId: evt.aggregateId,
        payload: evt.payload,
      });

      await withTransaction(db, async (_tx, uow) => {
        await uow.outbox.markSent(evt.eventId);
      });
    } catch (e) {
      const msg = e instanceof Error ? e.message : String(e);
      await withTransaction(db, async (_tx, uow) => {
        await uow.outbox.markFailed(evt.eventId, msg, now);
      });
    }
  }
}
```

---

## TODO bắt buộc (rõ ràng)

### 1) `markSending(eventId)` phải claim đúng

Trong `repos.ts`:

* chỉ claim khi `status === "PENDING"`
* set:

  * `status = "SENDING"`
  * `attempts += 1`
* return `true` nếu claim thành công, `false` nếu không claimable

Ví dụ:

```ts
if (r.status !== "PENDING") return false;
r.status = "SENDING";
r.attempts += 1;
s.outbox.set(eventId, r);
return true;
```

> Snapshot DB của kata này không có “real concurrency”, nhưng pattern này mô phỏng đúng production (và test sẽ dựa vào nó).

---

## Tests (vitest)

### `test/kata33.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { Database } from "../src/db";
import { placeOrder } from "../src/service";
import { dispatchOutboxOnce, DedupBus, type Bus } from "../src/dispatcher";
import { withTransaction } from "../src/tx";

class RecordingBus implements Bus {
  public published: any[] = [];
  async publish(evt: any) {
    this.published.push(evt);
  }
}

describe("kata33 - outbox pattern", () => {
  it("writes DB + outbox atomically on commit", async () => {
    const db = new Database();
    const { orderId } = await placeOrder(db, { userId: "usr_1", total: 100 });

    const snap = db.snapshot();
    expect(snap.orders.get(orderId)).toBeTruthy();
    expect(snap.outbox.size).toBe(1);

    const evt = [...snap.outbox.values()][0];
    expect(evt.type).toBe("order.created");
    expect((evt.payload as any).orderId).toBe(orderId);
    expect(evt.status).toBe("PENDING");
  });

  it("rollback removes both order and outbox record", async () => {
    const db = new Database();

    await expect(
      withTransaction(db, async (_tx, uow) => {
        await uow.orders.insert({ orderId: "ord_x", userId: "u", total: 1, status: "PENDING" });
        await uow.outbox.enqueue({
          eventId: "evt_x",
          type: "order.created",
          aggregateId: "ord_x",
          payload: { orderId: "ord_x" },
        });
        throw new Error("BOOM");
      })
    ).rejects.toThrow("BOOM");

    const snap = db.snapshot();
    expect(snap.orders.size).toBe(0);
    expect(snap.outbox.size).toBe(0);
  });

  it("dispatcher publishes pending outbox and marks SENT", async () => {
    const db = new Database();
    await placeOrder(db, { userId: "usr_1", total: 100 });

    const bus = new RecordingBus();
    await dispatchOutboxOnce(db, bus, { batchSize: 10, now: () => Date.now() });

    expect(bus.published.length).toBe(1);

    const snap = db.snapshot();
    const evt = [...snap.outbox.values()][0];
    expect(evt.status).toBe("SENT");
  });

  it("crash between publish and markSent does not cause double-effects with dedup", async () => {
    const db = new Database();
    await placeOrder(db, { userId: "usr_1", total: 100 });

    const inner = new RecordingBus();
    const bus = new DedupBus(inner);

    // 1) simulate: publish succeeds but markSent never happens
    // We'll do it by calling publish manually and leaving outbox as PENDING
    const evt1 = [...db.snapshot().outbox.values()][0];
    await bus.publish({ eventId: evt1.eventId, type: evt1.type, aggregateId: evt1.aggregateId, payload: evt1.payload });

    // outbox still PENDING => dispatcher will try again
    await dispatchOutboxOnce(db, bus, { batchSize: 10, now: () => Date.now() });

    // inner bus should only see one publish (dedup)
    expect(inner.published.length).toBe(1);

    // after dispatcher, should be SENT
    const evt = [...db.snapshot().outbox.values()][0];
    expect(evt.status).toBe("SENT");
  });
});
```

---

## Definition of Done (Checks)

Bạn pass kata này khi:

1. Use-case commit ⇒ vừa có `orders` record vừa có `outbox` record.
2. Use-case throw ⇒ rollback ⇒ không còn cả 2.
3. Dispatcher chạy ⇒ publish đúng số event và `status` chuyển `PENDING → SENDING → SENT`.
4. Crash-simulated ⇒ không tạo double-effect nhờ dedup theo `eventId`.

---

## Stretch (Senior+)

1. **Claim timeout / stuck SENDING**

* thêm `lockedAt`, `lockedBy`
* nếu `SENDING` quá lâu ⇒ release về `PENDING`

2. **Outbox schema evolution**

* thêm `schemaVersion` vào event payload
* backward/forward compatible (nối Kata 89 sau này)

3. **Exactly-once illusion** (nối Kata 34)

* chứng minh hệ chỉ đạt at-least-once, và nơi đảm bảo đúng là consumer idempotency

4. **Batch publish + partial failure**

* publish batch, markSent từng event; failure giữa batch vẫn retry phần còn lại