# Kata 35 — Read Model Projection: build projection từ event stream

## Goal

1. Xây dựng **Read Model** (projection) từ event stream:

* input: sequence events (`order.created`, `payment.captured`, `order.cancelled`)
* output: `OrderSummary` (read-optimized)

2. Projector phải có:

* **idempotency** (event đến trùng không double-apply)
* **checkpoint** (resume từ vị trí đã xử lý)
* **replay** (xoá read model → rebuild từ đầu)

3. Có tests chứng minh:

* projection đúng
* duplicate delivery không sai
* crash giữa chừng → resume đúng
* replay rebuild ra cùng kết quả

---

## Constraints

* ✅ Projection logic phải **pure-ish**: event in → read model update.
* ✅ Không query write-model để “bù” dữ liệu (projection phải sống bằng events).
* ✅ Có `checkpoint` lưu `lastProcessedSeq` (hoặc offset).
* ✅ Delivery có thể at-least-once + out-of-order (stretch).
* ❌ Không “update read model trực tiếp trong write transaction” (đó là coupling sai; projection là async).

---

## Context (event types)

Bạn có stream events có thứ tự (seq tăng dần):

1. `order.created` `{ orderId, userId, total }`
2. `payment.captured` `{ orderId, amount }`
3. `order.cancelled` `{ orderId, reason }`

Read model cần phục vụ UI:
`OrderSummary`:

* `orderId`
* `userId`
* `total`
* `paidAmount`
* `status`: `PENDING | PAID | CANCELLED`
* `lastEventSeq`
* `updatedAt`

---

## Deliverables

`kata-35/`

* `src/events.ts` (event schema + seq)
* `src/eventStore.ts` (append + readFromSeq)
* `src/readDb.ts` (projection state + checkpoint + inbox)
* `src/projector.ts` (applyEvent + runOnce + replay)
* `test/kata35.test.ts` (projection, duplicate, crash resume, replay)

---

## “Nhìn vào làm được” table

| Phần | Bạn làm gì         | Output                                  | Checks              |
| ---- | ------------------ | --------------------------------------- | ------------------- |
| 1    | Event store có seq | `append`, `readFrom(seq)`               | deterministic order |
| 2    | Read DB            | `orderSummaries`, `checkpoint`, `inbox` | resume + dedup      |
| 3    | Projector          | apply event + save checkpoint           | correct state       |
| 4    | Crash sim          | stop mid-batch                          | resume correct      |
| 5    | Replay             | reset read DB, run full                 | same result         |

---

## Starter Skeleton (điền TODO)

### `src/events.ts`

```ts
export type EventEnvelope<T> = {
  seq: number;        // monotonically increasing
  eventId: string;    // dedup key
  type: string;
  payload: T;
  occurredAt: number;
};

export type OrderCreated = { orderId: string; userId: string; total: number };
export type PaymentCaptured = { orderId: string; amount: number };
export type OrderCancelled = { orderId: string; reason: string };

export type DomainEvent =
  | EventEnvelope<OrderCreated> & { type: "order.created" }
  | EventEnvelope<PaymentCaptured> & { type: "payment.captured" }
  | EventEnvelope<OrderCancelled> & { type: "order.cancelled" };
```

### `src/eventStore.ts`

```ts
import type { DomainEvent } from "./events";

export class InMemoryEventStore {
  private events: DomainEvent[] = [];
  private nextSeq = 1;

  append(e: Omit<DomainEvent, "seq">): DomainEvent {
    const evt = { ...e, seq: this.nextSeq++ } as DomainEvent;
    this.events.push(evt);
    return evt;
  }

  readFromSeq(seqExclusive: number, limit: number): DomainEvent[] {
    // return events with seq > seqExclusive
    return this.events
      .filter((e) => e.seq > seqExclusive)
      .sort((a, b) => a.seq - b.seq)
      .slice(0, limit)
      .map((e) => ({ ...e, payload: { ...(e as any).payload } } as DomainEvent));
  }

  all(): DomainEvent[] {
    return this.readFromSeq(0, Number.MAX_SAFE_INTEGER);
  }
}
```

### `src/readDb.ts`

```ts
export type OrderStatus = "PENDING" | "PAID" | "CANCELLED";

export type OrderSummary = {
  orderId: string;
  userId: string;
  total: number;
  paidAmount: number;
  status: OrderStatus;
  lastEventSeq: number;
  updatedAt: number;
};

// Read DB state (projection store)
export class ReadDatabase {
  orderSummaries = new Map<string, OrderSummary>();

  // checkpoint (last processed seq)
  checkpointSeq = 0;

  // inbox (dedup processed eventId)
  inbox = new Set<string>();

  resetAll() {
    this.orderSummaries.clear();
    this.inbox.clear();
    this.checkpointSeq = 0;
  }
}
```

### `src/projector.ts`

```ts
import type { DomainEvent } from "./events";
import type { ReadDatabase, OrderSummary } from "./readDb";
import type { InMemoryEventStore } from "./eventStore";

export type ProjectorOptions = {
  batchSize: number;
};

export function applyEvent(db: ReadDatabase, evt: DomainEvent): void {
  // Idempotency: skip if already processed
  if (db.inbox.has(evt.eventId)) return;

  const now = evt.occurredAt;

  switch (evt.type) {
    case "order.created": {
      const { orderId, userId, total } = evt.payload;
      const existing = db.orderSummaries.get(orderId);

      // TODO:
      // - create if not exist
      // - if exists, decide policy (usually ignore or update)
      // - status default PENDING, paidAmount 0
      const next: OrderSummary = {
        orderId,
        userId,
        total,
        paidAmount: 0,
        status: "PENDING",
        lastEventSeq: evt.seq,
        updatedAt: now,
      };
      db.orderSummaries.set(orderId, next);
      break;
    }

    case "payment.captured": {
      const { orderId, amount } = evt.payload;
      const cur = db.orderSummaries.get(orderId);
      if (!cur) {
        // TODO: decide policy:
        // - create "stub" and fill later? or ignore? or throw?
        // For kata: create stub with unknown userId/total = 0 is dangerous.
        // Better: throw to force ordering assumptions (then add stretch for out-of-order).
        throw new Error("PROJECTION_MISSING_ORDER");
      }

      const paid = cur.paidAmount + amount;
      const status = paid >= cur.total ? "PAID" : cur.status;

      db.orderSummaries.set(orderId, {
        ...cur,
        paidAmount: paid,
        status,
        lastEventSeq: evt.seq,
        updatedAt: now,
      });
      break;
    }

    case "order.cancelled": {
      const { orderId } = evt.payload;
      const cur = db.orderSummaries.get(orderId);
      if (!cur) throw new Error("PROJECTION_MISSING_ORDER");

      db.orderSummaries.set(orderId, {
        ...cur,
        status: "CANCELLED",
        lastEventSeq: evt.seq,
        updatedAt: now,
      });
      break;
    }
  }

  // Mark processed
  db.inbox.add(evt.eventId);
  // Update checkpoint to seq (monotonic stream)
  db.checkpointSeq = Math.max(db.checkpointSeq, evt.seq);
}

export async function runProjectionOnce(
  store: InMemoryEventStore,
  db: ReadDatabase,
  opts: ProjectorOptions,
  crashAfterN?: number // for tests: simulate crash mid-batch
): Promise<number> {
  const batch = store.readFromSeq(db.checkpointSeq, opts.batchSize);
  let processed = 0;

  for (const evt of batch) {
    applyEvent(db, evt);
    processed++;

    if (crashAfterN && processed >= crashAfterN) {
      throw new Error("SIMULATED_CRASH");
    }
  }

  return processed;
}

export async function runUntilCaughtUp(store: InMemoryEventStore, db: ReadDatabase, opts: ProjectorOptions) {
  while (true) {
    const n = await runProjectionOnce(store, db, opts);
    if (n === 0) break;
  }
}

export async function replayProjection(store: InMemoryEventStore, db: ReadDatabase, opts: ProjectorOptions) {
  db.resetAll();
  await runUntilCaughtUp(store, db, opts);
}
```

---

## Tests (vitest) — prove projection correctness

### `test/kata35.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { InMemoryEventStore } from "../src/eventStore";
import { ReadDatabase } from "../src/readDb";
import { runUntilCaughtUp, runProjectionOnce, replayProjection } from "../src/projector";

function seed(store: InMemoryEventStore) {
  store.append({
    eventId: "e1",
    type: "order.created",
    payload: { orderId: "ord_1", userId: "usr_1", total: 100 },
    occurredAt: 1,
  } as any);

  store.append({
    eventId: "e2",
    type: "payment.captured",
    payload: { orderId: "ord_1", amount: 40 },
    occurredAt: 2,
  } as any);

  store.append({
    eventId: "e3",
    type: "payment.captured",
    payload: { orderId: "ord_1", amount: 60 },
    occurredAt: 3,
  } as any);

  store.append({
    eventId: "e4",
    type: "order.created",
    payload: { orderId: "ord_2", userId: "usr_9", total: 50 },
    occurredAt: 4,
  } as any);

  store.append({
    eventId: "e5",
    type: "order.cancelled",
    payload: { orderId: "ord_2", reason: "user_request" },
    occurredAt: 5,
  } as any);
}

describe("kata35 - read model projection", () => {
  it("builds correct projection from event stream", async () => {
    const store = new InMemoryEventStore();
    seed(store);

    const db = new ReadDatabase();
    await runUntilCaughtUp(store, db, { batchSize: 10 });

    const o1 = db.orderSummaries.get("ord_1")!;
    expect(o1.status).toBe("PAID");
    expect(o1.paidAmount).toBe(100);
    expect(o1.total).toBe(100);

    const o2 = db.orderSummaries.get("ord_2")!;
    expect(o2.status).toBe("CANCELLED");
    expect(o2.total).toBe(50);

    expect(db.checkpointSeq).toBe(5);
  });

  it("is idempotent: duplicate delivery does not change result", async () => {
    const store = new InMemoryEventStore();
    seed(store);

    const db = new ReadDatabase();
    await runUntilCaughtUp(store, db, { batchSize: 10 });

    const before = JSON.stringify([...db.orderSummaries.entries()]);

    // Simulate duplicates by replaying the SAME stream again without reset.
    await runUntilCaughtUp(store, db, { batchSize: 10 });

    const after = JSON.stringify([...db.orderSummaries.entries()]);
    expect(after).toBe(before);
    expect(db.inbox.size).toBe(5);
  });

  it("crash mid-batch can resume using checkpoint + inbox", async () => {
    const store = new InMemoryEventStore();
    seed(store);

    const db = new ReadDatabase();

    // process first 2 events then crash
    await expect(runProjectionOnce(store, db, { batchSize: 10 }, 2)).rejects.toThrow("SIMULATED_CRASH");

    // resume
    await runUntilCaughtUp(store, db, { batchSize: 10 });

    const o1 = db.orderSummaries.get("ord_1")!;
    expect(o1.status).toBe("PAID");
    expect(o1.paidAmount).toBe(100);
    expect(db.checkpointSeq).toBe(5);
  });

  it("replay rebuilds same read model from scratch", async () => {
    const store = new InMemoryEventStore();
    seed(store);

    const db = new ReadDatabase();
    await runUntilCaughtUp(store, db, { batchSize: 2 }); // force multi batches

    const snap1 = JSON.stringify([...db.orderSummaries.entries()]);

    await replayProjection(store, db, { batchSize: 3 });

    const snap2 = JSON.stringify([...db.orderSummaries.entries()]);
    expect(snap2).toBe(snap1);
  });
});
```

---

## TODO / Decisions bạn phải chốt

### 1) Policy khi `order.created` tới lại (duplicate but different payload?)

* Với kata: **ignore** nếu đã tồn tại và `lastEventSeq >= evt.seq`.
* Hoặc: chỉ set nếu record chưa exist.

### 2) Policy khi event out-of-order (stretch)

Hiện `payment.captured` trước `order.created` sẽ throw `PROJECTION_MISSING_ORDER`.
Đúng cho mạch đơn giản “ordered stream”. Stretch sẽ xử lý out-of-order.

---

## Definition of Done (Checks)

Bạn pass kata này khi:

1. Projection output đúng: `ord_1` PAID, `ord_2` CANCELLED.
2. Duplicate delivery không thay đổi state (idempotent).
3. Crash mid-batch rồi resume ra đúng kết quả.
4. Replay rebuild ra y hệt.

---

## Stretch (Senior+)

1. **Out-of-order tolerance**

* tạo `pendingEvents` buffer theo `orderId`, hoặc stub record và reconcile khi `order.created` tới.

2. **Per-aggregate ordering**

* enforce seq per aggregate, drop older seq

3. **Schema evolution**

* version payload; projector handle old/new versions

4. **Poison event**

* nếu projector fail N lần cho 1 event → quarantine/DLQ (nối Kata 88)