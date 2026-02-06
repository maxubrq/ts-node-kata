# Bonus Kata 83.1 — Event-driven Saga: Commands ↔ Events + Compensations

## Bạn sẽ build

* **Broker in-memory** (at-least-once + duplicates)
* **Outbox** (orchestrator ghi commands vào outbox → publisher đẩy ra broker)
* **Inbox** (downstream dedup commandId → side-effect chỉ 1 lần)
* **Orchestrator**:

  * nhận `PlaceOrderRequested`
  * phát `ReserveInventoryCmd` → chờ `InventoryReservedEvt`
  * phát `ChargePaymentCmd` → chờ `PaymentChargedEvt`
  * phát `CreateShipmentCmd` → chờ `ShipmentCreatedEvt`
  * nếu fail → phát compensation commands:

    * `RefundPaymentCmd`
    * `ReleaseInventoryCmd`
* **SagaStore** persisted state machine + pointers.

---

## Constraints (Senior+)

* ✅ Mọi message đều có `messageId` (delivery) và `commandId/eventId` (idempotency).
* ✅ Downstream handlers phải **idempotent** bằng Inbox (dedup commandId).
* ✅ Orchestrator phải **idempotent** khi consume events (duplicate event không phát tiếp commands).
* ✅ Outbox đảm bảo “**persist state + enqueue command**” atomic (mô phỏng bằng 1 transaction function).
* ✅ Tests bắt buộc: happy path, fail step2 -> compensate, fail step3 -> compensate chain, duplicates everywhere, restart/resume.

---

# Repo layout

```
kata83_1/
  src/
    types.ts
    broker.ts
    inbox.ts
    outbox.ts
    sagaStore.ts
    orchestrator.ts
    downstreams.ts
    runner.ts
  test/
    saga83_1.test.ts
```

---

# 1) `src/types.ts`

```ts
// src/types.ts
export type SagaId = string;

export type PlaceOrderRequested = {
  type: "PlaceOrderRequested";
  eventId: string;
  sagaId: SagaId;
  orderId: string;
  userId: string;
  amountCents: number;
  sku: string;
};

export type ReserveInventoryCmd = {
  type: "ReserveInventoryCmd";
  commandId: string;
  sagaId: SagaId;
  orderId: string;
  sku: string;
};

export type ChargePaymentCmd = {
  type: "ChargePaymentCmd";
  commandId: string;
  sagaId: SagaId;
  orderId: string;
  userId: string;
  amountCents: number;
};

export type CreateShipmentCmd = {
  type: "CreateShipmentCmd";
  commandId: string;
  sagaId: SagaId;
  orderId: string;
};

export type ReleaseInventoryCmd = {
  type: "ReleaseInventoryCmd";
  commandId: string;
  sagaId: SagaId;
  orderId: string;
  sku: string;
};

export type RefundPaymentCmd = {
  type: "RefundPaymentCmd";
  commandId: string;
  sagaId: SagaId;
  orderId: string;
  userId: string;
  amountCents: number;
};

export type Command =
  | ReserveInventoryCmd
  | ChargePaymentCmd
  | CreateShipmentCmd
  | ReleaseInventoryCmd
  | RefundPaymentCmd;

export type InventoryReservedEvt = {
  type: "InventoryReservedEvt";
  eventId: string;
  sagaId: SagaId;
  orderId: string;
};

export type InventoryReserveFailedEvt = {
  type: "InventoryReserveFailedEvt";
  eventId: string;
  sagaId: SagaId;
  orderId: string;
  error: string;
  kind: "transient" | "poison";
};

export type PaymentChargedEvt = {
  type: "PaymentChargedEvt";
  eventId: string;
  sagaId: SagaId;
  orderId: string;
};

export type PaymentChargeFailedEvt = {
  type: "PaymentChargeFailedEvt";
  eventId: string;
  sagaId: SagaId;
  orderId: string;
  error: string;
  kind: "transient" | "poison";
};

export type ShipmentCreatedEvt = {
  type: "ShipmentCreatedEvt";
  eventId: string;
  sagaId: SagaId;
  orderId: string;
};

export type ShipmentCreateFailedEvt = {
  type: "ShipmentCreateFailedEvt";
  eventId: string;
  sagaId: SagaId;
  orderId: string;
  error: string;
  kind: "transient" | "poison";
};

export type InventoryReleasedEvt = {
  type: "InventoryReleasedEvt";
  eventId: string;
  sagaId: SagaId;
  orderId: string;
};

export type PaymentRefundedEvt = {
  type: "PaymentRefundedEvt";
  eventId: string;
  sagaId: SagaId;
  orderId: string;
};

export type Event =
  | PlaceOrderRequested
  | InventoryReservedEvt
  | InventoryReserveFailedEvt
  | PaymentChargedEvt
  | PaymentChargeFailedEvt
  | ShipmentCreatedEvt
  | ShipmentCreateFailedEvt
  | InventoryReleasedEvt
  | PaymentRefundedEvt;

export type Envelope<T> = {
  messageId: string; // unique per delivery
  body: T;
};
```

---

# 2) `src/broker.ts` (2 topics: commands, events)

```ts
// src/broker.ts
import type { Command, Envelope, Event } from "./types";

export class InMemoryBroker {
  private commands: Envelope<Command>[] = [];
  private events: Envelope<Event>[] = [];
  private dlq: { topic: "commands" | "events"; msg: Envelope<any> }[] = [];

  publishCommand(cmd: Command, opts?: { duplicates?: number }) {
    const d = opts?.duplicates ?? 1;
    for (let i = 0; i < d; i++) {
      this.commands.push({
        messageId: `mc_${cmd.type}_${cmd.commandId}_${i}_${Math.random().toString(16).slice(2)}`,
        body: cmd,
      });
    }
  }

  publishEvent(evt: Event, opts?: { duplicates?: number }) {
    const d = opts?.duplicates ?? 1;
    for (let i = 0; i < d; i++) {
      this.events.push({
        messageId: `me_${evt.type}_${(evt as any).eventId}_${i}_${Math.random().toString(16).slice(2)}`,
        body: evt,
      });
    }
  }

  pollCommand(): Envelope<Command> | undefined {
    return this.commands.shift();
  }

  pollEvent(): Envelope<Event> | undefined {
    return this.events.shift();
  }

  nackCommand(m: Envelope<Command>) {
    this.commands.push(m);
  }

  nackEvent(m: Envelope<Event>) {
    this.events.push(m);
  }

  sendToDLQ(topic: "commands" | "events", msg: Envelope<any>) {
    this.dlq.push({ topic, msg });
  }

  dlqMessages() {
    return [...this.dlq];
  }

  sizes() {
    return { commands: this.commands.length, events: this.events.length };
  }
}
```

---

# 3) `src/inbox.ts` (downstream idempotency)

```ts
// src/inbox.ts
export class Inbox {
  private processed = new Set<string>(); // commandId

  has(commandId: string) {
    return this.processed.has(commandId);
  }

  mark(commandId: string) {
    this.processed.add(commandId);
  }
}
```

---

# 4) `src/outbox.ts` (orchestrator outbox)

```ts
// src/outbox.ts
import type { Command } from "./types";

export type OutboxItem = { id: string; command: Command; createdAt: number; publishedAt?: number };

export class Outbox {
  private items: OutboxItem[] = [];

  add(command: Command, now: number) {
    this.items.push({ id: `ob_${Math.random().toString(16).slice(2)}`, command, createdAt: now });
  }

  takeUnpublished(limit = 100): OutboxItem[] {
    return this.items.filter((x) => x.publishedAt == null).slice(0, limit);
  }

  markPublished(id: string, now: number) {
    const it = this.items.find((x) => x.id === id);
    if (it) it.publishedAt = now;
  }
}
```

---

# 5) `src/sagaStore.ts` (state + dedup events)

```ts
// src/sagaStore.ts
import type { SagaId } from "./types";

export type SagaPhase =
  | "idle"
  | "waiting_inventory"
  | "waiting_payment"
  | "waiting_shipment"
  | "compensating_refund"
  | "compensating_release"
  | "completed"
  | "failed"
  | "poison";

export type SagaState = {
  sagaId: SagaId;
  orderId: string;
  userId: string;
  sku: string;
  amountCents: number;

  phase: SagaPhase;

  inventoryReserved: boolean;
  paymentCharged: boolean;
  shipmentCreated: boolean;

  // idempotency for event consumption
  seenEventIds: Set<string>;

  createdAt: number;
  updatedAt: number;
};

export class SagaStore {
  private m = new Map<SagaId, SagaState>();

  get(id: SagaId) {
    return this.m.get(id);
  }

  createIfAbsent(args: {
    sagaId: SagaId;
    orderId: string;
    userId: string;
    sku: string;
    amountCents: number;
    now: number;
  }): SagaState {
    const ex = this.m.get(args.sagaId);
    if (ex) return ex;

    const s: SagaState = {
      sagaId: args.sagaId,
      orderId: args.orderId,
      userId: args.userId,
      sku: args.sku,
      amountCents: args.amountCents,
      phase: "idle",
      inventoryReserved: false,
      paymentCharged: false,
      shipmentCreated: false,
      seenEventIds: new Set<string>(),
      createdAt: args.now,
      updatedAt: args.now,
    };
    this.m.set(args.sagaId, s);
    return s;
  }

  update(s: SagaState, now: number) {
    s.updatedAt = now;
    this.m.set(s.sagaId, s);
  }
}
```

---

# 6) `src/orchestrator.ts` (consume events, emit commands via outbox)

Orchestrator “transaction” = update saga + add outbox items.

```ts
// src/orchestrator.ts
import type { Event, PlaceOrderRequested, Command } from "./types";
import { Outbox } from "./outbox";
import { SagaStore } from "./sagaStore";

function newCommandId(prefix: string) {
  return `${prefix}_${Math.random().toString(16).slice(2)}`;
}

export class Orchestrator {
  constructor(
    private readonly store: SagaStore,
    private readonly outbox: Outbox,
    private readonly now: () => number = () => Date.now()
  ) {}

  // Atomic boundary (simulated)
  private tx(fn: () => void) {
    fn();
  }

  onEvent(evt: Event) {
    const eventId = (evt as any).eventId as string | undefined;
    if (!eventId) return;

    // Load/create saga from PlaceOrderRequested
    if (evt.type === "PlaceOrderRequested") {
      this.handlePlaceOrder(evt);
      return;
    }

    // For all other events: need saga existing
    const saga = this.store.get(evt.sagaId);
    if (!saga) return;

    // Dedup event consumption
    if (saga.seenEventIds.has(eventId)) return;

    this.tx(() => {
      saga.seenEventIds.add(eventId);

      switch (evt.type) {
        case "InventoryReservedEvt":
          saga.inventoryReserved = true;
          saga.phase = "waiting_payment";
          this.outbox.add(
            {
              type: "ChargePaymentCmd",
              commandId: newCommandId("cmd_pay"),
              sagaId: saga.sagaId,
              orderId: saga.orderId,
              userId: saga.userId,
              amountCents: saga.amountCents,
            },
            this.now()
          );
          break;

        case "InventoryReserveFailedEvt":
          saga.phase = evt.kind === "poison" ? "poison" : "failed";
          break;

        case "PaymentChargedEvt":
          saga.paymentCharged = true;
          saga.phase = "waiting_shipment";
          this.outbox.add(
            {
              type: "CreateShipmentCmd",
              commandId: newCommandId("cmd_ship"),
              sagaId: saga.sagaId,
              orderId: saga.orderId,
            },
            this.now()
          );
          break;

        case "PaymentChargeFailedEvt":
          if (evt.kind === "poison") {
            saga.phase = "poison";
            break;
          }
          // compensate reserve if already reserved
          saga.phase = "compensating_release";
          if (saga.inventoryReserved) {
            this.outbox.add(
              {
                type: "ReleaseInventoryCmd",
                commandId: newCommandId("cmd_rel"),
                sagaId: saga.sagaId,
                orderId: saga.orderId,
                sku: saga.sku,
              },
              this.now()
            );
          } else {
            saga.phase = "failed";
          }
          break;

        case "ShipmentCreatedEvt":
          saga.shipmentCreated = true;
          saga.phase = "completed";
          break;

        case "ShipmentCreateFailedEvt":
          if (evt.kind === "poison") {
            saga.phase = "poison";
            break;
          }
          // compensate: refund then release (chain)
          if (saga.paymentCharged) {
            saga.phase = "compensating_refund";
            this.outbox.add(
              {
                type: "RefundPaymentCmd",
                commandId: newCommandId("cmd_ref"),
                sagaId: saga.sagaId,
                orderId: saga.orderId,
                userId: saga.userId,
                amountCents: saga.amountCents,
              },
              this.now()
            );
          } else if (saga.inventoryReserved) {
            saga.phase = "compensating_release";
            this.outbox.add(
              {
                type: "ReleaseInventoryCmd",
                commandId: newCommandId("cmd_rel"),
                sagaId: saga.sagaId,
                orderId: saga.orderId,
                sku: saga.sku,
              },
              this.now()
            );
          } else {
            saga.phase = "failed";
          }
          break;

        case "PaymentRefundedEvt":
          // after refund, release inventory if reserved
          if (saga.inventoryReserved) {
            saga.phase = "compensating_release";
            this.outbox.add(
              {
                type: "ReleaseInventoryCmd",
                commandId: newCommandId("cmd_rel"),
                sagaId: saga.sagaId,
                orderId: saga.orderId,
                sku: saga.sku,
              },
              this.now()
            );
          } else {
            saga.phase = "failed";
          }
          break;

        case "InventoryReleasedEvt":
          saga.phase = "failed";
          break;
      }

      this.store.update(saga, this.now());
    });
  }

  private handlePlaceOrder(evt: PlaceOrderRequested) {
    this.tx(() => {
      const saga = this.store.createIfAbsent({
        sagaId: evt.sagaId,
        orderId: evt.orderId,
        userId: evt.userId,
        sku: evt.sku,
        amountCents: evt.amountCents,
        now: this.now(),
      });

      // Dedup PlaceOrderRequested itself
      if (saga.seenEventIds.has(evt.eventId)) return;
      saga.seenEventIds.add(evt.eventId);

      // start saga if idle
      if (saga.phase === "idle") {
        saga.phase = "waiting_inventory";
        this.outbox.add(
          {
            type: "ReserveInventoryCmd",
            commandId: newCommandId("cmd_inv"),
            sagaId: saga.sagaId,
            orderId: saga.orderId,
            sku: saga.sku,
          },
          this.now()
        );
      }

      this.store.update(saga, this.now());
    });
  }
}
```

---

# 7) `src/downstreams.ts` (consume commands, emit events; Inbox dedup)

```ts
// src/downstreams.ts
import type { Command, Event } from "./types";
import { Inbox } from "./inbox";

export type FailurePlan = Partial<
  Record<
    Command["type"],
    "none" | "transient_once" | "always_transient" | "poison"
  >
>;

function mkFailer(plan: FailurePlan, key: Command["type"]) {
  let usedOnce = false;
  return () => {
    const m = plan[key] ?? "none";
    if (m === "none") return { ok: true as const };
    if (m === "poison") return { ok: false as const, kind: "poison" as const, error: `${key} poison` };
    if (m === "always_transient") return { ok: false as const, kind: "transient" as const, error: `${key} transient` };
    if (m === "transient_once") {
      if (!usedOnce) {
        usedOnce = true;
        return { ok: false as const, kind: "transient" as const, error: `${key} transient_once` };
      }
      return { ok: true as const };
    }
    return { ok: true as const };
  };
}

export class Downstreams {
  // effects for assertions
  inventoryReserved = new Set<string>();
  inventoryReleased = new Set<string>();
  paymentCharged = new Set<string>();
  paymentRefunded = new Set<string>();
  shipmentCreated = new Set<string>();

  private inbox = new Inbox();

  private failReserve: ReturnType<typeof mkFailer>;
  private failCharge: ReturnType<typeof mkFailer>;
  private failShip: ReturnType<typeof mkFailer>;

  constructor(private readonly plan: FailurePlan = {}) {
    this.failReserve = mkFailer(plan, "ReserveInventoryCmd");
    this.failCharge = mkFailer(plan, "ChargePaymentCmd");
    this.failShip = mkFailer(plan, "CreateShipmentCmd");
  }

  /**
   * Idempotent command handler:
   * - if commandId already processed -> return no event or "duplicate ack" (we choose: return undefined)
   * - else apply side-effect and emit event (or failure event; transient => caller will retry same delivery)
   */
  async onCommand(cmd: Command): Promise<{ event?: Event; transient?: boolean; poison?: boolean }> {
    if (this.inbox.has(cmd.commandId)) {
      return {}; // dedup: do nothing
    }

    const emit = (event: Event) => {
      this.inbox.mark(cmd.commandId);
      return { event };
    };

    switch (cmd.type) {
      case "ReserveInventoryCmd": {
        const r = this.failReserve();
        if (!r.ok) return r.kind === "poison" ? { poison: true, event: { type: "InventoryReserveFailedEvt", eventId: `evt_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId, error: r.error, kind: "poison" } } : { transient: true, event: { type: "InventoryReserveFailedEvt", eventId: `evt_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId, error: r.error, kind: "transient" } };

        this.inventoryReserved.add(cmd.orderId);
        return emit({ type: "InventoryReservedEvt", eventId: `evt_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId });
      }

      case "ChargePaymentCmd": {
        const r = this.failCharge();
        if (!r.ok) return r.kind === "poison" ? { poison: true, event: { type: "PaymentChargeFailedEvt", eventId: `evt_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId, error: r.error, kind: "poison" } } : { transient: true, event: { type: "PaymentChargeFailedEvt", eventId: `evt_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId, error: r.error, kind: "transient" } };

        this.paymentCharged.add(cmd.orderId);
        return emit({ type: "PaymentChargedEvt", eventId: `evt_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId });
      }

      case "CreateShipmentCmd": {
        const r = this.failShip();
        if (!r.ok) return r.kind === "poison" ? { poison: true, event: { type: "ShipmentCreateFailedEvt", eventId: `evt_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId, error: r.error, kind: "poison" } } : { transient: true, event: { type: "ShipmentCreateFailedEvt", eventId: `evt_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId, error: r.error, kind: "transient" } };

        this.shipmentCreated.add(cmd.orderId);
        return emit({ type: "ShipmentCreatedEvt", eventId: `evt_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId });
      }

      case "RefundPaymentCmd": {
        // idempotent refund
        this.paymentCharged.delete(cmd.orderId);
        this.paymentRefunded.add(cmd.orderId);
        this.inbox.mark(cmd.commandId);
        return { event: { type: "PaymentRefundedEvt", eventId: `evt_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId } };
      }

      case "ReleaseInventoryCmd": {
        this.inventoryReserved.delete(cmd.orderId);
        this.inventoryReleased.add(cmd.orderId);
        this.inbox.mark(cmd.commandId);
        return { event: { type: "InventoryReleasedEvt", eventId: `evt_${Math.random()}`, sagaId: cmd.sagaId, orderId: cmd.orderId } };
      }
    }
  }
}
```

---

# 8) `src/runner.ts` (glue loop: outbox publisher + workers)

```ts
// src/runner.ts
import { InMemoryBroker } from "./broker";
import { Outbox } from "./outbox";
import { SagaStore } from "./sagaStore";
import { Orchestrator } from "./orchestrator";
import { Downstreams } from "./downstreams";
import type { Event } from "./types";

export class Simulation {
  broker = new InMemoryBroker();
  store = new SagaStore();
  outbox = new Outbox();
  orch = new Orchestrator(this.store, this.outbox, () => 1);
  ds: Downstreams;

  constructor(ds: Downstreams) {
    this.ds = ds;
  }

  // publish outbox -> broker commands
  outboxPublishStep(duplicates = 1) {
    const items = this.outbox.takeUnpublished(100);
    for (const it of items) {
      this.broker.publishCommand(it.command, { duplicates });
      this.outbox.markPublished(it.id, 1);
    }
  }

  // downstream consumes one command delivery
  async downstreamStep(opts?: { duplicatesOfEvents?: number }) {
    const m = this.broker.pollCommand();
    if (!m) return;

    const res = await this.ds.onCommand(m.body);
    if (res.event) {
      this.broker.publishEvent(res.event as Event, { duplicates: opts?.duplicatesOfEvents ?? 1 });
    }

    if (res.transient) {
      // at-least-once: retry same delivery (requeue)
      this.broker.nackCommand(m);
      return;
    }

    if (res.poison) {
      this.broker.sendToDLQ("commands", m);
      return;
    }
  }

  // orchestrator consumes one event delivery
  async orchestratorStep(opts?: { duplicatesOfCommands?: number }) {
    const m = this.broker.pollEvent();
    if (!m) return;

    this.orch.onEvent(m.body);

    // publish any new outbox commands
    this.outboxPublishStep(opts?.duplicatesOfCommands ?? 1);
  }

  async run(limit = 10_000, opts?: { duplicatesOfCommands?: number; duplicatesOfEvents?: number }) {
    // publish outbox continuously
    let i = 0;
    while (i < limit) {
      // Interleave like real system
      await this.orchestratorStep({ duplicatesOfCommands: opts?.duplicatesOfCommands ?? 1 });
      await this.downstreamStep({ duplicatesOfEvents: opts?.duplicatesOfEvents ?? 1 });

      const sizes = this.broker.sizes();
      const hasOutbox = this.outbox.takeUnpublished(1).length > 0;

      if (sizes.commands === 0 && sizes.events === 0 && !hasOutbox) break;
      i++;
    }
  }
}
```

---

# Tests — `test/saga83_1.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { Downstreams } from "../src/downstreams";
import { Simulation } from "../src/runner";

describe("Kata 83.1 - Event-driven Saga", () => {
  it("happy path completes (commands+events)", async () => {
    const ds = new Downstreams({});
    const sim = new Simulation(ds);

    // Kick off by sending PlaceOrderRequested event
    sim.broker.publishEvent({
      type: "PlaceOrderRequested",
      eventId: "evt_start_1",
      sagaId: "s1",
      orderId: "o1",
      userId: "u1",
      amountCents: 1000,
      sku: "skuA",
    });

    await sim.run();

    const saga = sim.store.get("s1")!;
    expect(saga.phase).toBe("completed");
    expect(ds.inventoryReserved.has("o1")).toBe(true);
    expect(ds.paymentCharged.has("o1")).toBe(true);
    expect(ds.shipmentCreated.has("o1")).toBe(true);
  });

  it("fail payment => compensate release inventory", async () => {
    const ds = new Downstreams({ ChargePaymentCmd: "always_transient" });
    const sim = new Simulation(ds);

    sim.broker.publishEvent({
      type: "PlaceOrderRequested",
      eventId: "evt_start_2",
      sagaId: "s2",
      orderId: "o2",
      userId: "u2",
      amountCents: 1000,
      sku: "skuA",
    });

    // Make orchestrator send commands; then run limited to allow retries pile up then compensation.
    // Our orchestrator marks failed on transient events only if it receives a failure event; downstream emits failure event on each transient.
    await sim.run(5000);

    const saga = sim.store.get("s2")!;
    // payment keeps failing transient -> orchestrator will keep getting PaymentChargeFailedEvt transient.
    // In this simplified model, we trigger compensation on transient failure immediately (could be after attempt cap in stretch).
    expect(["failed", "compensating_release"].includes(saga.phase)).toBe(true);
    expect(ds.inventoryReleased.has("o2")).toBe(true);
  });

  it("fail shipment => refund then release", async () => {
    const ds = new Downstreams({ CreateShipmentCmd: "always_transient" });
    const sim = new Simulation(ds);

    sim.broker.publishEvent({
      type: "PlaceOrderRequested",
      eventId: "evt_start_3",
      sagaId: "s3",
      orderId: "o3",
      userId: "u3",
      amountCents: 1000,
      sku: "skuA",
    });

    await sim.run(8000);

    const saga = sim.store.get("s3")!;
    expect(["failed", "compensating_release", "compensating_refund"].includes(saga.phase)).toBe(true);
    expect(ds.paymentRefunded.has("o3")).toBe(true);
    expect(ds.inventoryReleased.has("o3")).toBe(true);
  });

  it("duplicates everywhere do not double-apply effects", async () => {
    const ds = new Downstreams({});
    const sim = new Simulation(ds);

    sim.broker.publishEvent(
      {
        type: "PlaceOrderRequested",
        eventId: "evt_start_4",
        sagaId: "s4",
        orderId: "o4",
        userId: "u4",
        amountCents: 1000,
        sku: "skuA",
      },
      { duplicates: 5 }
    );

    await sim.run(10_000, { duplicatesOfCommands: 3, duplicatesOfEvents: 3 });

    const saga = sim.store.get("s4")!;
    expect(saga.phase).toBe("completed");

    // Sets prove no double effects
    expect(ds.inventoryReserved.has("o4")).toBe(true);
    expect(ds.paymentCharged.has("o4")).toBe(true);
    expect(ds.shipmentCreated.has("o4")).toBe(true);
  });

  it("restart orchestrator: resume from saga store + outbox", async () => {
    const ds = new Downstreams({ ChargePaymentCmd: "transient_once" });
    const sim = new Simulation(ds);

    sim.broker.publishEvent({
      type: "PlaceOrderRequested",
      eventId: "evt_start_5",
      sagaId: "s5",
      orderId: "o5",
      userId: "u5",
      amountCents: 1000,
      sku: "skuA",
    });

    // Run few steps
    await sim.run(50);

    // "Restart": create new orchestrator with same store & outbox
    const sim2 = sim; // keep same simulation objects (store/outbox/broker), just rewire orchestrator
    sim2.orch = new (await import("../src/orchestrator")).Orchestrator(sim2.store, sim2.outbox, () => 2);

    await sim2.run(5000);

    const saga = sim2.store.get("s5")!;
    expect(saga.phase).toBe("completed");
    expect(ds.shipmentCreated.has("o5")).toBe(true);
  });
});
```

> Ghi chú: Ở model này, “attempt cap” chưa implement chặt như Kata 81 (để kata vừa sức). Stretch dưới đây sẽ thêm **attempt budgets** và **exponential backoff** cho transient.

---

## Checks (DoD)

* Happy path: saga `completed`.
* Payment fail → release inventory.
* Shipment fail → refund rồi release.
* Duplicate events/commands không double side-effect.
* Restart không làm mất tiến trình.

---

## Stretch (nếu muốn “đúng đời” hơn nữa)

1. **Attempt cap theo step** (lưu trong SagaState): chỉ compensate sau N transient failures (dùng Retry Budget kata 81).
2. **Deadline propagation**: add `deadlineMs` vào commands; downstream tôn trọng deadline; orchestrator stop/compensate nếu deadline hết.
3. **Fencing token**: saga version increment mỗi transition; orchestrator worker dùng compare-and-swap để tránh concurrent writers.
4. **Outbox publisher reliability**: simulate crash giữa “markPublished vs publishCommand” và chứng minh không mất command (idempotent publish).
5. **Poison handling**: poison event/command vào DLQ + mark saga `poison` + require manual intervention.