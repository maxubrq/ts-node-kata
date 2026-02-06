# Kata 97 — Testing Pyramid in Node: Unit / Integration / Contract / E2E

## Goal (Senior+)

Bạn sẽ tạo một service Node nhỏ nhưng “production-real”, rồi xây **4 lớp test** đúng vai trò:

1. **Unit**: nhanh, cô lập, cover logic lõi (đặc biệt edge-cases).
2. **Integration**: test với DB/queue thật (docker compose) + migrations.
3. **Contract**: đảm bảo interface giữa service và consumer/downstream không drift.
4. **E2E**: test hành trình người dùng qua HTTP như thật.

Cuối cùng bạn phải có:

* test pyramid ratio rõ
* gating rules (CI)
* anti-flake playbook

---

## Context (service cố định)

Bạn xây service `Orders API` tối thiểu:

### Endpoints

1. `POST /orders`

   * idempotency key bắt buộc (header `Idempotency-Key`)
   * validate body
   * tạo order + persist
   * emit event `OrderCreated`
2. `GET /orders/:id`

   * return order

### Data

* Postgres (hoặc sqlite nếu bạn không muốn docker, nhưng production-grade khuyến nghị Postgres)
* Table: `orders(id, tenant_id, user_id, total, status, idempotency_key, created_at)`
* Unique constraint: `(tenant_id, idempotency_key)`

### Event

* Outbox table hoặc fake event bus (tùy bạn), nhưng phải có **contract test** cho event schema.

---

## Constraints

* ✅ TypeScript + Node.
* ✅ Test runner: `vitest` (khuyến nghị) hoặc `jest`.
* ✅ Mỗi layer phải có folder riêng + naming convention.
* ✅ Không “test trùng” vô ích: mỗi layer có scope rõ.
* ✅ Có seed/time control để deterministic.
* ✅ Có CI gating spec (README).
* ❌ Không dùng E2E để test business logic (đó là sai pyramid).
* ❌ Không mock DB ở integration.

---

## Deliverables

```
packages/kata-97/
  src/
    domain/
      money.ts
      order.ts
      errors.ts
    app/
      createOrder.ts
      getOrder.ts
    infra/
      db.ts
      orderRepo.ts
      outboxRepo.ts
      eventBus.ts
    http/
      server.ts
      routes.ts
      validation.ts
  test/
    unit/
    integration/
    contract/
    e2e/
  docker-compose.yml
  README.md
```

Bạn giao:

1. Service chạy được
2. 4 lớp test chạy được
3. README: pyramid strategy + gating + flake rules

---

# Part 1 — Test Taxonomy (bắt buộc ghi trong README)

Bạn phải ghi bảng này:

| Layer       | Purpose                           | Runs in CI                                  | Speed   | Mocks allowed         | Typical failures      |
| ----------- | --------------------------------- | ------------------------------------------- | ------- | --------------------- | --------------------- |
| Unit        | business logic, pure functions    | always                                      | ms      | yes                   | logic bug             |
| Integration | DB/outbox wiring, SQL constraints | always                                      | seconds | limited               | schema/migration bug  |
| Contract    | schema/events/API compatibility   | always                                      | fast    | no (schema is source) | drift/breaking change |
| E2E         | full user journey                 | nightly / pre-release (hoặc always nếu nhỏ) | slowest | no                    | config/env/routing    |

---

# Part 2 — Implement minimal domain logic (để unit tests có “đất”)

## Domain rules (đủ để unit test chất)

* `total` phải > 0
* status init = `CREATED`
* id generation deterministic in tests (inject clock/id provider)
* idempotency:

  * cùng `(tenantId, idemKey)` → trả lại order cũ, không tạo mới
  * body khác mà idemKey trùng → trả error 409 (bạn quyết định contract)

---

# Part 3 — Unit tests (fast, lots)

## Target

* `domain/money.ts`: parse/round
* `domain/order.ts`: creation rules
* `app/createOrder.ts`: orchestration logic với repo mocked

### Unit requirements

* property-ish tests nhẹ: random totals, ensure invariants
* test error mapping: validation error vs conflict

**Definition of Done (Unit)**

* chạy < 1s
* cover edge cases: 0, negative, huge number, missing userId, etc.

---

# Part 4 — Integration tests (DB real)

## Must include

1. **Migration + schema constraints**

   * unique `(tenant_id, idempotency_key)` hoạt động
2. **Transaction behavior**

   * create order + outbox write atomic
3. **Query correctness**

   * get by id returns same data

### Setup

* docker compose Postgres
* test harness:

  * create schema
  * truncate tables per test (transaction rollback ok nếu bạn làm tốt)

**Definition of Done (Integration)**

* chạy < ~20–40s locally
* không mock DB

---

# Part 5 — Contract tests (API + Event schema)

Bạn chọn 1 trong 2 (hoặc làm cả 2):

## Option A — Event contract (khuyến nghị)

Define schema `OrderCreated`:

```ts
type OrderCreated = {
  type: "OrderCreated";
  version: 1;
  tenantId: string;
  orderId: string;
  userId: string;
  total: number;
  createdAt: number;
};
```

Contract tests phải:

* validate event emitted matches schema
* versioning rules: adding optional fields ok, removing required fields fails

## Option B — HTTP contract

* publish OpenAPI spec hoặc Zod schema
* contract test đảm bảo route handler output matches schema

**Definition of Done (Contract)**

* test fail nếu breaking change xảy ra
* chạy nhanh (no DB required nếu event-only)

---

# Part 6 — E2E tests (full journey)

E2E flow:

1. start server (random port)
2. `POST /orders` (with idem key)
3. `POST /orders` same idem key → same order id
4. `GET /orders/:id` returns created order

E2E must include:

* real HTTP stack
* real DB (tái sử dụng docker)
* no internal imports (treat as blackbox)

**Definition of Done (E2E)**

* deterministic
* cleanup robust
* timeouts set

---

# Part 7 — CI gating (bắt buộc)

Bạn phải đưa rule cụ thể:

* PR:

  * unit + integration + contract
* main merge:

  * unit + integration + contract + (optional) smoke e2e
* nightly:

  * full e2e suite + stress (optional)

Cộng thêm:

* retry policy: **0 retries** mặc định, nếu flake → fix root cause
* test timeouts: explicit

---

# Starter checklist (copy làm theo)

### Repo conventions

* [ ] `test/unit/**/*.spec.ts`
* [ ] `test/integration/**/*.spec.ts`
* [ ] `test/contract/**/*.spec.ts`
* [ ] `test/e2e/**/*.spec.ts`

### Unit

* [ ] money/order create validations
* [ ] createOrder orchestrator with mocked repo/event

### Integration

* [ ] migration runs
* [ ] unique idem constraint
* [ ] outbox atomic

### Contract

* [ ] OrderCreated schema test
* [ ] versioning test (breaking vs non-breaking)

### E2E

* [ ] POST/GET flows
* [ ] idempotency behavior
* [ ] environment isolation

---

# Flake Killer rules (bắt buộc implement ít nhất 3)

1. **Freeze time**: inject clock provider
2. **Random seed fixed**: deterministic data generator
3. **No shared DB state**: truncate/transaction per test
4. **No port collisions**: bind random port
5. **No network dependency**: all local docker only

---

# Stretch (Senior+ → Staff-grade)

Chọn ≥ 6:

1. **Contract as artifact**: generate JSON schema and commit it; diff in CI.
2. **Mutation testing mindset**: thử cố ý bug 1 rule, xem unit test bắt được không.
3. **Test impact analysis**: tag tests theo risk area; run selective.
4. **Consumer-driven contract**: mock consumer expectations (Pact-like đơn giản).
5. **Hermetic integration**: run Postgres in testcontainers-like (tự viết wrapper).
6. **SLO tests**: add perf smoke (p95 < X) for create order.
7. **Chaos e2e**: simulate DB down; ensure health/readiness behavior.

---

## Output expectation (bạn phải tạo)

Trong README, cuối cùng phải có:

* “Pyramid ratio” bạn chọn (ví dụ 70% unit / 20% integration / 8% contract / 2% e2e)
* Tổng thời gian chạy từng layer
* Khi nào thêm test ở layer nào (decision rules)