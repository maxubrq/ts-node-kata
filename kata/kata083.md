# Kata 83 — Saga Orchestration: workflow + compensate

## Goal

1. Implement **Saga Orchestrator** theo model **state machine**:

   * Step1 → Step2 → Step3 (happy path)
   * Fail ở bước nào → chạy **compensations ngược** (Step(k-1)…Step1).
2. Orchestrator phải chịu được:

   * **at-least-once delivery** (command/event duplicate)
   * **crash/restart** giữa chừng (resume từ persisted state)
   * **transient errors** (retry với budget/cap)
   * **poison** (DLQ)
3. Proof bằng tests: không double-charge, không double-reserve, compensation đúng thứ tự.

---

## Context (bài toán thật)

Bạn cần workflow `PlaceOrder` gồm:

1. **ReserveInventory**
2. **ChargePayment**
3. **CreateShipment**

Nếu bước 2 fail sau khi reserve → phải **ReleaseInventory**.
Nếu bước 3 fail sau khi charge → phải **RefundPayment** rồi **ReleaseInventory**.

> Đây là “distributed transaction” phiên bản sống sót.

---

## Constraints

* ❌ Không dùng temporal/cadence libs. Tự viết orchestration.
* ✅ Saga state phải persisted (in-memory store mô phỏng DB) và **idempotent**:

  * command/event duplicate không làm chạy step 2 lần.
* ✅ Mỗi step có:

  * `action()` có thể fail transient/poison
  * `compensate()` có thể fail transient (cũng cần retry cap)
* ✅ Có retry cap per step + overall attempt guard.
* ✅ Có test bắt buộc:

  1. Happy path: done
  2. Fail step2 → compensate step1
  3. Fail step3 → compensate step2 rồi step1
  4. Duplicate delivery → không chạy action/compensate lặp
  5. Crash mid-way → resume đúng

---

## Deliverables

```
kata83/
  src/
    types.ts
    sagaStore.ts
    broker.ts
    steps.ts
    orchestrator.ts
  test/
    saga.test.ts
```

---

# Starter skeleton (copy là làm)

## 1) `src/types.ts`

```ts
// src/types.ts
export type SagaId = string;

export type PlaceOrderCommand = {
  type: "PlaceOrder";
  commandId: string; // idempotency key for command
  sagaId: SagaId;
  orderId: string;
  userId: string;
  amountCents: number;
  sku: string;
};

export type SagaCommand = PlaceOrderCommand;

export type StepName = "ReserveInventory" | "ChargePayment" | "CreateShipment";

export type StepStatus =
  | "not_started"
  | "in_progress"
  | "succeeded"
  | "failed"
  | "compensated"
  | "compensating";

export type SagaStatus = "running" | "completed" | "compensating" | "failed" | "poison";

export type StepState = {
  name: StepName;
  status: StepStatus;
  attempts: number;
  error?: string;
};

export type SagaState = {
  sagaId: SagaId;
  commandId: string;       // to dedup PlaceOrder
  status: SagaStatus;
  currentStepIndex: number; // pointer for forward progress
  steps: StepState[];
  createdAt: number;
  updatedAt: number;
  // For compensation pointer (which step to compensate next)
  compensateIndex: number; // starts at last succeeded step index
};
```

---

## 2) `src/sagaStore.ts` (persistence + idempotency)

```ts
// src/sagaStore.ts
import type { SagaId, SagaState, StepName } from "./types";

export class SagaStore {
  private bySaga = new Map<SagaId, SagaState>();
  private commandToSaga = new Map<string, SagaId>(); // commandId -> sagaId (dedup)

  get(sagaId: SagaId) {
    return this.bySaga.get(sagaId);
  }

  findByCommand(commandId: string): SagaState | undefined {
    const sagaId = this.commandToSaga.get(commandId);
    if (!sagaId) return;
    return this.bySaga.get(sagaId);
  }

  createIfAbsent(args: { sagaId: SagaId; commandId: string; now: number; steps: StepName[] }): SagaState {
    const existing = this.bySaga.get(args.sagaId);
    if (existing) return existing;

    // command dedup: if same commandId already used, return that saga
    const existingSagaId = this.commandToSaga.get(args.commandId);
    if (existingSagaId) {
      const s = this.bySaga.get(existingSagaId);
      if (s) return s;
    }

    const state: SagaState = {
      sagaId: args.sagaId,
      commandId: args.commandId,
      status: "running",
      currentStepIndex: 0,
      steps: args.steps.map((name) => ({ name, status: "not_started", attempts: 0 })),
      createdAt: args.now,
      updatedAt: args.now,
      compensateIndex: -1,
    };
    this.bySaga.set(args.sagaId, state);
    this.commandToSaga.set(args.commandId, args.sagaId);
    return state;
  }

  update(s: SagaState, now: number) {
    s.updatedAt = now;
    this.bySaga.set(s.sagaId, s);
  }
}
```

---

## 3) `src/broker.ts` (in-memory at-least-once)

```ts
// src/broker.ts
import type { SagaCommand } from "./types";

export type BrokerMessage = { deliveryId: string; command: SagaCommand };

export class InMemoryBroker {
  private q: BrokerMessage[] = [];
  private dlq: BrokerMessage[] = [];

  publish(command: SagaCommand, opts?: { duplicates?: number }) {
    const d = opts?.duplicates ?? 1;
    for (let i = 0; i < d; i++) {
      this.q.push({
        deliveryId: `d_${command.type}_${command.commandId}_${i}_${Math.random().toString(16).slice(2)}`,
        command,
      });
    }
  }

  poll(): BrokerMessage | undefined {
    return this.q.shift();
  }

  ack(_m: BrokerMessage) {}
  nack(m: BrokerMessage) {
    this.q.push(m);
  }

  sendToDLQ(m: BrokerMessage) {
    this.dlq.push(m);
  }

  dlqMessages() {
    return [...this.dlq];
  }

  size() {
    return this.q.length;
  }
}
```

---

## 4) `src/steps.ts` (downstream services simulation)

Mỗi step có action + compensate, và có failure injection.

```ts
// src/steps.ts
import type { PlaceOrderCommand, StepName } from "./types";

export type StepResult = { ok: true } | { ok: false; kind: "transient" | "poison"; error: string };

export type StepHandler = {
  name: StepName;
  action: (cmd: PlaceOrderCommand) => Promise<StepResult>;
  compensate: (cmd: PlaceOrderCommand) => Promise<StepResult>;
};

export type FailurePlan = Partial<Record<StepName, "none" | "transient_once" | "always_transient" | "poison">>;

function mkFailer(plan: FailurePlan, step: StepName) {
  let usedOnce = false;
  return (): StepResult => {
    const mode = plan[step] ?? "none";
    if (mode === "none") return { ok: true };
    if (mode === "poison") return { ok: false, kind: "poison", error: `${step} poison` };
    if (mode === "always_transient") return { ok: false, kind: "transient", error: `${step} transient` };
    if (mode === "transient_once") {
      if (!usedOnce) {
        usedOnce = true;
        return { ok: false, kind: "transient", error: `${step} transient_once` };
      }
      return { ok: true };
    }
    return { ok: true };
  };
}

export class Downstreams {
  // “real side effects” we can assert in tests
  inventoryReserved = new Set<string>(); // orderId
  paymentCharged = new Set<string>();    // orderId
  shipmentCreated = new Set<string>();   // orderId
  paymentRefunded = new Set<string>();   // orderId
  inventoryReleased = new Set<string>(); // orderId

  constructor(private readonly plan: FailurePlan = {}) {}

  handlers(): StepHandler[] {
    const inv = mkFailer(this.plan, "ReserveInventory");
    const pay = mkFailer(this.plan, "ChargePayment");
    const ship = mkFailer(this.plan, "CreateShipment");

    return [
      {
        name: "ReserveInventory",
        action: async (cmd) => {
          const r = inv();
          if (!r.ok) return r;
          this.inventoryReserved.add(cmd.orderId);
          return { ok: true };
        },
        compensate: async (cmd) => {
          // releasing inventory should be idempotent
          this.inventoryReserved.delete(cmd.orderId);
          this.inventoryReleased.add(cmd.orderId);
          return { ok: true };
        },
      },
      {
        name: "ChargePayment",
        action: async (cmd) => {
          const r = pay();
          if (!r.ok) return r;
          this.paymentCharged.add(cmd.orderId);
          return { ok: true };
        },
        compensate: async (cmd) => {
          // refund should be idempotent
          this.paymentCharged.delete(cmd.orderId);
          this.paymentRefunded.add(cmd.orderId);
          return { ok: true };
        },
      },
      {
        name: "CreateShipment",
        action: async (cmd) => {
          const r = ship();
          if (!r.ok) return r;
          this.shipmentCreated.add(cmd.orderId);
          return { ok: true };
        },
        compensate: async (_cmd) => {
          // shipment compensation might be “cancel shipment” in real life
          return { ok: true };
        },
      },
    ];
  }
}
```

---

## 5) `src/orchestrator.ts` (the core)

Orchestrator phải:

* dedup commandId
* progress step-by-step
* mark step state before/after action
* on failure: switch to compensating, run compensate in reverse
* retries capped per step (action + compensate)

```ts
// src/orchestrator.ts
import type { InMemoryBroker, BrokerMessage } from "./broker";
import type { SagaStore } from "./sagaStore";
import type { PlaceOrderCommand, SagaState, StepHandler, StepName } from "./types";

export type OrchestratorConfig = {
  maxAttemptsPerStep: number;     // cap retry for action/compensate
  maxTotalTicks?: number;         // safety in drain
};

export class SagaOrchestrator {
  private stepMap: Map<StepName, StepHandler>;

  constructor(
    private readonly broker: InMemoryBroker,
    private readonly store: SagaStore,
    handlers: StepHandler[],
    private readonly cfg: OrchestratorConfig,
    private readonly now: () => number = () => Date.now()
  ) {
    this.stepMap = new Map(handlers.map((h) => [h.name, h]));
  }

  async tick() {
    const msg = this.broker.poll();
    if (!msg) return;

    const cmd = msg.command as PlaceOrderCommand;

    // Create or load saga (command dedup)
    const saga = this.store.createIfAbsent({
      sagaId: cmd.sagaId,
      commandId: cmd.commandId,
      now: this.now(),
      steps: ["ReserveInventory", "ChargePayment", "CreateShipment"],
    });

    // If already terminal, ack duplicates
    if (saga.status === "completed" || saga.status === "failed" || saga.status === "poison") {
      this.broker.ack(msg);
      return;
    }

    const progressed = await this.runSaga(cmd, saga);

    this.store.update(progressed, this.now());

    // Decide ack/nack/dlq
    if (progressed.status === "poison") {
      this.broker.sendToDLQ(msg);
      this.broker.ack(msg);
      return;
    }

    // If still running/compensating, we can requeue for next tick (simulate async continuation)
    if (progressed.status === "running" || progressed.status === "compensating") {
      this.broker.nack(msg);
      return;
    }

    // completed/failed => ack
    this.broker.ack(msg);
  }

  private async runSaga(cmd: PlaceOrderCommand, saga: SagaState): Promise<SagaState> {
    if (saga.status === "running") {
      return this.forward(cmd, saga);
    }
    if (saga.status === "compensating") {
      return this.backward(cmd, saga);
    }
    return saga;
  }

  private async forward(cmd: PlaceOrderCommand, saga: SagaState): Promise<SagaState> {
    const idx = saga.currentStepIndex;
    if (idx >= saga.steps.length) {
      saga.status = "completed";
      return saga;
    }

    const step = saga.steps[idx];
    const handler = this.stepMap.get(step.name);
    if (!handler) {
      saga.status = "poison";
      step.status = "failed";
      step.error = "missing handler";
      return saga;
    }

    // Idempotency: if step already succeeded, skip forward
    if (step.status === "succeeded") {
      saga.currentStepIndex += 1;
      return saga;
    }

    // mark in progress
    step.status = "in_progress";
    step.attempts += 1;

    if (step.attempts > this.cfg.maxAttemptsPerStep) {
      // Switch to compensating
      step.status = "failed";
      step.error = "attempts_exhausted";
      saga.status = "compensating";
      saga.compensateIndex = idx - 1; // compensate all previous succeeded steps
      return saga;
    }

    const res = await handler.action(cmd);
    if (res.ok) {
      step.status = "succeeded";
      saga.currentStepIndex += 1;
      // Update compensate pointer: last succeeded step
      saga.compensateIndex = idx;
      return saga;
    }

    // poison => terminal poison
    if (res.kind === "poison") {
      step.status = "failed";
      step.error = res.error;
      saga.status = "poison";
      return saga;
    }

    // transient => keep running, will retry next tick
    step.status = "failed";
    step.error = res.error;
    return saga;
  }

  private async backward(cmd: PlaceOrderCommand, saga: SagaState): Promise<SagaState> {
    let idx = saga.compensateIndex;
    if (idx < 0) {
      saga.status = "failed"; // compensation done
      return saga;
    }

    const step = saga.steps[idx];
    const handler = this.stepMap.get(step.name);
    if (!handler) {
      saga.status = "poison";
      step.error = "missing handler for compensate";
      return saga;
    }

    // Only compensate steps that succeeded
    if (step.status === "compensated") {
      saga.compensateIndex -= 1;
      return saga;
    }

    if (step.status !== "succeeded" && step.status !== "compensating") {
      // if never succeeded, skip
      saga.compensateIndex -= 1;
      return saga;
    }

    // mark compensating
    step.status = "compensating";
    step.attempts += 1;

    if (step.attempts > this.cfg.maxAttemptsPerStep) {
      saga.status = "failed";
      step.error = "compensation_attempts_exhausted";
      return saga;
    }

    const res = await handler.compensate(cmd);
    if (res.ok) {
      step.status = "compensated";
      saga.compensateIndex -= 1;
      return saga;
    }

    if (res.kind === "poison") {
      saga.status = "poison";
      step.error = res.error;
      return saga;
    }

    // transient: keep compensating and retry next tick
    step.error = res.error;
    return saga;
  }

  async drain(limit = 10_000) {
    let i = 0;
    while (this.broker.size() > 0 && i < limit) {
      // eslint-disable-next-line no-await-in-loop
      await this.tick();
      i++;
    }
  }
}
```

> Đây là “single-threaded orchestrator” để test deterministic. Production sẽ là nhiều workers + DB locking, nhưng logic state machine giữ nguyên.

---

# Tests (vitest)

## `test/saga.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { InMemoryBroker } from "../src/broker";
import { SagaStore } from "../src/sagaStore";
import { Downstreams } from "../src/steps";
import { SagaOrchestrator } from "../src/orchestrator";
import type { PlaceOrderCommand } from "../src/types";

function cmd(overrides: Partial<PlaceOrderCommand> = {}): PlaceOrderCommand {
  return {
    type: "PlaceOrder",
    commandId: overrides.commandId ?? "cmd_1",
    sagaId: overrides.sagaId ?? "saga_1",
    orderId: overrides.orderId ?? "ord_1",
    userId: overrides.userId ?? "usr_1",
    amountCents: overrides.amountCents ?? 1000,
    sku: overrides.sku ?? "sku_A",
  };
}

describe("Kata 83 - Saga Orchestration", () => {
  it("happy path completes", async () => {
    const broker = new InMemoryBroker();
    const store = new SagaStore();
    const ds = new Downstreams({});
    const orch = new SagaOrchestrator(broker, store, ds.handlers(), { maxAttemptsPerStep: 3 }, () => 1);

    broker.publish(cmd(), { duplicates: 1 });
    await orch.drain();

    const s = store.get("saga_1")!;
    expect(s.status).toBe("completed");
    expect(ds.inventoryReserved.has("ord_1")).toBe(true);
    expect(ds.paymentCharged.has("ord_1")).toBe(true);
    expect(ds.shipmentCreated.has("ord_1")).toBe(true);
  });

  it("fail step2 => compensate step1 (release inventory)", async () => {
    const broker = new InMemoryBroker();
    const store = new SagaStore();
    const ds = new Downstreams({ ChargePayment: "always_transient" }); // will exceed attempts -> compensation
    const orch = new SagaOrchestrator(broker, store, ds.handlers(), { maxAttemptsPerStep: 2 }, () => 1);

    broker.publish(cmd({ sagaId: "saga_2", commandId: "cmd_2", orderId: "ord_2" }));
    await orch.drain();

    const s = store.get("saga_2")!;
    expect(s.status).toBe("failed");
    expect(ds.inventoryReleased.has("ord_2")).toBe(true);
    expect(ds.paymentRefunded.has("ord_2")).toBe(false);
  });

  it("fail step3 => compensate step2 then step1 (refund + release)", async () => {
    const broker = new InMemoryBroker();
    const store = new SagaStore();
    const ds = new Downstreams({ CreateShipment: "always_transient" });
    const orch = new SagaOrchestrator(broker, store, ds.handlers(), { maxAttemptsPerStep: 2 }, () => 1);

    broker.publish(cmd({ sagaId: "saga_3", commandId: "cmd_3", orderId: "ord_3" }));
    await orch.drain();

    const s = store.get("saga_3")!;
    expect(s.status).toBe("failed");
    expect(ds.paymentRefunded.has("ord_3")).toBe(true);
    expect(ds.inventoryReleased.has("ord_3")).toBe(true);
  });

  it("duplicate delivery does not double-run actions (idempotent progress)", async () => {
    const broker = new InMemoryBroker();
    const store = new SagaStore();
    const ds = new Downstreams({});
    const orch = new SagaOrchestrator(broker, store, ds.handlers(), { maxAttemptsPerStep: 3 }, () => 1);

    const c = cmd({ sagaId: "saga_4", commandId: "cmd_4", orderId: "ord_4" });
    broker.publish(c, { duplicates: 5 });

    await orch.drain();

    const s = store.get("saga_4")!;
    expect(s.status).toBe("completed");

    // effects are sets => must only contain ord_4 once (by definition)
    expect(ds.inventoryReserved.has("ord_4")).toBe(true);
    expect(ds.paymentCharged.has("ord_4")).toBe(true);
    expect(ds.shipmentCreated.has("ord_4")).toBe(true);
  });

  it("crash/restart resumes from persisted saga state", async () => {
    const broker = new InMemoryBroker();
    const store = new SagaStore();

    // Step2 transient once => requires multiple ticks
    const ds = new Downstreams({ ChargePayment: "transient_once" });

    const orch1 = new SagaOrchestrator(broker, store, ds.handlers(), { maxAttemptsPerStep: 3 }, () => 1);

    const c = cmd({ sagaId: "saga_5", commandId: "cmd_5", orderId: "ord_5" });
    broker.publish(c);

    // Run only 1 tick -> likely stuck before completion
    await orch1.tick();

    // "restart": new orchestrator, same store & broker
    const orch2 = new SagaOrchestrator(broker, store, ds.handlers(), { maxAttemptsPerStep: 3 }, () => 2);
    await orch2.drain();

    const s = store.get("saga_5")!;
    expect(s.status).toBe("completed");
    expect(ds.shipmentCreated.has("ord_5")).toBe(true);
  });
});
```

---

## Checks (Definition of Done)

* Workflow complete khi đủ 3 steps.
* Fail step2 → release inventory.
* Fail step3 → refund payment rồi release inventory.
* Duplicate command delivery không làm side-effect chạy lặp.
* Restart vẫn resume đúng.

---

# Senior+ Notes (điểm sắc)

* **State machine rõ ràng**: forward pointer + compensation pointer.
* **Idempotency nằm ở state**: step succeeded thì skip action; compensated thì skip compensate.
* **Retry cap per step**: không retry vô hạn, có điểm chuyển sang compensate.
* **Terminal poison**: không cố sửa “logic poison” bằng retry.

---

## Stretch (hard mode – đúng distributed)

1. **Outbox / Inbox**: Orchestrator phát command ra downstream bằng outbox, downstream trả event, orchestrator consume events (event-driven saga).
2. **Concurrency**: 2 workers xử lý cùng saga → cần DB lock / fencing token.
3. **Timeout-driven compensation**: step “in_progress” quá lâu => trigger compensate (watchdog).
4. **Partial compensation failure**: compensate step2 fail transient nhiều lần nhưng step1 đã compensate → saga stuck; thiết kế “manual intervention state”.
5. **SLO & metrics**:

   * `saga_duration_ms` histogram
   * `saga_compensations_total{step}`
   * `saga_failures_total{reason}`
6. **Exactly-once effects**: mỗi downstream action/compensate cũng phải có idempotency key riêng (commandId per step), không chỉ saga-level.