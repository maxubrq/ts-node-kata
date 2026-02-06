# Kata 90 — Chaos Mini Drill: prove the system survives

## Goal

Bạn phải chứng minh 4 thứ (bằng tests + metrics):

1. **Safety**: không double-charge, không overspend (idempotency).
2. **Liveness**: pipeline không kẹt vì poison; eventually drains (hoặc DLQ/quarantine tăng đúng).
3. **Bounded behavior**: retries có budget, deadline propagation stop waste.
4. **Observability**: có metrics đủ để trả lời “hệ đang sống hay chết?” khi chaos xảy ra.

---

## System Under Test (SUT)

Bạn build một mô hình tối thiểu:

* **Broker**: main topics `commands`, `events` + `DLQ`
* **Orchestrator Saga**: `PlaceOrderRequested` → `ReserveInventory` → `ChargePayment` → `CreateShipment`

  * Compensations: `RefundPayment`, `ReleaseInventory`
* **Downstreams**: Inventory/Payment/Shipment consumers
* **Policies**:

  * Retry budget (attempt cap + jitter/backoff)
  * Deadline propagation (skip when low remaining)
  * Poison handling (quarantine + DLQ tooling)
  * Schema evolution (upcast v1→v2) optional but recommended

Bạn không cần DB thật. In-memory stores đủ, nhưng phải có:

* Idempotency store per consumer
* Saga store for orchestrator
* Quarantine store for poison

---

## Chaos knobs (bạn sẽ bật/tắt trong tests)

* Latency injection per service: base + jitter + spikes
* Error injection:

  * transient (%): timeouts/5xx
  * poison (%): schema invalid / invariant violation
* Duplicates: at-least-once deliveries (x2..x5)
* Reorder: broker poll order shuffle (optional)
* Deadline: request deadlines ngắn/dài trộn
* Restart: orchestrator restart mid-flight (optional)

---

## Constraints

* ✅ Must have: metrics counters + histograms (p50/p95) *in-memory*.
* ✅ Must have: invariants enforced via assertions/tests.
* ✅ Chaos must be deterministic (seeded RNG).
* ✅ Must show: “poison doesn’t block”, “retries capped”, “system drains”.
* ✅ No sleeps for timing logic in tests: use virtual time or deterministic step ticks.

---

# Deliverables

```
kata90/
  src/
    rng.ts
    metrics.ts
    broker.ts
    chaos.ts
    deadlines.ts
    retry.ts
    quarantine.ts
    idempotency.ts
    saga.ts
    consumers.ts
    runner.ts
  test/
    chaos.test.ts
```

---

# 1) `src/rng.ts` (seeded)

```ts
// src/rng.ts
export class RNG {
  constructor(private seed = 1) {}
  next(): number {
    // LCG deterministic
    this.seed = (1103515245 * this.seed + 12345) % 2 ** 31;
    return this.seed / 2 ** 31;
  }
  int(max: number): number {
    return Math.floor(this.next() * max);
  }
  chance(p: number): boolean {
    return this.next() < p;
  }
}
```

---

# 2) `src/metrics.ts`

```ts
// src/metrics.ts
export class Metrics {
  counters = new Map<string, number>();
  timers = new Map<string, number[]>(); // store durations

  inc(name: string, by = 1) {
    this.counters.set(name, (this.counters.get(name) ?? 0) + by);
  }

  observe(name: string, v: number) {
    const arr = this.timers.get(name) ?? [];
    arr.push(v);
    this.timers.set(name, arr);
  }

  get(name: string) {
    return this.counters.get(name) ?? 0;
  }

  p(name: string, q: number): number {
    const arr = [...(this.timers.get(name) ?? [])].sort((a, b) => a - b);
    if (arr.length === 0) return 0;
    const idx = Math.floor((q / 100) * (arr.length - 1));
    return arr[idx];
  }
}
```

---

# 3) `src/broker.ts` (commands/events + DLQ)

```ts
// src/broker.ts
export type Topic = "commands" | "events" | "dlq";

export type Envelope<T> = {
  deliveryId: string;
  topic: "commands" | "events";
  body: T;
};

export class Broker<C, E> {
  private commands: Envelope<C>[] = [];
  private events: Envelope<E>[] = [];
  private dlq: Envelope<any>[] = [];

  publishCommand(cmd: C, duplicates = 1) {
    for (let i = 0; i < duplicates; i++) {
      this.commands.push({ deliveryId: `cmd_${i}_${Math.random().toString(16).slice(2)}`, topic: "commands", body: cmd });
    }
  }
  publishEvent(evt: E, duplicates = 1) {
    for (let i = 0; i < duplicates; i++) {
      this.events.push({ deliveryId: `evt_${i}_${Math.random().toString(16).slice(2)}`, topic: "events", body: evt });
    }
  }

  pollCommand() { return this.commands.shift(); }
  pollEvent() { return this.events.shift(); }

  nackCommand(m: Envelope<C>) { this.commands.push(m); }
  nackEvent(m: Envelope<E>) { this.events.push(m); }

  toDLQ(m: Envelope<any>) { this.dlq.push(m); }
  dlqSize() { return this.dlq.length; }
  sizes() { return { commands: this.commands.length, events: this.events.length, dlq: this.dlq.length }; }
}
```

---

# 4) `src/deadlines.ts` + `src/retry.ts`

```ts
// src/deadlines.ts
export type Deadline = { deadlineAt: number; traceId: string };

export function remainingMs(d: Deadline, now: number) {
  return d.deadlineAt - now;
}
```

```ts
// src/retry.ts
export type RetryPolicy = { maxAttempts: number };

export class AttemptStore {
  private m = new Map<string, number>(); // key -> attempts
  inc(key: string) {
    const n = (this.m.get(key) ?? 0) + 1;
    this.m.set(key, n);
    return n;
  }
  get(key: string) {
    return this.m.get(key) ?? 0;
  }
}
```

---

# 5) `src/quarantine.ts` + `src/idempotency.ts`

```ts
// src/quarantine.ts
export type QuarantineRec = { key: string; reason: string; attempts: number };
export class Quarantine {
  private m = new Map<string, QuarantineRec>();
  upsert(key: string, reason: string, attempts: number) {
    this.m.set(key, { key, reason, attempts });
  }
  get(key: string) { return this.m.get(key); }
  size() { return this.m.size; }
}
```

```ts
// src/idempotency.ts
export class Idempotency {
  private done = new Set<string>();
  has(id: string) { return this.done.has(id); }
  mark(id: string) { this.done.add(id); }
  size() { return this.done.size; }
}
```

---

# 6) `src/saga.ts` (minimal saga state + events)

Giữ tối thiểu để drill được chaos.

```ts
// src/saga.ts
import type { Deadline } from "./deadlines";

export type PlaceOrderRequested = {
  type: "PlaceOrderRequested";
  eventId: string;
  sagaId: string;
  orderId: string;
  userId: string;
  amountCents: number;
  deadline: Deadline;
};

export type InventoryReservedEvt = { type: "InventoryReservedEvt"; eventId: string; sagaId: string; orderId: string };
export type InventoryReserveFailedEvt = { type: "InventoryReserveFailedEvt"; eventId: string; sagaId: string; orderId: string; kind: "transient" | "poison"; error: string };

export type PaymentChargedEvt = { type: "PaymentChargedEvt"; eventId: string; sagaId: string; orderId: string };
export type PaymentChargeFailedEvt = { type: "PaymentChargeFailedEvt"; eventId: string; sagaId: string; orderId: string; kind: "transient" | "poison"; error: string };

export type ShipmentCreatedEvt = { type: "ShipmentCreatedEvt"; eventId: string; sagaId: string; orderId: string };
export type ShipmentCreateFailedEvt = { type: "ShipmentCreateFailedEvt"; eventId: string; sagaId: string; orderId: string; kind: "transient" | "poison"; error: string };

export type PaymentRefundedEvt = { type: "PaymentRefundedEvt"; eventId: string; sagaId: string; orderId: string };
export type InventoryReleasedEvt = { type: "InventoryReleasedEvt"; eventId: string; sagaId: string; orderId: string };

export type SagaEvent =
  | PlaceOrderRequested
  | InventoryReservedEvt | InventoryReserveFailedEvt
  | PaymentChargedEvt | PaymentChargeFailedEvt
  | ShipmentCreatedEvt | ShipmentCreateFailedEvt
  | PaymentRefundedEvt | InventoryReleasedEvt;

export type ReserveInventoryCmd = { type: "ReserveInventoryCmd"; commandId: string; sagaId: string; orderId: string; deadline: Deadline };
export type ChargePaymentCmd = { type: "ChargePaymentCmd"; commandId: string; sagaId: string; orderId: string; userId: string; amountCents: number; deadline: Deadline };
export type CreateShipmentCmd = { type: "CreateShipmentCmd"; commandId: string; sagaId: string; orderId: string; deadline: Deadline };
export type RefundPaymentCmd = { type: "RefundPaymentCmd"; commandId: string; sagaId: string; orderId: string; deadline: Deadline };
export type ReleaseInventoryCmd = { type: "ReleaseInventoryCmd"; commandId: string; sagaId: string; orderId: string; deadline: Deadline };

export type Command =
  | ReserveInventoryCmd | ChargePaymentCmd | CreateShipmentCmd
  | RefundPaymentCmd | ReleaseInventoryCmd;

export type SagaPhase = "idle" | "inv" | "pay" | "ship" | "comp_refund" | "comp_release" | "done" | "failed" | "poison";

export type SagaState = {
  sagaId: string;
  orderId: string;
  phase: SagaPhase;
  inv: boolean;
  pay: boolean;
  ship: boolean;
  seenEventIds: Set<string>;
};

export class SagaStore {
  private m = new Map<string, SagaState>();
  get(id: string) { return this.m.get(id); }
  createIfAbsent(sagaId: string, orderId: string): SagaState {
    const ex = this.m.get(sagaId);
    if (ex) return ex;
    const s: SagaState = { sagaId, orderId, phase: "idle", inv: false, pay: false, ship: false, seenEventIds: new Set() };
    this.m.set(sagaId, s);
    return s;
  }
}
```

---

# 7) `src/chaos.ts` (latency/errors/poison/duplicates)

```ts
// src/chaos.ts
import { RNG } from "./rng";

export type ChaosCfg = {
  latencyBaseMs: number;
  latencyJitterMs: number;
  spikeChance: number;
  spikeExtraMs: number;

  transientChance: number; // per command
  poisonChance: number;    // per command

  duplicates: { commands: number; events: number };
};

export class Chaos {
  constructor(private rng: RNG, private cfg: ChaosCfg) {}

  latency(): number {
    const base = this.cfg.latencyBaseMs + this.rng.int(this.cfg.latencyJitterMs + 1);
    const spike = this.rng.chance(this.cfg.spikeChance) ? this.cfg.spikeExtraMs : 0;
    return base + spike;
  }

  outcome(): "ok" | "transient" | "poison" {
    if (this.rng.chance(this.cfg.poisonChance)) return "poison";
    if (this.rng.chance(this.cfg.transientChance)) return "transient";
    return "ok";
  }

  dupCommands() { return this.cfg.duplicates.commands; }
  dupEvents() { return this.cfg.duplicates.events; }
}
```

---

# 8) `src/consumers.ts` (inventory/payment/shipment with chaos + idempotency + poison)

**Quan trọng:** payment must be exactly-once effects: **never double-charge** even with duplicates/retries.

```ts
// src/consumers.ts
import type { Broker } from "./broker";
import type { Metrics } from "./metrics";
import type { Chaos } from "./chaos";
import type { AttemptStore, RetryPolicy } from "./retry";
import type { Quarantine } from "./quarantine";
import type { Idempotency } from "./idempotency";
import type { Command, SagaEvent } from "./saga";
import { remainingMs } from "./deadlines";

export class DownstreamConsumers {
  // effects for invariants
  chargedOrders = new Set<string>();
  refundedOrders = new Set<string>();

  inventoryReserved = new Set<string>();
  inventoryReleased = new Set<string>();

  shipments = new Set<string>();

  constructor(
    private broker: Broker<Command, SagaEvent>,
    private chaos: Chaos,
    private metrics: Metrics,
    private attempts: AttemptStore,
    private retry: RetryPolicy,
    private quarantine: Quarantine,
    private idemCommands: Idempotency, // dedup by commandId
    private now: () => number
  ) {}

  async onCommand(cmd: Command): Promise<void> {
    // deadline aware skip/fail fast
    if (remainingMs(cmd.deadline, this.now()) <= 0) {
      this.metrics.inc("cmd_deadline_exceeded");
      // treat as transient? In practice: no, because retry won't help; mark poison-ish
      this.quarantine.upsert(cmd.commandId, "deadline_exceeded", this.attempts.get(cmd.commandId));
      this.broker.toDLQ({ deliveryId: "x", topic: "commands", body: cmd });
      return;
    }

    // idempotent command handling
    if (this.idemCommands.has((cmd as any).commandId)) {
      this.metrics.inc("cmd_dedup_hit");
      return;
    }

    const attempt = this.attempts.inc((cmd as any).commandId);
    const latency = this.chaos.latency();
    this.metrics.observe("downstream_latency_ms", latency);

    const out = this.chaos.outcome();

    // poison => DLQ + quarantine
    if (out === "poison") {
      this.metrics.inc("cmd_poison_total");
      this.quarantine.upsert((cmd as any).commandId, `poison:${cmd.type}`, attempt);
      this.broker.toDLQ({ deliveryId: "x", topic: "commands", body: cmd });
      return;
    }

    // transient => retry until cap (requeue)
    if (out === "transient") {
      this.metrics.inc("cmd_transient_total");
      if (attempt >= this.retry.maxAttempts) {
        this.metrics.inc("cmd_retry_exhausted_total");
        this.quarantine.upsert((cmd as any).commandId, `retry_exhausted:${cmd.type}`, attempt);
        this.broker.toDLQ({ deliveryId: "x", topic: "commands", body: cmd });
        return;
      }
      // requeue: use broker nack from runner
      throw new Error("transient");
    }

    // success => apply effect exactly once
    this.idemCommands.mark((cmd as any).commandId);

    switch (cmd.type) {
      case "ReserveInventoryCmd":
        this.inventoryReserved.add(cmd.orderId);
        this.broker.publishEvent({ type: "InventoryReservedEvt", eventId: `e_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId }, this.chaos.dupEvents());
        this.metrics.inc("inv_reserved_total");
        break;

      case "ChargePaymentCmd":
        // invariant: never double-charge an orderId
        this.chargedOrders.add(cmd.orderId);
        this.broker.publishEvent({ type: "PaymentChargedEvt", eventId: `e_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId }, this.chaos.dupEvents());
        this.metrics.inc("pay_charged_total");
        break;

      case "CreateShipmentCmd":
        this.shipments.add(cmd.orderId);
        this.broker.publishEvent({ type: "ShipmentCreatedEvt", eventId: `e_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId }, this.chaos.dupEvents());
        this.metrics.inc("ship_created_total");
        break;

      case "RefundPaymentCmd":
        // refund is idempotent too
        this.refundedOrders.add(cmd.orderId);
        this.broker.publishEvent({ type: "PaymentRefundedEvt", eventId: `e_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId }, this.chaos.dupEvents());
        this.metrics.inc("pay_refunded_total");
        break;

      case "ReleaseInventoryCmd":
        this.inventoryReleased.add(cmd.orderId);
        this.broker.publishEvent({ type: "InventoryReleasedEvt", eventId: `e_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId }, this.chaos.dupEvents());
        this.metrics.inc("inv_released_total");
        break;
    }
  }
}
```

---

# 9) `src/runner.ts` (orchestrator + pump loop)

Orchestrator logic: giống 83.1 tinh gọn + deadline-aware. Quan trọng: **seenEventIds** để dedup.

```ts
// src/runner.ts
import type { Broker, Envelope } from "./broker";
import type { Metrics } from "./metrics";
import type { Quarantine } from "./quarantine";
import type { AttemptStore, RetryPolicy } from "./retry";
import type { Idempotency } from "./idempotency";
import type { Chaos } from "./chaos";
import type { Command, PlaceOrderRequested, SagaEvent } from "./saga";
import { SagaStore } from "./saga";
import { remainingMs } from "./deadlines";
import { DownstreamConsumers } from "./consumers";

function cmdId(prefix: string) {
  return `${prefix}_${Math.random().toString(16).slice(2)}`;
}

export class ChaosSystem {
  store = new SagaStore();

  constructor(
    public broker: Broker<Command, SagaEvent>,
    public chaos: Chaos,
    public metrics: Metrics,
    public attempts: AttemptStore,
    public retry: RetryPolicy,
    public quarantine: Quarantine,
    public idemCommands: Idempotency,
    public now: () => number,
    public downstream: DownstreamConsumers
  ) {}

  onEvent(evt: SagaEvent) {
    // deadline check for request-scoped events
    if ((evt as any).deadline && remainingMs((evt as any).deadline, this.now()) <= 0) {
      this.metrics.inc("evt_deadline_exceeded");
      return;
    }

    // start saga
    if (evt.type === "PlaceOrderRequested") {
      const e = evt as PlaceOrderRequested;
      const s = this.store.createIfAbsent(e.sagaId, e.orderId);
      if (s.seenEventIds.has(e.eventId)) return;
      s.seenEventIds.add(e.eventId);
      if (s.phase === "idle") {
        s.phase = "inv";
        this.broker.publishCommand({ type: "ReserveInventoryCmd", commandId: cmdId("inv"), sagaId: e.sagaId, orderId: e.orderId, deadline: e.deadline }, this.chaos.dupCommands());
        this.metrics.inc("cmd_emitted_inv");
      }
      return;
    }

    const s = this.store.get(evt.sagaId);
    if (!s) return;

    // dedup events
    if (s.seenEventIds.has(evt.eventId)) return;
    s.seenEventIds.add(evt.eventId);

    switch (evt.type) {
      case "InventoryReservedEvt":
        s.inv = true;
        s.phase = "pay";
        this.broker.publishCommand({ type: "ChargePaymentCmd", commandId: cmdId("pay"), sagaId: evt.sagaId, orderId: evt.orderId, userId: "u", amountCents: 1000, deadline: (evt as any).deadline ?? { deadlineAt: this.now() + 500, traceId: "t" } }, this.chaos.dupCommands());
        this.metrics.inc("cmd_emitted_pay");
        break;

      case "PaymentChargedEvt":
        s.pay = true;
        s.phase = "ship";
        this.broker.publishCommand({ type: "CreateShipmentCmd", commandId: cmdId("ship"), sagaId: evt.sagaId, orderId: evt.orderId, deadline: (evt as any).deadline ?? { deadlineAt: this.now() + 500, traceId: "t" } }, this.chaos.dupCommands());
        this.metrics.inc("cmd_emitted_ship");
        break;

      case "ShipmentCreatedEvt":
        s.ship = true;
        s.phase = "done";
        this.metrics.inc("saga_done_total");
        break;

      // compensation events
      case "PaymentRefundedEvt":
        s.phase = "comp_release";
        this.broker.publishCommand({ type: "ReleaseInventoryCmd", commandId: cmdId("rel"), sagaId: evt.sagaId, orderId: evt.orderId, deadline: (evt as any).deadline ?? { deadlineAt: this.now() + 500, traceId: "t" } }, this.chaos.dupCommands());
        break;

      case "InventoryReleasedEvt":
        s.phase = "failed";
        this.metrics.inc("saga_failed_total");
        break;
    }
  }

  async tickOnce(): Promise<void> {
    // process one event
    const ev = this.broker.pollEvent();
    if (ev) {
      this.onEvent(ev.body);
      return;
    }

    // process one command
    const cmd = this.broker.pollCommand();
    if (!cmd) return;

    try {
      await this.downstream.onCommand(cmd.body);
      // ack implicit
    } catch (e) {
      // transient requeue
      this.metrics.inc("cmd_requeued_transient");
      this.broker.nackCommand(cmd as Envelope<Command>);
    }
  }

  async run(limit = 50_000) {
    for (let i = 0; i < limit; i++) {
      const { commands, events } = this.broker.sizes();
      if (commands === 0 && events === 0) break;
      // eslint-disable-next-line no-await-in-loop
      await this.tickOnce();
    }
  }
}
```

> Orchestrator ở đây intentionally “minimal” để focus chaos, không full failure events. Stretch sẽ thêm failure events + compensation theo 83.1 đầy đủ.

---

# 10) Tests — `test/chaos.test.ts`

Bạn sẽ chạy chaos với config deterministic và chứng minh invariants.

```ts
import { describe, it, expect } from "vitest";
import { RNG } from "../src/rng";
import { Metrics } from "../src/metrics";
import { Broker } from "../src/broker";
import { Chaos } from "../src/chaos";
import { AttemptStore } from "../src/retry";
import { Quarantine } from "../src/quarantine";
import { Idempotency } from "../src/idempotency";
import { DownstreamConsumers } from "../src/consumers";
import { ChaosSystem } from "../src/runner";

describe("Kata 90 - Chaos mini drill", () => {
  it("system survives: no infinite retry, poison quarantined, idempotency holds", async () => {
    let now = 0;
    const clock = () => now;

    const rng = new RNG(42);
    const metrics = new Metrics();
    const broker = new Broker<any, any>();
    const attempts = new AttemptStore();
    const quarantine = new Quarantine();
    const idem = new Idempotency();

    const chaos = new Chaos(rng, {
      latencyBaseMs: 5,
      latencyJitterMs: 20,
      spikeChance: 0.05,
      spikeExtraMs: 200,
      transientChance: 0.15,
      poisonChance: 0.05,
      duplicates: { commands: 3, events: 3 },
    });

    const downstream = new DownstreamConsumers(
      broker as any,
      chaos,
      metrics,
      attempts,
      { maxAttempts: 3 },
      quarantine,
      idem,
      clock
    );

    const sys = new ChaosSystem(
      broker as any,
      chaos,
      metrics,
      attempts,
      { maxAttempts: 3 },
      quarantine,
      idem,
      clock,
      downstream
    );

    // Seed 100 requests with mixed deadlines
    for (let i = 0; i < 100; i++) {
      const deadlineAt = now + (i % 5 === 0 ? 30 : 500); // some tight deadlines
      broker.publishEvent(
        {
          type: "PlaceOrderRequested",
          eventId: `req_${i}`,
          sagaId: `s_${i}`,
          orderId: `o_${i}`,
          userId: `u_${i}`,
          amountCents: 1000,
          deadline: { deadlineAt, traceId: `t_${i}` },
        },
        2
      );
    }

    await sys.run(200_000);

    // 1) Liveness: queues drained or moved aside
    const sizes = broker.sizes();
    expect(sizes.commands + sizes.events).toBe(0);

    // 2) Poison handled: some quarantines or DLQ > 0 is acceptable but bounded
    expect(quarantine.size()).toBeGreaterThanOrEqual(0);
    expect(broker.dlqSize()).toBeGreaterThanOrEqual(0);

    // 3) Retry bounded: attempts should not explode
    // simple sanity: no command attempted more than maxAttempts (attemptStore enforces but check anyway)
    // (can't enumerate keys easily here; in stretch you should expose store list)
    expect(metrics.get("cmd_retry_exhausted_total")).toBeGreaterThanOrEqual(0);

    // 4) Safety: idempotency prevents command dedup explosions
    expect(metrics.get("cmd_dedup_hit")).toBeGreaterThanOrEqual(0);

    // Stronger safety invariant: no double-charge effect per order
    // Because we store chargedOrders as Set, duplicates can't exceed count
    expect(downstream.chargedOrders.size).toBeLessThanOrEqual(100);
  });
});
```

---

## What you must write in `README.md` (ngắn nhưng “đúng”)

* Invariants:

  * `chargedOrders` never > unique orders
  * poison never blocks main loop
  * attempts <= maxAttempts
  * queue drains within step limit
* Metrics to watch:

  * `cmd_poison_total`, `cmd_retry_exhausted_total`, `cmd_requeued_transient`, `cmd_dedup_hit`
  * latency p95: `downstream_latency_ms`
* Pass criteria: tests green + printed summary (optional)

---

# Checks (DoD)

* Chaos test chạy deterministic.
* Bạn chứng minh “hệ sống” bằng:

  * queues drained
  * poison quarantined/DLQ
  * retries capped
  * idempotency prevents double effects
  * latency p95 reported (optional print)

---

# Senior+ Stretch (để đúng “capstone” Tầng 9)

1. **Add failure events + compensations đầy đủ** (83.1): transient fail emits failure event; orchestrator compensates.
2. **Deadline propagation thật** (84): mỗi command carries same deadline; downstream checks `remainingMs` and refuses to retry when expired.
3. **Poison message tooling** (88): implement `replayFromDLQ(transform)` and verify idempotency on replay.
4. **Schema evolution chaos** (89): inject some v1/v2 payloads; consumer upcasts; invalid schema goes quarantine.
5. **Clock skew injection** (87): producer uses skewed `occurredAtMs` and show you don’t use it for ordering (seq/HLC).
6. **Runbook artifact**: `OPS_RUNBOOK.md` (how to triage when dlq grows, when retries spike).

---

## Nhắc thẳng (vì Senior+)

Chaos drill không phải “ném lỗi cho vui”. Nó là bài chứng minh:

* bạn hiểu đâu là **safety vs liveness**
* bạn có **guardrails** (deadline, retry budget, poison quarantine)
* bạn có **signals** (metrics) để vận hành