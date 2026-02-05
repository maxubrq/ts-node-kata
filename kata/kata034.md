# Kata 34 — Exactly-once Illusion: at-least-once + idempotency

## Goal

1. Chứng minh bằng thực nghiệm (test) rằng **duplicate delivery** xảy ra:

* publish thành công
* nhưng ack/markSent fail/crash
* dispatcher chạy lại ⇒ publish lại ⇒ duplicate

2. Implement **idempotent consumer**:

* consumer xử lý event theo `eventId` (hoặc business key) và **dedup** trong DB
* effect (ví dụ “credit points” / “create invoice”) chỉ xảy ra **một lần** dù event tới nhiều lần

3. Chứng minh “đúng” bằng tests:

* không idempotent ⇒ double effect
* idempotent ⇒ single effect, dù double delivery

---

## Constraints

* ✅ Producer side: Outbox pattern (Kata 33), dispatcher có thể publish trùng.
* ✅ Consumer side: có **Inbox/Dedup store** (DB table) để ghi nhận processed events.
* ✅ Idempotency phải nằm trong **transaction** với side-effect của consumer.
* ✅ Có test mô phỏng crash/duplicate.
* ❌ Không được “dựa vào bus đảm bảo exactly-once”.

---

## Context (scenario demo)

Event: `order.created` (từ producer)

Consumer: `AccountingService` tạo `Invoice` khi nhận `order.created`.

* Nếu event bị deliver 2 lần:

  * naive consumer ⇒ tạo 2 invoice (bug)
  * idempotent consumer ⇒ tạo 1 invoice, lần 2 ignore

---

## Deliverables

`kata-34/`

* `src/db.ts` (DB cho consumer: invoices + inbox/dedup)
* `src/tx.ts` (withTransaction cho consumer DB)
* `src/consumer.ts` (naive + idempotent handler)
* `src/bus.ts` (fake bus deliver duplicates)
* `test/kata34.test.ts` (prove duplicates + prove idempotency)

> Bạn có thể reuse DB snapshot/tx wrapper của Kata 31/33.

---

## “Nhìn vào làm được” table

| Phần | Bạn làm gì                     | Output                                               | Checks               |
| ---- | ------------------------------ | ---------------------------------------------------- | -------------------- |
| 1    | Fake bus deliver at-least-once | deliver duplicate events                             | duplicate observable |
| 2    | Naive consumer                 | creates invoice always                               | double invoice bug   |
| 3    | Inbox table                    | store processed eventId                              | prevents reprocess   |
| 4    | Idempotent consumer            | transactionally: check+insert inbox + create invoice | exactly-once effect  |
| 5    | Tests                          | compare naive vs idempotent                          | prove thesis         |

---

## Starter Skeleton (điền TODO)

### `src/db.ts`

```ts
export type InvoiceRow = {
  invoiceId: string;
  orderId: string;
  amount: number;
};

export type InboxRow = {
  eventId: string;      // dedup key
  receivedAt: number;
};

export type DbState = {
  invoices: Map<string, InvoiceRow>;   // invoiceId -> invoice
  inbox: Map<string, InboxRow>;        // eventId -> processed marker
};

function cloneState(s: DbState): DbState {
  return {
    invoices: new Map([...s.invoices.entries()].map(([k, v]) => [k, { ...v }])),
    inbox: new Map([...s.inbox.entries()].map(([k, v]) => [k, { ...v }])),
  };
}

export class Database {
  private state: DbState;

  constructor(seed?: Partial<DbState>) {
    this.state = {
      invoices: seed?.invoices ?? new Map(),
      inbox: seed?.inbox ?? new Map(),
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

### `src/tx.ts`

```ts
import type { Database, Transaction } from "./db";
import { makeInvoiceRepo, makeInboxRepo, type UnitOfWork } from "./repos";

export async function withTransaction<T>(
  db: Database,
  fn: (tx: Transaction, uow: UnitOfWork) => Promise<T> | T
): Promise<T> {
  const tx = db.beginTransaction();
  const uow: UnitOfWork = {
    tx,
    invoices: makeInvoiceRepo(tx),
    inbox: makeInboxRepo(tx),
  };

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

### `src/repos.ts`

```ts
import type { Transaction, InvoiceRow, InboxRow } from "./db";

export type UnitOfWork = {
  tx: Transaction;
  invoices: InvoiceRepo;
  inbox: InboxRepo;
};

export interface InvoiceRepo {
  insert(row: InvoiceRow): Promise<void>;
  findByOrderId(orderId: string): Promise<InvoiceRow | null>;
  count(): Promise<number>;
}

export interface InboxRepo {
  has(eventId: string): Promise<boolean>;
  insert(row: InboxRow): Promise<void>;
}

export function makeInvoiceRepo(tx: Transaction): InvoiceRepo {
  return {
    async insert(row) {
      const s = tx.state();
      if (s.invoices.has(row.invoiceId)) throw new Error("INVOICE_DUPLICATE");
      s.invoices.set(row.invoiceId, { ...row });
    },
    async findByOrderId(orderId) {
      const s = tx.state();
      for (const inv of s.invoices.values()) {
        if (inv.orderId === orderId) return { ...inv };
      }
      return null;
    },
    async count() {
      return tx.state().invoices.size;
    },
  };
}

export function makeInboxRepo(tx: Transaction): InboxRepo {
  return {
    async has(eventId) {
      return tx.state().inbox.has(eventId);
    },
    async insert(row) {
      const s = tx.state();
      if (s.inbox.has(row.eventId)) throw new Error("INBOX_DUPLICATE");
      s.inbox.set(row.eventId, { ...row });
    },
  };
}
```

### `src/events.ts`

```ts
export type OrderCreatedEvent = {
  eventId: string;
  type: "order.created";
  aggregateId: string; // orderId
  payload: { orderId: string; amount: number };
};

export type DomainEvent = OrderCreatedEvent;
```

### `src/consumer.ts`

```ts
import { randomUUID } from "node:crypto";
import type { Database } from "./db";
import { withTransaction } from "./tx";
import type { DomainEvent } from "./events";

// Naive: always creates invoice => not idempotent
export async function handleEventNaive(db: Database, evt: DomainEvent) {
  if (evt.type !== "order.created") return;

  return withTransaction(db, async (_tx, uow) => {
    await uow.invoices.insert({
      invoiceId: "inv_" + randomUUID().slice(0, 12),
      orderId: evt.payload.orderId,
      amount: evt.payload.amount,
    });
  });
}

// Idempotent: inbox + side-effect in SAME TX
export async function handleEventIdempotent(db: Database, evt: DomainEvent) {
  if (evt.type !== "order.created") return;

  return withTransaction(db, async (_tx, uow) => {
    // TODO: if inbox already has evt.eventId => return (no-op)
    // TODO: else insert inbox marker then create invoice
    throw new Error("TODO");
  });
}
```

### `src/bus.ts`

```ts
import type { DomainEvent } from "./events";

export class AtLeastOnceBus {
  private consumers: Array<(evt: DomainEvent) => Promise<void> | void> = [];

  subscribe(fn: (evt: DomainEvent) => Promise<void> | void) {
    this.consumers.push(fn);
  }

  async publish(evt: DomainEvent) {
    // Deliver at-least-once (caller may re-publish same event)
    for (const c of this.consumers) await c(evt);
  }

  async publishDuplicate(evt: DomainEvent, times: number) {
    for (let i = 0; i < times; i++) await this.publish(evt);
  }
}
```

---

## TODO bắt buộc (rõ ràng)

Trong `handleEventIdempotent`:

1. `if (await uow.inbox.has(evt.eventId)) return;`
2. `await uow.inbox.insert({ eventId: evt.eventId, receivedAt: Date.now() });`
3. tạo invoice như naive
4. đảm bảo 2) và 3) cùng transaction

---

## Tests (vitest) — prove thesis

### `test/kata34.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { Database } from "../src/db";
import { AtLeastOnceBus } from "../src/bus";
import { handleEventNaive, handleEventIdempotent } from "../src/consumer";
import type { DomainEvent } from "../src/events";

function mkOrderCreated(eventId: string, orderId: string, amount: number): DomainEvent {
  return {
    eventId,
    type: "order.created",
    aggregateId: orderId,
    payload: { orderId, amount },
  };
}

describe("kata34 - exactly once illusion", () => {
  it("naive consumer: duplicate delivery causes double side-effect", async () => {
    const db = new Database();
    const bus = new AtLeastOnceBus();
    bus.subscribe((evt) => handleEventNaive(db, evt));

    const evt = mkOrderCreated("evt_1", "ord_1", 100);

    await bus.publishDuplicate(evt, 2);

    const snap = db.snapshot();
    expect(snap.invoices.size).toBe(2); // ✅ proves exactly-once is illusion
  });

  it("idempotent consumer: duplicate delivery results in single side-effect", async () => {
    const db = new Database();
    const bus = new AtLeastOnceBus();
    bus.subscribe((evt) => handleEventIdempotent(db, evt));

    const evt = mkOrderCreated("evt_1", "ord_1", 100);

    await bus.publishDuplicate(evt, 2);

    const snap = db.snapshot();
    expect(snap.invoices.size).toBe(1);
    expect(snap.inbox.size).toBe(1);
    expect([...snap.inbox.keys()][0]).toBe("evt_1");
  });

  it("idempotent consumer remains correct if crash happens after inserting inbox? (must be transactional)", async () => {
    const db = new Database();

    // Simulate a bug: inbox inserted but invoice creation throws => transaction rollback => inbox not persisted.
    // We'll assert: after failure, retry delivers again and succeeds once.
    const evt = mkOrderCreated("evt_2", "ord_2", 200);

    // First attempt: force throw after inbox insert by temporarily using a patched handler
    const badHandler = async () => {
      // emulate handleEventIdempotent structure but throw before commit
      const { withTransaction } = await import("../src/tx");
      const { makeInvoiceRepo, makeInboxRepo } = await import("../src/repos");

      await withTransaction(db, async (_tx: any, uow: any) => {
        await uow.inbox.insert({ eventId: evt.eventId, receivedAt: Date.now() });
        throw new Error("CRASH_BEFORE_INVOICE");
      });
    };

    await expect(badHandler()).rejects.toThrow("CRASH_BEFORE_INVOICE");

    // After rollback, inbox must NOT contain evt_2
    expect(db.snapshot().inbox.has("evt_2")).toBe(false);

    // Now normal idempotent handler processes event successfully once
    await handleEventIdempotent(db, evt);
    await handleEventIdempotent(db, evt); // duplicate

    expect(db.snapshot().invoices.size).toBe(1);
    expect(db.snapshot().inbox.size).toBe(1);
  });
});
```

> Test #3 ép bạn hiểu một điều: **idempotency marker phải commit cùng side-effect**. Nếu marker commit trước side-effect (hoặc ngược lại) là bạn tự tạo “ghost states”.

---

## Checks (Definition of Done)

Bạn pass kata này khi:

1. Có test chứng minh naive consumer tạo double invoices khi duplicate delivery.
2. Idempotent consumer:

* dùng inbox/dedup table
* side-effect chỉ xảy ra 1 lần
* marker + side-effect nằm trong cùng transaction

3. Crash test chứng minh rollback không để lại inbox marker “mồ côi”.

---

## Stretch (Senior+)

1. **Idempotency key = business key** thay vì eventId
   Ví dụ: `type + aggregateId` để handle trường hợp upstream bug phát 2 eventId khác nhau cho cùng order.

2. **Inbox retention**

* xóa inbox sau N ngày
* hoặc partition theo date (với DB thật)

3. **Consumer concurrency**

* 2 consumer instances cùng nhận event: dùng unique constraint trên inbox.eventId để guarantee.

4. **Negative ack + poison message**

* nếu processing fail quá N lần → DLQ (nối Kata 88 sau này)