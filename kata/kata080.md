# Kata 80 — Migration: Mono → Services: Carve-out Plan cho 1 Module

## Goal

1. Chọn **1 module** trong monolith (từ Kata 75) và thiết kế kế hoạch **carve-out** thành service riêng:

   * minimal disruption
   * rollback được
   * không “big bang”
2. Có **contract tests** giữ behavior (Kata 79 style).
3. Có **data migration strategy** (expand/contract), và **event compatibility** (Kata 76/77).
4. Có **cutover plan**: canary, metrics, SLO, kill switch.

---

## Context (bài toán)

Monolith có modules: `identity`, `wallet`, `transfer`, `notification`, `analytics`.

Bạn sẽ carve-out **`notification`** thành service `notification-service` (đây là easiest/realistic vì event-driven).

* Trước: `notification` chạy in-process, subscribe internal bus.
* Sau: `notification-service` consume event từ broker (giả lập), gửi receipt.

Mục tiêu: **transfer vẫn hoạt động** trong lúc migrate, và notification không mất event.

---

## Constraints

* ✅ Không thay đổi public API của `transfer` module (consumer không sửa).
* ✅ Không được “dừng hệ thống để migrate”.
* ✅ Phải có **Strangler/Facade**: monolith có thể route sang service mới theo flag.
* ✅ Phải có **backward-compatible events**: `MoneyTransferred.v1` (schema locked).
* ✅ Phải có **at-least-once** + **idempotent consumer** (notification).
* ✅ Phải có **observability + runbook**: metrics, logs, alerts, rollback steps.
* ✅ Deliverable có cả **plan + artifacts** (config, endpoints, tests).

---

## Deliverables (repo shape)

Tạo workspace 2 app:

```
kata-80/
  monolith/
    src/
      modules/transfer/...
      modules/notification/...
      infra/
        eventPublisher.ts
        outboxRepo.ts
      main/
        compositionRoot.ts
        config.ts
    test/
      contracts.transfer.test.ts
      outbox.publisher.test.ts
      migration.strangler.test.ts

  services/
    notification-service/
      src/
        consumer.ts
        handler.ts
        idempotencyStore.ts
        http/health.ts
        config.ts
      test/
        consumer.idempotency.test.ts
        contract.event-schema.test.ts
        e2e.consume.test.ts

  contracts/
    events/
      moneyTransferred.v1.ts
      jsonschema.moneyTransferred.v1.json (optional)
    http/
      notification-api.v1.ts (optional)

  docs/
    carveout-plan.md
    runbook-notification.md
    rollback-plan.md
    cutover-checklist.md
```

> “contracts/” là chỗ **khóa giao ước** giữa mono và service mới.

---

## Phần 1 — Chọn chiến lược migration

### Chiến lược chuẩn cho carve-out notification

* **Strangler Fig Pattern** cho traffic/behavior.
* **Outbox Pattern** trong monolith: khi transfer commit → ghi event vào outbox (DB) → publisher đẩy ra broker.
* `notification-service` consume từ broker → idempotent 처리.

**Tại sao chọn notification trước?**

* Coupling thấp (nghe event).
* Data ownership đơn giản (không cần shared DB).
* Rollback dễ (chỉ cần tắt consumer/flag).

---

## Phần 2 — Define contracts (bắt buộc làm trước)

### `contracts/events/moneyTransferred.v1.ts`

```ts
export type MoneyTransferredV1 = {
  type: "MoneyTransferred.v1";
  id: string;                 // event id (unique)
  occurredAt: string;         // ISO
  payload: {
    transferId: string;
    userId: string;
    amountCents: number;
    fromWalletId: string;
    toWalletId: string;
    currency: "USD";
  };
  meta: {
    traceId?: string;
    source: "monolith";
  };
};
```

**Rules**

* Không đổi field names trong v1.
* Nếu cần đổi → tạo `v2` và support song song.

### Contract test schema (mono & service đều phải pass)

* mono publish event đúng shape
* service parse event strict, reject invalid

---

## Phần 3 — Monolith changes (Expand phase)

### 3.1 Add Outbox table/repo (mô phỏng)

* `outbox_events`:

  * `id`, `type`, `payload_json`, `occurred_at`, `published_at nullable`, `attempts`, `last_error`
* Khi transfer thành công: `appendOutbox(event)` trong transaction.

### 3.2 Add Publisher worker

* Poll outbox every N ms
* Publish to “broker”
* Mark `published_at`
* Retry transient (with backoff), poison → DLQ/park.

### 3.3 Add Strangler switch

Monolith có `NotificationDispatcher` interface:

* Old path: `InProcessNotification` (internal module)
* New path: `RemoteNotificationService` (HTTP call) *hoặc* “none” (chỉ publish event, service tự consume)

**Best practice cho notification**: **không gọi HTTP**, chỉ event. Strangler ở đây là:

* Phase 1: publish event + vẫn chạy in-process notification (dual-run)
* Phase 2: disable in-process notification (flag), chỉ service mới làm

---

## Phần 4 — Service mới (notification-service)

### 4.1 Consumer

* Consume `MoneyTransferred.v1`
* Idempotency:

  * store processed event ids (TTL dài hoặc forever)
  * nếu duplicate → ack, không gửi lại receipt
* Handler “send receipt” (mock) + log/metrics

### 4.2 Health/Readiness

* `/health` (process alive)
* `/ready` (connected to broker, idempotency store ok)

### 4.3 Observability

* metrics:

  * `events_consumed_total`
  * `events_deduped_total`
  * `handler_fail_total`
  * `consumer_lag_seconds` (approx)
* logs with `traceId`, `eventId`, `transferId`

---

## Phần 5 — Cutover plan (phần làm “Senior+”)

### Stage 0 — Baseline (before change)

* Capture current behavior:

  * contract tests for transfer + notification output (in monolith)
  * define “receipt sent” semantics (exactly once from user POV)

### Stage 1 — Expand: publish events (no behavior change)

* Add outbox + publisher
* Keep in-process notification ON
* Deploy
* Verify:

  * outbox size stable
  * publish success rate
  * no impact latency on transfer (p95 unchanged or acceptable)

### Stage 2 — Shadow consume (service runs but doesn’t send)

* Run notification-service consuming events but **dry-run**:

  * only validate + log
* Verify parsing, throughput, lag

### Stage 3 — Dual-run (send in both, but only one “counts”)

* Option A: keep mono sending real receipts, service sends to “sandbox channel”
* Option B: service sends real, mono switched to “no-op” but logs
* Verify:

  * dedupe works
  * no double receipts in real channel

### Stage 4 — Canary cutover

* Flag: `NOTIFICATION_MODE = service` for 1% tenants/users (or by user hash)
* Monitor:

  * receipt success rate
  * lag
  * duplicate rate
* Ramp: 1% → 10% → 50% → 100%

### Stage 5 — Contract: disable old module

* Turn off in-process notification
* Remove code later (contract phase, not now)

---

## Failure model + rollback (bắt buộc viết)

### Failure cases

1. Broker down → outbox grows
2. Service down → lag grows, outbox publish still ok
3. Handler fail (email provider) → retry + DLQ
4. Duplicate events → must not send duplicate receipt

### Rollback steps (đơn giản, an toàn)

* Flip flag `NOTIFICATION_MODE=monolith` (re-enable in-process)
* Stop notification-service consumer (or disable sending)
* Keep outbox publisher running (optional) nhưng you can pause if needed

**Invariant:** rollback không làm mất money transfer.

---

## Các bài Task cụ thể (nhìn vào là làm)

### Task A — Contracts first

* Tạo `MoneyTransferred.v1` type + strict parser
* Viết `contract.event-schema.test.ts` ở cả mono và service

### Task B — Outbox in monolith

* Implement `OutboxRepo.append(event)` within transfer transaction
* Implement `OutboxPublisher.tick()` publish + mark

### Task C — Notification service consumer

* Implement `consumeLoop()`:

  * fetch batch events
  * for each: `handle(event)` with idempotency
* Implement `IdempotencyStore` (memory ok, stretch: sqlite/redis mock)

### Task D — Strangler / feature flag

* Config:

  * `NOTIFICATION_SEND_MODE = "monolith" | "service" | "dual" | "dry_run"`
* Implement behavior based on mode

### Task E — Cutover checklist + runbook docs

* `docs/cutover-checklist.md` (step-by-step)
* `docs/runbook-notification.md` (alerts + commands + what to check)
* `docs/rollback-plan.md`

---

## Tests (bắt buộc)

### 1) Contract tests (mono)

`monolith/test/outbox.publisher.test.ts`

* when transfer succeeds → outbox has event with correct schema
* publisher publishes and marks published_at
* retry on transient publish failure

### 2) Contract tests (service)

`services/notification-service/test/contract.event-schema.test.ts`

* parse valid event ok
* reject invalid (missing payload fields)

### 3) Idempotency test (service)

`consumer.idempotency.test.ts`

* same event delivered twice → handler called once

### 4) Migration strangler test (mono)

`migration.strangler.test.ts`

* mode=monolith → in-process handler called
* mode=service → in-process not called, outbox publish occurs
* mode=dual → both path invoked but real-send only one (you define)

### 5) End-to-end simulated

`services/notification-service/test/e2e.consume.test.ts`

* monolith writes outbox → publisher pushes to fake broker → service consumes → “receipt sent” recorded

---

## Checks (Definition of Done)

* ✅ Có `contracts/events/moneyTransferred.v1` + tests lock schema.
* ✅ Monolith publish event via outbox, không ảnh hưởng transfer semantics.
* ✅ notification-service consume at-least-once + idempotent.
* ✅ Có feature flag modes + canary plan + rollback plan.
* ✅ Có runbook với metrics/alerts tối thiểu.

---

## Stretch (Senior+ → rất cao)

1. **Data ownership & future carve-out**
   Chọn module khó hơn (wallet/transfer) và làm dual-write + read shadow.
2. **Backfill**
   Service mới cần state ban đầu → backfill from monolith DB + reconcile.
3. **SLO-based cutover gate**
   Tự động dừng ramp nếu error budget vượt.
4. **Event versioning**
   Support v1 + v2 concurrently, with compatibility tests.
5. **Chaos cutover drill**
   Inject broker outage, service crash loop, verify rollback and no data loss.