# Kata 82 — Idempotent Consumer: at-least-once + exactly-once effects

## Goal

1. Viết consumer xử lý message **at-least-once** (broker có thể giao trùng).
2. Đảm bảo:

   * **Dedup theo idempotency key** (messageId / eventId).
   * **Processing atomic**: “apply side-effect” + “mark processed” không bị lệch.
   * Có **retry** cho lỗi transient, **DLQ** cho poison message.
3. Có test mô phỏng:

   * duplicate delivery,
   * crash giữa chừng,
   * retry,
   * poison message.

---

## Context (mô phỏng production)

Bạn có event `OrderPaid` từ queue. Consumer sẽ “ghi credit vào ledger” (side-effect). Broker **at-least-once** => event có thể tới 2–10 lần.

Nếu bạn không idempotent → **double credit** (toang).

---

## Constraints

* ❌ Không dùng Kafka lib/queue lib thật. Làm **in-memory broker** để test deterministic.
* ✅ Event phải có `eventId` (UUID-like string) và `occurredAt`.
* ✅ Dedup store phải có trạng thái tối thiểu:

  * `processing` (đang xử lý) + `processed` (đã xong),
  * có `startedAt`, `finishedAt`, `attempts`.
* ✅ Xử lý phải theo **transaction boundary** (ở kata này mô phỏng bằng “UnitOfWork” với commit/rollback).
* ✅ Có retry policy (dùng lại tư duy Kata 81, nhưng đơn giản hơn được).
* ✅ Test bắt buộc:

  1. duplicate => ledger chỉ ghi 1 lần,
  2. crash giữa chừng => retry vẫn chỉ ghi 1 lần,
  3. poison => vào DLQ sau N attempts.

---

## Deliverables (cấu trúc)

```
kata82/
  src/
    types.ts
    broker.ts
    store.ts
    uow.ts
    consumer.ts
    handlers.ts
  test/
    consumer.test.ts
```

---

# Starter skeleton (copy là làm)

## 1) `src/types.ts`

```ts
// src/types.ts
export type OrderPaid = {
  type: "OrderPaid";
  eventId: string;        // idempotency key
  occurredAt: number;     // epoch ms
  orderId: string;
  userId: string;
  amountCents: number;
};

export type Event = OrderPaid;

export type ConsumerResult =
  | { ok: true }
  | { ok: false; kind: "transient" | "poison"; error: string };

export type ProcessState =
  | { state: "processing"; startedAt: number; attempts: number; lockToken: string }
  | { state: "processed"; startedAt: number; finishedAt: number; attempts: number };

export type IdempotencyRecord = {
  eventId: string;
  state: ProcessState;
};
```

---

## 2) `src/broker.ts` (at-least-once + duplicates)

```ts
// src/broker.ts
import type { Event } from "./types";

export type BrokerMessage = {
  deliveryId: string; // unique per delivery attempt
  event: Event;
};

export class InMemoryBroker {
  private queue: BrokerMessage[] = [];
  private dlq: BrokerMessage[] = [];

  publish(event: Event, opts?: { duplicates?: number }) {
    const dups = opts?.duplicates ?? 1;
    for (let i = 0; i < dups; i++) {
      this.queue.push({
        deliveryId: `d_${event.eventId}_${i}_${Math.random().toString(16).slice(2)}`,
        event,
      });
    }
  }

  poll(): BrokerMessage | undefined {
    return this.queue.shift();
  }

  ack(_msg: BrokerMessage) {
    // noop: in-memory
  }

  nack(msg: BrokerMessage) {
    // at-least-once: put back to tail
    this.queue.push(msg);
  }

  sendToDLQ(msg: BrokerMessage) {
    this.dlq.push(msg);
  }

  dlqMessages() {
    return [...this.dlq];
  }

  size() {
    return this.queue.length;
  }
}
```

---

## 3) `src/store.ts` (idempotency store + “lock”)

Key part: **claim processing** theo eventId, chống concurrent duplicate.

```ts
// src/store.ts
import type { IdempotencyRecord, ProcessState } from "./types";

export class IdempotencyStore {
  private records = new Map<string, IdempotencyRecord>();

  get(eventId: string): IdempotencyRecord | undefined {
    return this.records.get(eventId);
  }

  /**
   * Try to acquire processing lock for an eventId.
   * Returns:
   * - "already_processed" if done
   * - "locked" if someone else processing (or stale lock policy could be added)
   * - "acquired" with lockToken if caller can proceed
   */
  claim(eventId: string, nowMs: number): { kind: "acquired"; lockToken: string; attempts: number } | { kind: "already_processed" } | { kind: "locked" } {
    const existing = this.records.get(eventId);

    if (!existing) {
      const lockToken = `lock_${Math.random().toString(16).slice(2)}`;
      const state: ProcessState = { state: "processing", startedAt: nowMs, attempts: 1, lockToken };
      this.records.set(eventId, { eventId, state });
      return { kind: "acquired", lockToken, attempts: 1 };
    }

    if (existing.state.state === "processed") return { kind: "already_processed" };

    // existing is processing
    return { kind: "locked" };
  }

  markProcessed(args: { eventId: string; lockToken: string; nowMs: number }) {
    const rec = this.records.get(args.eventId);
    if (!rec) throw new Error("Idempotency record missing");
    if (rec.state.state !== "processing") throw new Error("Not in processing state");
    if (rec.state.lockToken !== args.lockToken) throw new Error("Lock token mismatch");

    const finished: ProcessState = {
      state: "processed",
      startedAt: rec.state.startedAt,
      finishedAt: args.nowMs,
      attempts: rec.state.attempts,
    };
    this.records.set(args.eventId, { eventId: args.eventId, state: finished });
  }

  bumpAttempt(eventId: string, lockToken: string) {
    const rec = this.records.get(eventId);
    if (!rec) throw new Error("Idempotency record missing");
    if (rec.state.state !== "processing") throw new Error("Not in processing state");
    if (rec.state.lockToken !== lockToken) throw new Error("Lock token mismatch");
    rec.state.attempts += 1;
  }
}
```

---

## 4) `src/uow.ts` (transaction boundary giả lập)

Ý tưởng: hoặc **commit cả ledger + markProcessed**, hoặc rollback cả hai.

```ts
// src/uow.ts
export type LedgerEntry = { orderId: string; userId: string; amountCents: number; eventId: string };

export class LedgerDB {
  private entries: LedgerEntry[] = [];
  insert(e: LedgerEntry) {
    this.entries.push(e);
  }
  all() {
    return [...this.entries];
  }
}

export class UnitOfWork {
  private staged: (() => void)[] = [];
  stage(fn: () => void) {
    this.staged.push(fn);
  }
  commit() {
    for (const fn of this.staged) fn();
    this.staged = [];
  }
  rollback() {
    this.staged = [];
  }
}
```

---

## 5) `src/handlers.ts` (business handler)

Handler có thể ném lỗi transient/poison (để test).

```ts
// src/handlers.ts
import type { Event, ConsumerResult } from "./types";
import type { LedgerDB, UnitOfWork } from "./uow";

export type HandlerDeps = {
  ledger: LedgerDB;
  uow: UnitOfWork;
  // For failure injection
  failMode?: "none" | "transient_once" | "poison";
};

export async function handleEvent(event: Event, deps: HandlerDeps): Promise<ConsumerResult> {
  if (event.type === "OrderPaid") {
    if (deps.failMode === "poison") return { ok: false, kind: "poison", error: "schema/logic poison" };

    // transient_once: fail first call then succeed
    if (deps.failMode === "transient_once") {
      deps.failMode = "none";
      return { ok: false, kind: "transient", error: "temporary downstream failure" };
    }

    // Stage ledger insert (side-effect)
    deps.uow.stage(() =>
      deps.ledger.insert({
        orderId: event.orderId,
        userId: event.userId,
        amountCents: event.amountCents,
        eventId: event.eventId,
      })
    );

    return { ok: true };
  }

  return { ok: false, kind: "poison", error: "unknown event type" };
}
```

---

## 6) `src/consumer.ts` (core orchestration)

Đây là “Senior+ glue”: claim → process in UoW → markProcessed in same commit → ack/nack/dlq.

```ts
// src/consumer.ts
import type { InMemoryBroker, BrokerMessage } from "./broker";
import type { IdempotencyStore } from "./store";
import { UnitOfWork } from "./uow";
import type { LedgerDB } from "./uow";
import { handleEvent, type HandlerDeps } from "./handlers";

export type ConsumerConfig = {
  maxAttemptsPerEvent: number; // after that -> DLQ
};

export class IdempotentConsumer {
  constructor(
    private readonly broker: InMemoryBroker,
    private readonly store: IdempotencyStore,
    private readonly ledger: LedgerDB,
    private readonly cfg: ConsumerConfig,
    private readonly now: () => number = () => Date.now()
  ) {}

  /**
   * Process exactly one message (for deterministic tests)
   */
  async tick(deps?: Omit<HandlerDeps, "ledger" | "uow">) {
    const msg = this.broker.poll();
    if (!msg) return;

    const eventId = msg.event.eventId;
    const claim = this.store.claim(eventId, this.now());

    if (claim.kind === "already_processed") {
      // duplicate delivery: safe to ack
      this.broker.ack(msg);
      return;
    }

    if (claim.kind === "locked") {
      // Someone else processing (in real world). Here just requeue.
      this.broker.nack(msg);
      return;
    }

    const lockToken = claim.lockToken;

    // Unit of Work boundary
    const uow = new UnitOfWork();
    const handlerDeps: HandlerDeps = { ledger: this.ledger, uow, failMode: deps?.failMode };

    // bump attempts for tracking (optional; do it before run)
    // (Note: we already set attempts=1 at claim. For retries, we'd claim again in real db w/ same row. Here: simple.)
    // We'll keep attempts from record; and bump when we decide to retry.
    const res = await handleEvent(msg.event, handlerDeps);

    if (res.ok) {
      // Stage idempotency mark as part of the same commit
      uow.stage(() => this.store.markProcessed({ eventId, lockToken, nowMs: this.now() }));

      // Commit atomically
      uow.commit();
      this.broker.ack(msg);
      return;
    }

    // Error path: rollback staged side-effects
    uow.rollback();

    // Decide retry / DLQ
    const rec = this.store.get(eventId);
    const attempts = rec?.state.state === "processing" ? rec.state.attempts : 1;

    if (res.kind === "poison") {
      this.broker.sendToDLQ(msg);
      this.broker.ack(msg); // ack original, we moved to DLQ
      return;
    }

    // transient
    if (attempts >= this.cfg.maxAttemptsPerEvent) {
      this.broker.sendToDLQ(msg);
      this.broker.ack(msg);
      return;
    }

    // bump attempt and nack for retry
    this.store.bumpAttempt(eventId, lockToken);
    this.broker.nack(msg);
  }

  async drain(limit = 10_000, deps?: Omit<HandlerDeps, "ledger" | "uow">) {
    let i = 0;
    while (this.broker.size() > 0 && i < limit) {
      // eslint-disable-next-line no-await-in-loop
      await this.tick(deps);
      i++;
    }
  }
}
```

> Lưu ý: In-memory store ở đây “lock” không timeout. Ở stretch bạn sẽ thêm **stale lock + fencing token**.

---

# Tests (vitest)

## `test/consumer.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { InMemoryBroker } from "../src/broker";
import { IdempotencyStore } from "../src/store";
import { LedgerDB } from "../src/uow";
import { IdempotentConsumer } from "../src/consumer";
import type { Event } from "../src/types";

function mkOrderPaid(e: Partial<Event> = {}): Event {
  return {
    type: "OrderPaid",
    eventId: e.eventId ?? `evt_${Math.random().toString(16).slice(2)}`,
    occurredAt: e.occurredAt ?? 1,
    orderId: (e as any).orderId ?? "ord_1",
    userId: (e as any).userId ?? "usr_1",
    amountCents: (e as any).amountCents ?? 1000,
  } as Event;
}

describe("Kata 82 - Idempotent Consumer", () => {
  it("dedups duplicates: ledger credited exactly once", async () => {
    const broker = new InMemoryBroker();
    const store = new IdempotencyStore();
    const ledger = new LedgerDB();
    const now = () => 100;

    const consumer = new IdempotentConsumer(broker, store, ledger, { maxAttemptsPerEvent: 3 }, now);

    const event = mkOrderPaid({ eventId: "evt_A" });
    broker.publish(event, { duplicates: 5 });

    await consumer.drain();

    const entries = ledger.all().filter((x) => x.eventId === "evt_A");
    expect(entries.length).toBe(1);
  });

  it("crash between attempts: transient failure then success still credits once", async () => {
    const broker = new InMemoryBroker();
    const store = new IdempotencyStore();
    const ledger = new LedgerDB();
    let t = 0;
    const now = () => (t += 1);

    const consumer = new IdempotentConsumer(broker, store, ledger, { maxAttemptsPerEvent: 5 }, now);

    const event = mkOrderPaid({ eventId: "evt_B" });
    broker.publish(event, { duplicates: 2 }); // duplicates + retry sim

    // First tick transient once => nack => requeued
    await consumer.tick({ failMode: "transient_once" });

    // Drain remaining => should succeed and dedup
    await consumer.drain();

    const entries = ledger.all().filter((x) => x.eventId === "evt_B");
    expect(entries.length).toBe(1);
  });

  it("poison message goes to DLQ", async () => {
    const broker = new InMemoryBroker();
    const store = new IdempotencyStore();
    const ledger = new LedgerDB();
    const now = () => 1;

    const consumer = new IdempotentConsumer(broker, store, ledger, { maxAttemptsPerEvent: 3 }, now);

    const event = mkOrderPaid({ eventId: "evt_POISON" });
    broker.publish(event);

    await consumer.tick({ failMode: "poison" });

    expect(broker.dlqMessages().length).toBe(1);
    expect(ledger.all().length).toBe(0);
  });

  it("transient retries are capped then DLQ", async () => {
    const broker = new InMemoryBroker();
    const store = new IdempotencyStore();
    const ledger = new LedgerDB();
    const now = () => 1;

    const consumer = new IdempotentConsumer(broker, store, ledger, { maxAttemptsPerEvent: 2 }, now);

    const event = mkOrderPaid({ eventId: "evt_CAP" });
    broker.publish(event);

    // Make it transient forever by always failing transient_once each tick (reset it manually)
    await consumer.tick({ failMode: "transient_once" }); // attempt 1 -> nack
    await consumer.tick({ failMode: "transient_once" }); // attempt 2 -> should DLQ by cap

    expect(broker.dlqMessages().length).toBe(1);
    expect(ledger.all().length).toBe(0);
  });
});
```

---

## Checks (Definition of Done)

* Duplicate delivery (N lần) vẫn chỉ tạo **1 ledger entry**.
* Transient error retry vẫn không double-write.
* Poison message đi **DLQ**.
* Attempts cap hoạt động.

---

# Senior+ Notes (điểm “tầng 9”)

Bạn vừa dựng **exactly-once effects** trên nền **at-least-once delivery** bằng:

* **Idempotency key** (eventId)
* **claim/lock** để tránh xử lý song song
* **transaction boundary** để “side-effect + mark processed” đồng bộ

Đây là thứ làm consumer “sống” trong distributed reality.

---

## Stretch (cao hơn nữa)

1. **Stale lock recovery**: nếu record `processing` quá lâu → cho phép “takeover” với fencing token (monotonic version).
2. **Inbox table pattern**: thay `IdempotencyStore` bằng bảng `inbox(event_id, status, updated_at)` + unique constraint.
3. **Reordering**: thêm event `OrderCreated` và rule: `OrderPaid` chỉ hợp lệ sau `OrderCreated` → cần buffer hoặc “pending state”.
4. **Side-effect idempotency**: ledger insert dùng unique constraint `(eventId)` để “double safety” (defense in depth).
5. **Observability**: metrics:

   * `consumer_dedup_total`
   * `consumer_retries_total{reason}`
   * `consumer_dlq_total{reason}`
6. **Replay safety**: chạy lại toàn bộ stream 1 ngày → ledger không đổi.