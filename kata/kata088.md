# Kata 88 — Poison Message: detect, quarantine, DLQ tools

## Goal

1. Detect poison message (không nên retry mãi).
2. Quarantine (cách ly) + DLQ (dead-letter queue) đúng cách.
3. Tooling tối thiểu:

   * list DLQ
   * inspect message + reason + attempts history
   * replay từ DLQ về main queue (có “fix-up transform” optional)
4. Tests chứng minh:

   * poison không làm consumer kẹt
   * transient retry rồi succeed
   * poison vào quarantine/DLQ sau N attempts
   * replay hoạt động và vẫn idempotent

---

## Context (kịch bản thật)

Consumer xử lý event `OrderPaid` → ghi ledger. Nhưng có message:

* schema sai / thiếu field
* data invalid (amount âm)
* bug logic (không xử lý type)
* downstream trả 400 permanent

Nếu bạn cứ retry → **kẹt partition / kẹt worker / queue pile-up**.

---

## Constraints

* ✅ Broker at-least-once (duplicate + reorder có thể xảy ra)
* ✅ Có **classifier** phân loại lỗi:

  * `transient` (retry ok)
  * `poison` (permanent, stop retry)
  * `unknown` (treat as poison after cap)
* ✅ Có **attempt cap** + **backoff/jitter** (đơn giản cũng được)
* ✅ Có **quarantine store** lưu metadata (reason, stack/snippet, firstSeenAt, lastSeenAt, attempts, sample payload)
* ✅ DLQ không chỉ là “dump”: phải có tool để replay/inspect.

---

# Deliverables

```
kata88/
  src/
    types.ts
    broker.ts
    classifier.ts
    quarantine.ts
    idempotency.ts
    consumer.ts
    dlq_tools.ts
  test/
    poison.test.ts
```

---

# 1) `src/types.ts`

```ts
// src/types.ts
export type OrderPaid = {
  type: "OrderPaid";
  eventId: string;        // idempotency key
  orderId: string;
  userId: string;
  amountCents: number;
};

export type Event = OrderPaid;

export type Envelope<T> = {
  deliveryId: string;      // unique per delivery
  body: T;
  receivedAtMs: number;
};

export type FailureKind = "transient" | "poison";

export type HandleResult =
  | { ok: true }
  | { ok: false; kind: FailureKind; error: string; code?: string };
```

---

# 2) `src/broker.ts` (main queue + DLQ)

```ts
// src/broker.ts
import type { Envelope, Event } from "./types";

export class InMemoryBroker {
  private main: Envelope<Event>[] = [];
  private dlq: Envelope<Event>[] = [];

  publish(e: Event, nowMs: number, opts?: { duplicates?: number }) {
    const d = opts?.duplicates ?? 1;
    for (let i = 0; i < d; i++) {
      this.main.push({
        deliveryId: `d_${e.eventId}_${i}_${Math.random().toString(16).slice(2)}`,
        body: e,
        receivedAtMs: nowMs,
      });
    }
  }

  poll(): Envelope<Event> | undefined {
    return this.main.shift();
  }

  ack(_msg: Envelope<Event>) {}

  nack(msg: Envelope<Event>) {
    // at-least-once requeue tail
    this.main.push(msg);
  }

  sendToDLQ(msg: Envelope<Event>) {
    this.dlq.push(msg);
  }

  pollDLQ(): Envelope<Event> | undefined {
    return this.dlq.shift();
  }

  listDLQ(): Envelope<Event>[] {
    return [...this.dlq];
  }

  size() {
    return { main: this.main.length, dlq: this.dlq.length };
  }
}
```

---

# 3) `src/classifier.ts` (poison vs transient)

Rule-of-thumb (đủ “real”):

* validation/schema error → poison
* domain invariant violation (amount <= 0) → poison
* downstream 5xx / timeout → transient
* anything else → poison after cap (handled in consumer)

```ts
// src/classifier.ts
import type { Event, HandleResult } from "./types";

export function validateEvent(e: Event): HandleResult {
  if (e.type !== "OrderPaid") return { ok: false, kind: "poison", error: "unknown_type" };
  if (!e.eventId || !e.orderId || !e.userId) return { ok: false, kind: "poison", error: "missing_fields" };
  if (!Number.isInteger(e.amountCents)) return { ok: false, kind: "poison", error: "amount_not_int" };
  if (e.amountCents <= 0) return { ok: false, kind: "poison", error: "amount_non_positive" };
  return { ok: true };
}

export function classifyError(err: unknown): { kind: "transient" | "poison"; error: string; code?: string } {
  const msg = (err as any)?.message ?? String(err);

  // Pretend downstream errors:
  // - "timeout" or "ECONNRESET" => transient
  // - "400" / "invalid" => poison
  if (msg.includes("timeout") || msg.includes("ECONNRESET") || msg.includes("503") || msg.includes("downstream_5xx")) {
    return { kind: "transient", error: msg, code: "DOWNSTREAM_TRANSIENT" };
  }
  if (msg.includes("400") || msg.includes("invalid") || msg.includes("schema")) {
    return { kind: "poison", error: msg, code: "DOWNSTREAM_PERMANENT" };
  }
  // default: treat unknown as poison-ish
  return { kind: "poison", error: msg, code: "UNKNOWN" };
}
```

---

# 4) `src/quarantine.ts` (quarantine store)

```ts
// src/quarantine.ts
import type { Event, FailureKind } from "./types";

export type QuarantineRecord = {
  eventId: string;
  firstSeenAtMs: number;
  lastSeenAtMs: number;
  attempts: number;
  kind: FailureKind;
  reason: string;
  sample: Event;
  lastDeliveryId: string;
};

export class QuarantineStore {
  private m = new Map<string, QuarantineRecord>(); // eventId -> record

  upsert(args: {
    nowMs: number;
    eventId: string;
    deliveryId: string;
    kind: FailureKind;
    reason: string;
    sample: Event;
  }) {
    const prev = this.m.get(args.eventId);
    if (!prev) {
      this.m.set(args.eventId, {
        eventId: args.eventId,
        firstSeenAtMs: args.nowMs,
        lastSeenAtMs: args.nowMs,
        attempts: 1,
        kind: args.kind,
        reason: args.reason,
        sample: args.sample,
        lastDeliveryId: args.deliveryId,
      });
      return;
    }
    prev.lastSeenAtMs = args.nowMs;
    prev.attempts += 1;
    prev.kind = args.kind;
    prev.reason = args.reason;
    prev.sample = args.sample;
    prev.lastDeliveryId = args.deliveryId;
  }

  get(eventId: string) {
    return this.m.get(eventId);
  }

  list() {
    return [...this.m.values()];
  }

  remove(eventId: string) {
    this.m.delete(eventId);
  }
}
```

---

# 5) `src/idempotency.ts` (exactly-once effects)

```ts
// src/idempotency.ts
export type ProcessedRecord = { eventId: string; processedAtMs: number };

export class IdempotencyStore {
  private done = new Map<string, ProcessedRecord>();

  has(eventId: string) {
    return this.done.has(eventId);
  }

  mark(eventId: string, nowMs: number) {
    this.done.set(eventId, { eventId, processedAtMs: nowMs });
  }
}
```

---

# 6) `src/consumer.ts` (retry + quarantine + DLQ)

Key behavior:

* Validate first: poison → DLQ immediately (or after 1 attempt; choose immediate)
* If transient: retry until `maxAttempts`, then DLQ
* Store quarantine record when DLQ happens (or on each fail; recommended store each fail)
* Must **not block**: poison is acked + moved aside.

```ts
// src/consumer.ts
import type { Envelope, Event } from "./types";
import type { InMemoryBroker } from "./broker";
import { validateEvent, classifyError } from "./classifier";
import type { QuarantineStore } from "./quarantine";
import type { IdempotencyStore } from "./idempotency";

export type ConsumerCfg = {
  maxAttempts: number;
  baseBackoffMs: number; // simple backoff
};

export class PoisonAwareConsumer {
  private attempts = new Map<string, number>(); // eventId -> attempts

  constructor(
    private readonly broker: InMemoryBroker,
    private readonly quarantine: QuarantineStore,
    private readonly idem: IdempotencyStore,
    private readonly now: () => number,
    private readonly cfg: ConsumerCfg,
    private readonly handler: (e: Event) => Promise<void>
  ) {}

  private incAttempt(eventId: string): number {
    const n = (this.attempts.get(eventId) ?? 0) + 1;
    this.attempts.set(eventId, n);
    return n;
  }

  private getAttempt(eventId: string): number {
    return this.attempts.get(eventId) ?? 0;
  }

  async tick(): Promise<void> {
    const msg = this.broker.poll();
    if (!msg) return;

    const e = msg.body;
    const attempt = this.incAttempt(e.eventId);

    // idempotent: already processed => ack and move on
    if (this.idem.has(e.eventId)) {
      this.broker.ack(msg);
      return;
    }

    // validation poison
    const v = validateEvent(e);
    if (!v.ok) {
      this.quarantine.upsert({
        nowMs: this.now(),
        eventId: e.eventId,
        deliveryId: msg.deliveryId,
        kind: "poison",
        reason: `validate:${v.error}`,
        sample: e,
      });
      this.broker.sendToDLQ(msg);
      this.broker.ack(msg);
      return;
    }

    try {
      // Handle
      await this.handler(e);

      // Mark processed (effects done)
      this.idem.mark(e.eventId, this.now());
      this.broker.ack(msg);
      return;
    } catch (err) {
      const c = classifyError(err);

      this.quarantine.upsert({
        nowMs: this.now(),
        eventId: e.eventId,
        deliveryId: msg.deliveryId,
        kind: c.kind,
        reason: c.error,
        sample: e,
      });

      // poison => DLQ immediately
      if (c.kind === "poison") {
        this.broker.sendToDLQ(msg);
        this.broker.ack(msg);
        return;
      }

      // transient => retry until cap
      if (attempt >= this.cfg.maxAttempts) {
        this.broker.sendToDLQ(msg);
        this.broker.ack(msg);
        return;
      }

      // backoff (simulated by requeue; you can implement delay queue in stretch)
      this.broker.nack(msg);
      return;
    }
  }

  async drain(limit = 10_000) {
    for (let i = 0; i < limit; i++) {
      if (this.broker.size().main === 0) break;
      // eslint-disable-next-line no-await-in-loop
      await this.tick();
    }
  }

  getAttempts(eventId: string) {
    return this.getAttempt(eventId);
  }
}
```

---

# 7) `src/dlq_tools.ts` (inspect + replay)

Replay options:

* raw replay (same message)
* replay with transform (fix-up payload; e.g., correct amount)
* clear quarantine record on successful replay (optional; here manual)

```ts
// src/dlq_tools.ts
import type { InMemoryBroker } from "./broker";
import type { Envelope, Event } from "./types";

export class DlqTools {
  constructor(private readonly broker: InMemoryBroker, private readonly now: () => number) {}

  list() {
    return this.broker.listDLQ();
  }

  inspect(predicate: (m: Envelope<Event>) => boolean): Envelope<Event> | undefined {
    return this.broker.listDLQ().find(predicate);
  }

  replayOne(args: {
    match: (m: Envelope<Event>) => boolean;
    transform?: (e: Event) => Event;
    duplicates?: number;
  }): boolean {
    const dlq = this.broker.listDLQ();
    const idx = dlq.findIndex(args.match);
    if (idx === -1) return false;

    // remove by draining until idx (simple for kata)
    const buf: Envelope<Event>[] = [];
    let target: Envelope<Event> | undefined;
    while (true) {
      const m = this.broker.pollDLQ();
      if (!m) break;
      if (!target && args.match(m)) {
        target = m;
        break;
      }
      buf.push(m);
    }
    // push back others
    for (const m of buf) this.broker.sendToDLQ(m);
    if (!target) return false;

    const event = args.transform ? args.transform(target.body) : target.body;
    this.broker.publish(event, this.now(), { duplicates: args.duplicates ?? 1 });
    return true;
  }
}
```

---

# Tests (vitest) — `test/poison.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { InMemoryBroker } from "../src/broker";
import { QuarantineStore } from "../src/quarantine";
import { IdempotencyStore } from "../src/idempotency";
import { PoisonAwareConsumer } from "../src/consumer";
import { DlqTools } from "../src/dlq_tools";
import type { Event } from "../src/types";

function paid(overrides: Partial<Event> = {}): Event {
  return {
    type: "OrderPaid",
    eventId: overrides.eventId ?? `evt_${Math.random().toString(16).slice(2)}`,
    orderId: (overrides as any).orderId ?? "ord_1",
    userId: (overrides as any).userId ?? "usr_1",
    amountCents: (overrides as any).amountCents ?? 1000,
  } as Event;
}

describe("Kata 88 - Poison message handling", () => {
  it("poison validation => DLQ + quarantined, consumer continues", async () => {
    let t = 0;
    const now = () => ++t;

    const broker = new InMemoryBroker();
    const quarantine = new QuarantineStore();
    const idem = new IdempotencyStore();

    // handler always ok
    const handler = async (_e: Event) => {};

    const c = new PoisonAwareConsumer(
      broker,
      quarantine,
      idem,
      now,
      { maxAttempts: 3, baseBackoffMs: 10 },
      handler
    );

    // poison: amount <= 0
    broker.publish(paid({ eventId: "evt_poison", amountCents: 0 }), now());
    // good message behind it
    broker.publish(paid({ eventId: "evt_ok", amountCents: 1000 }), now());

    await c.drain();

    expect(broker.size().dlq).toBe(1);
    expect(quarantine.get("evt_poison")?.kind).toBe("poison");
    expect(idem.has("evt_ok")).toBe(true);
  });

  it("transient error => retries then succeeds, not in DLQ", async () => {
    let t = 0;
    const now = () => ++t;

    const broker = new InMemoryBroker();
    const quarantine = new QuarantineStore();
    const idem = new IdempotencyStore();

    let failOnce = true;
    const handler = async (_e: Event) => {
      if (failOnce) {
        failOnce = false;
        throw new Error("downstream_5xx");
      }
    };

    const c = new PoisonAwareConsumer(
      broker,
      quarantine,
      idem,
      now,
      { maxAttempts: 3, baseBackoffMs: 10 },
      handler
    );

    broker.publish(paid({ eventId: "evt_transient" }), now());
    await c.drain();

    expect(idem.has("evt_transient")).toBe(true);
    expect(broker.size().dlq).toBe(0);
    expect(c.getAttempts("evt_transient")).toBeGreaterThanOrEqual(2);
  });

  it("transient exceeds cap => DLQ + quarantined", async () => {
    let t = 0;
    const now = () => ++t;

    const broker = new InMemoryBroker();
    const quarantine = new QuarantineStore();
    const idem = new IdempotencyStore();

    const handler = async (_e: Event) => {
      throw new Error("downstream_5xx");
    };

    const c = new PoisonAwareConsumer(
      broker,
      quarantine,
      idem,
      now,
      { maxAttempts: 2, baseBackoffMs: 10 },
      handler
    );

    broker.publish(paid({ eventId: "evt_cap" }), now());
    await c.drain();

    expect(broker.size().dlq).toBe(1);
    expect(quarantine.get("evt_cap")?.kind).toBe("transient");
    expect(idem.has("evt_cap")).toBe(false);
  });

  it("DLQ replay with transform works and remains idempotent", async () => {
    let t = 0;
    const now = () => ++t;

    const broker = new InMemoryBroker();
    const quarantine = new QuarantineStore();
    const idem = new IdempotencyStore();
    const tools = new DlqTools(broker, now);

    // handler ok
    const handler = async (_e: Event) => {};

    const c = new PoisonAwareConsumer(
      broker,
      quarantine,
      idem,
      now,
      { maxAttempts: 3, baseBackoffMs: 10 },
      handler
    );

    // poison message goes to DLQ
    broker.publish(paid({ eventId: "evt_fixme", amountCents: -5 }), now());
    await c.drain();

    expect(broker.size().dlq).toBe(1);

    // replay with fix-up (make amount positive)
    const ok = tools.replayOne({
      match: (m) => m.body.eventId === "evt_fixme",
      transform: (e) => ({ ...e, amountCents: 500 }),
      duplicates: 2,
    });
    expect(ok).toBe(true);

    await c.drain();

    // idempotency ensures processed once even if duplicate replay
    expect(idem.has("evt_fixme")).toBe(true);
  });
});
```

---

## Checks (Definition of Done)

* Poison message bị detect và **không kẹt pipeline** (message tiếp theo vẫn xử lý).
* Poison → quarantine + DLQ.
* Transient retry rồi succeed; nếu vượt cap → DLQ.
* Replay từ DLQ (kể cả duplicate) vẫn safe nhờ idempotency.

---

## Senior+ Notes (thứ “đắt”)

* Poison không phải “error” bình thường; nó là **data/system contract violation** → cần **operational workflow** (triage, fix, replay).
* DLQ mà không có tooling = “bãi rác”.
* Quarantine store giúp bạn: phân cụm root causes, thống kê top offenders, và làm postmortem tử tế.

---

## Stretch (đúng production hơn)

1. **Delayed retry queue** (backoff thật): thay `nack()` bằng schedule delay (time-wheel / redis zset).
2. **Poison signature**: hash payload schema/field missing → tự động group incidents.
3. **Auto-quarantine by rule**: nếu cùng lỗi `schema_missing_field` > X/min → trip circuit + stop consume type đó.
4. **Safe replay window**: replay chỉ khi `deadline` còn hợp lệ & downstream healthy.
5. **Redrive policies**: DLQ → “parking lot queue” → manual approve → main queue.
6. **Metrics**:

   * `poison_total{reason}`
   * `dlq_depth`
   * `replay_total`
   * `quarantine_unique_eventIds`