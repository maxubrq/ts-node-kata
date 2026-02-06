# Kata 76 — Event-driven Module: Internal Typed Event Bus

## Goal

1. Xây **internal event bus typed** cho monolith/modules:

   * event name → payload type (no `any`)
   * subscribe/publish trong process
2. Thiết kế **event-driven module**:

   * module A phát event, module B nghe event
   * giảm coupling: không gọi trực tiếp service của nhau
3. Có **reliability policy**:

   * sync vs async handlers
   * error handling (fail-fast vs isolate)
   * ordering per key (per user/transfer)
4. Có **internal outbox** (tối giản) để bảo đảm “state change + event” không bị lệch.

---

## Context

Trong Kata 75 bạn có flow:

* `transfer` cập nhật wallet + tạo transfer record
* `notification` gửi biên nhận

Giờ bạn nâng cấp:

* `transfer` **không gọi** notification
* `notification` nghe event `MoneyTransferred`
* Thêm module `analytics` nghe event và cập nhật projection (ví dụ tổng số tiền chuyển theo user)

Yêu cầu mới:

* `MoneyTransferred` handlers chạy **async** và **không làm hỏng** transfer (transfer đã commit rồi)
* Nhưng event phải **không mất** nếu process crash ngay sau commit → dùng **internal outbox queue** (in-memory cho kata; stretch: persist).

---

## Constraints

* ✅ Typed event map: compile-time guarantee đúng payload
* ✅ Bus support:

  * `on(event, handler)`
  * `emit(event, payload, meta?)`
  * `emitAsync` queue-based (recommended)
* ✅ Error policy:

  * handler lỗi không được làm crash emit caller
  * lỗi phải được capture (log + dead-letter list)
* ✅ Ordering:

  * events cùng `key` (vd `userId`) phải xử lý **in order**
  * events khác key có thể parallel (có concurrency cap)
* ✅ Outbox:

  * module phát event không gọi bus trực tiếp
  * module ghi event vào outbox trong transaction boundary (mô phỏng)
  * dispatcher đọc outbox và emit
* ✅ Tests:

  * type-level compile checks
  * runtime tests for ordering, isolation, DLQ

---

## Deliverables (repo shape)

```
kata-76/
  src/
    shared/
      typedEventBus.ts
      outbox.ts
      dispatcher.ts
      keyedQueue.ts
    modules/
      transfer/
        index.ts
        internal/
          transferService.ts
      notification/
        index.ts
        internal/
          receiptHandler.ts
      analytics/
        index.ts
        internal/
          projection.ts
    main/
      compositionRoot.ts
  test/
    bus.types.test-d.ts        // compile-time checks (or ts-expect-error in .ts)
    bus.ordering.test.ts
    bus.errorIsolation.test.ts
    outbox.dispatcher.test.ts
  README.md
```

---

## Event contract (Spec)

### Event Map

Bạn phải có typed map như sau:

```ts
export type InternalEvents = {
  "transfer.moneyTransferred": {
    transferId: string;
    userId: string;
    amountCents: number;
    fromWalletId: string;
    toWalletId: string;
    createdAt: string; // ISO
  };

  "notification.receiptSent": {
    transferId: string;
    userId: string;
    channel: "email";
    at: string;
  };

  "analytics.userTransferUpdated": {
    userId: string;
    totalTransferredCents: number;
    count: number;
  };
};
```

Meta (optional):

```ts
export type EventMeta = {
  key?: string;           // for ordering (use userId)
  traceId?: string;
  causedBy?: string;
};
```

---

## Starter skeleton

### `src/shared/typedEventBus.ts`

```ts
export type Handler<Payload, Meta> = (payload: Payload, meta: Meta) => Promise<void> | void;

export type EventBusOptions = {
  concurrency: number;     // max parallel handlers across different keys
  onError?: (e: { event: string; err: unknown; payload: unknown; meta: unknown }) => void;
};

export class TypedEventBus<Events extends Record<string, any>, Meta extends Record<string, any>> {
  private handlers = new Map<keyof Events, Handler<any, Meta>[]>();

  constructor(private opts: EventBusOptions) {}

  on<K extends keyof Events>(event: K, handler: Handler<Events[K], Meta>) {
    const list = this.handlers.get(event) ?? [];
    list.push(handler as any);
    this.handlers.set(event, list);
  }

  /** Sync-ish emit: calls handlers immediately (no ordering guarantee across async). Use for very local stuff. */
  emit<K extends keyof Events>(event: K, payload: Events[K], meta: Meta): void {
    const list = this.handlers.get(event) ?? [];
    for (const h of list) {
      try {
        const r = h(payload, meta);
        // fire-and-forget
        void r;
      } catch (err) {
        this.opts.onError?.({ event: String(event), err, payload, meta });
      }
    }
  }
}
```

> Bus này chưa đủ. Bạn phải làm `emitAsync` + ordering per key + DLQ. Phần đó nằm ở `keyedQueue.ts` + `dispatcher.ts`.

### `src/shared/keyedQueue.ts` (ordering engine)

```ts
type Task = () => Promise<void>;

export type KeyedQueueOptions = {
  concurrency: number;
  onTaskError?: (err: unknown) => void;
};

export class KeyedQueue {
  private queues = new Map<string, Task[]>();
  private runningKeys = new Set<string>();
  private runningCount = 0;

  constructor(private opts: KeyedQueueOptions) {}

  enqueue(key: string, task: Task) {
    const q = this.queues.get(key) ?? [];
    q.push(task);
    this.queues.set(key, q);
    void this.pump();
  }

  private async pump() {
    // TODO:
    // - while runningCount < concurrency:
    //   - pick a key that is not running and has tasks
    //   - run tasks sequentially for that key (ensure ordering)
    //   - tasks of different keys can run in parallel up to concurrency
    // - ensure errors are caught and reported, but do not stop the queue
  }
}
```

### `src/shared/outbox.ts`

```ts
export type OutboxItem<E extends string, P, M> = {
  id: string;
  event: E;
  payload: P;
  meta: M;
  createdAt: number;
};

export class InMemoryOutbox<E extends string, P, M> {
  private items: OutboxItem<E, P, M>[] = [];

  append(item: OutboxItem<E, P, M>) {
    this.items.push(item);
  }

  drain(batchSize: number): OutboxItem<E, P, M>[] {
    return this.items.splice(0, batchSize);
  }

  size() { return this.items.length; }
}
```

### `src/shared/dispatcher.ts`

```ts
import type { TypedEventBus } from "./typedEventBus";
import { KeyedQueue } from "./keyedQueue";

export type DispatcherOptions = {
  batchSize: number;
  defaultKey: string;
};

export class OutboxDispatcher<Events extends Record<string, any>, Meta extends Record<string, any>> {
  private dlq: { event: string; payload: unknown; meta: unknown; err: unknown }[] = [];
  private queue: KeyedQueue;

  constructor(
    private deps: {
      bus: TypedEventBus<Events, Meta>;
      outbox: { drain(n: number): { event: keyof Events; payload: any; meta: Meta }[]; size(): number };
      opts: DispatcherOptions;
      onDispatchError?: (x: any) => void;
    }
  ) {
    this.queue = new KeyedQueue({
      concurrency:  deps.opts.batchSize, // TODO: not ideal; you decide concurrency separately
      onTaskError: (err) => deps.onDispatchError?.(err),
    });
  }

  getDLQ() { return this.dlq.slice(); }

  tick() {
    const batch = this.deps.outbox.drain(this.deps.opts.batchSize);

    for (const item of batch) {
      const key = (item.meta as any)?.key ?? this.deps.opts.defaultKey;

      this.queue.enqueue(String(key), async () => {
        try {
          // TODO: call bus handlers and await them
          // You will need bus.emitAsync-like behavior; simplest:
          // - extend TypedEventBus with asyncEmit that awaits handlers
        } catch (err) {
          this.dlq.push({ event: String(item.event), payload: item.payload, meta: item.meta, err });
        }
      });
    }
  }
}
```

Bạn sẽ phải **nâng** `TypedEventBus`:

* thêm `emitAndWait(event, payload, meta): Promise<void>` để dispatcher await handlers
* dispatcher dùng `KeyedQueue` để bảo đảm ordering per meta.key

---

## Module implementation tasks

### `modules/transfer/internal/transferService.ts`

* Không gọi bus trực tiếp
* Ghi outbox item `transfer.moneyTransferred` khi transfer thành công

### `modules/notification/internal/receiptHandler.ts`

* Subscribe `transfer.moneyTransferred`
* “send receipt” (mô phỏng) rồi emit `notification.receiptSent` (có thể emit sync hoặc append outbox; bạn quyết định, nhưng ghi rõ)

### `modules/analytics/internal/projection.ts`

* Subscribe `transfer.moneyTransferred`
* Update projection:

  * totalTransferredCents += amount
  * count += 1
* Emit `analytics.userTransferUpdated` (optional)

---

## Tests (bắt buộc)

### 1) Ordering by key: `test/bus.ordering.test.ts`

* tạo outbox với 5 events cùng `meta.key="user1"` và payload index 1..5
* dispatcher tick
* handler push index vào array
  ✅ assert array đúng thứ tự 1..5

### 2) Parallel across keys with cap: `test/bus.ordering.test.ts` (phần 2)

* 2 keys user1/user2, mỗi key 3 tasks, handler delay 50ms
* concurrency=2
  ✅ expect total time ~150ms (không phải 300ms), nhưng mỗi key vẫn in order

### 3) Error isolation + DLQ: `test/bus.errorIsolation.test.ts`

* handler notification throws cho event #2
* dispatcher tick
  ✅ expect:

  * event #1 and #3 vẫn xử lý
  * DLQ có 1 item (event #2)
  * process không crash

### 4) Outbox reliability: `test/outbox.dispatcher.test.ts`

* simulate: transfer ghi outbox rồi “crash” (không chạy notification ngay)
* restart: create dispatcher mới, drain outbox
  ✅ expect receipt sent after tick

### 5) Type checks: `test/bus.types.test-d.ts` hoặc dùng `@ts-expect-error`

* `bus.on("transfer.moneyTransferred", (p) => p.userId)` OK
* `bus.emit("transfer.moneyTransferred", { wrong: 1 }, meta)` phải fail compile

---

## Checks (Definition of Done)

* ✅ Typed event map, không any.
* ✅ Outbox + dispatcher: state change + event không lệch (trong mô phỏng).
* ✅ Ordering per key guaranteed.
* ✅ Errors isolated, DLQ capture.
* ✅ Tests cover ordering/parallel/error/outbox/type-safety.

---

## Stretch (Senior+ → rất cao)

1. **Exactly-once illusion (internal)**
   Add idempotency in handlers: store processed event ids, ignore duplicates.
2. **Retry with backoff**
   Dispatcher retry DLQ items N lần trước khi “park”.
3. **Handler timeout budget**
   Nếu handler quá lâu → mark fail + DLQ (đừng treo queue).
4. **Event versioning**
   `transfer.moneyTransferred.v1` với upgrade path.
5. **Tracing propagation**
   Meta includes traceId, handler logs correlation chain.