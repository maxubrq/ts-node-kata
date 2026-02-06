# Kata 71 — Hexagonal Boundary: Domain / Application / Infra tách đúng chuẩn

## Goal

1. Thiết kế 1 service theo **Hexagonal Architecture**:

   * **Domain**: business rules thuần (no DB/HTTP/time/system env).
   * **Application**: orchestration use-case (gọi ports, transaction, idempotency…).
   * **Infra**: adapters (DB, HTTP client, clock, logger…).
2. Enforce **dependency direction**:

   * `domain` **không import** `application/infra`.
   * `application` **chỉ phụ thuộc** `domain` + **ports**.
   * `infra` phụ thuộc `application` (để implement ports) + third-party.
3. Có **composition root** dựng wiring (DI) duy nhất ở entrypoint.
4. Có **test pyramid đúng**: domain unit, application integration (fake ports), infra contract tests.

---

## Bài toán (Context)

Bạn xây mini-service: **Wallet Transfer** (chuyển tiền giữa 2 ví).

Yêu cầu business (domain):

* Không được chuyển số âm / 0.
* Không được chuyển nếu ví nguồn thiếu tiền.
* Transfer tạo **TransferRecord** với trạng thái `COMPLETED`.
* **Idempotency**: cùng `idempotencyKey` → không tạo transfer mới (trả lại kết quả cũ).

Yêu cầu hệ thống (application):

* Use-case `TransferMoney`:

  * validate input
  * load wallets
  * apply domain logic
  * persist (atomic)
  * publish event `MoneyTransferred` (outbox style tối giản)
* Infra:

  * Repo in-memory (để test), và “DB adapter giả lập” (có thể vẫn in-memory nhưng tách module đúng).
  * Event publisher adapter.

---

## Constraints (ép trade-off đúng)

* ❌ Domain **không được**:

  * import `fs`, `process`, `Date.now`, HTTP, DB libs, logger libs
  * throw lỗi “random” từ infra (domain dùng **domain error types**)
* ✅ Mọi IO đi qua **ports** (interfaces) nằm ở `application/ports` (hoặc `domain/ports` nếu bạn prefer, nhưng thống nhất).
* ✅ Có **transaction boundary** ở application (port `UnitOfWork`).
* ✅ Idempotency phải nằm ở **application** (port `IdempotencyStore`).
* ✅ Enforce boundary bằng:

  * 1 file `dependency-rules.test.ts` (static import scan), hoặc eslint rule (tối thiểu test).
* ✅ Có tests:

  * domain unit tests (pure)
  * application tests dùng fake ports
  * infra contract test (adapter phải pass contract)

---

## Deliverables (repo shape)

Tạo package `kata-71/` với structure:

```
kata-71/
  src/
    domain/
      wallet.ts
      transfer.ts
      errors.ts
      events.ts
    application/
      ports/
        walletRepo.ts
        transferRepo.ts
        idempotencyStore.ts
        unitOfWork.ts
        eventBus.ts
        clock.ts
      usecases/
        transferMoney.ts
      dto.ts
    infra/
      memory/
        memoryWalletRepo.ts
        memoryTransferRepo.ts
        memoryIdempotencyStore.ts
        memoryUnitOfWork.ts
        memoryEventBus.ts
        systemClock.ts
      http/
        controller.ts
      compositionRoot.ts
  test/
    domain.wallet.test.ts
    application.transferMoney.test.ts
    infra.contract.walletRepo.test.ts
    dependency-rules.test.ts
  README.md
```

Tooling gợi ý (không bắt buộc): `vitest`, `tsx`, `eslint`.

---

## Domain model (spec)

### Entities / Value objects

* `Money` (integer cents) — không float.
* `Wallet { id, balance }`
* `Transfer { id, fromWalletId, toWalletId, amount, createdAt }`

### Domain errors

* `InvalidAmount`
* `InsufficientFunds`
* `SameWalletTransferNotAllowed` (optional)

### Domain function

* `applyTransfer(from: Wallet, to: Wallet, amount: Money): { fromNext, toNext }`
* domain emits event object (pure): `MoneyTransferredDomainEvent`

> Domain chỉ “tính toán & luật”. Persist / publish để application làm.

---

## Application (use-case) spec

### Input DTO

```ts
type TransferMoneyInput = {
  idempotencyKey: string;
  fromWalletId: string;
  toWalletId: string;
  amountCents: number;
};
```

### Output DTO

```ts
type TransferMoneyOutput =
  | { ok: true; transferId: string; fromBalance: number; toBalance: number }
  | { ok: false; code: "INVALID_AMOUNT" | "INSUFFICIENT_FUNDS" | "NOT_FOUND" | "CONFLICT"; message: string };
```

### Flow

1. Check idempotency:

   * nếu key đã có result → return result.
2. Start UoW transaction.
3. Load wallets (repo).
4. Apply domain transfer.
5. Persist wallet balances + create transfer record.
6. Publish event (eventBus) **hoặc** write “outbox” (tối giản: eventBus within UoW).
7. Save idempotency result.
8. Commit.

### Failure model

* Repo returns not found → output `NOT_FOUND`
* UoW commit fail → output `CONFLICT` (simulate)
* Event publish fail → phải quyết định:

  * option A: publish inside transaction → if fail rollback
  * option B: outbox → commit first, publish async

> Bài này chọn **B (outbox-ish)**: application writes `TransferRecorded` to `transferRepo.appendOutbox(event)` trong UoW; infra event dispatcher đọc và publish. (Tối giản vẫn được.)

---

## Starter skeleton (điền TODO)

### `src/domain/errors.ts`

```ts
export type DomainError =
  | { type: "InvalidAmount"; message: string }
  | { type: "InsufficientFunds"; message: string }
  | { type: "SameWalletTransferNotAllowed"; message: string };

export const InvalidAmount = (msg = "Amount must be > 0") => ({ type: "InvalidAmount", message: msg } as const);
export const InsufficientFunds = (msg = "Insufficient funds") => ({ type: "InsufficientFunds", message: msg } as const);
export const SameWalletTransferNotAllowed = (msg = "Cannot transfer to same wallet") =>
  ({ type: "SameWalletTransferNotAllowed", message: msg } as const);
```

### `src/domain/wallet.ts`

```ts
import { DomainError, InsufficientFunds, InvalidAmount, SameWalletTransferNotAllowed } from "./errors";

export type WalletId = string;
export type MoneyCents = number;

export type Wallet = {
  readonly id: WalletId;
  readonly balanceCents: MoneyCents;
};

export function assertValidAmount(amountCents: MoneyCents): DomainError | null {
  if (!Number.isInteger(amountCents) || amountCents <= 0) return InvalidAmount();
  return null;
}

export function applyTransfer(args: {
  from: Wallet;
  to: Wallet;
  amountCents: MoneyCents;
}): { ok: true; fromNext: Wallet; toNext: Wallet } | { ok: false; error: DomainError } {
  const { from, to, amountCents } = args;

  // TODO: optional: block same wallet
  if (from.id === to.id) return { ok: false, error: SameWalletTransferNotAllowed() };

  const invalid = assertValidAmount(amountCents);
  if (invalid) return { ok: false, error: invalid };

  // TODO: insufficient funds
  if (from.balanceCents < amountCents) return { ok: false, error: InsufficientFunds() };

  return {
    ok: true,
    fromNext: { ...from, balanceCents: from.balanceCents - amountCents },
    toNext: { ...to, balanceCents: to.balanceCents + amountCents },
  };
}
```

### `src/application/ports/unitOfWork.ts`

```ts
export interface UnitOfWork {
  within<T>(fn: () => Promise<T>): Promise<T>;
}
```

### `src/application/ports/walletRepo.ts`

```ts
import type { Wallet } from "../../domain/wallet";

export interface WalletRepo {
  getById(id: string): Promise<Wallet | null>;
  save(wallet: Wallet): Promise<void>;
}
```

### `src/application/ports/transferRepo.ts`

```ts
export type TransferRecord = {
  id: string;
  fromWalletId: string;
  toWalletId: string;
  amountCents: number;
  createdAt: string; // ISO
};

export type OutboxEvent = {
  id: string;
  type: "MoneyTransferred";
  payload: {
    transferId: string;
    fromWalletId: string;
    toWalletId: string;
    amountCents: number;
    createdAt: string;
  };
};

export interface TransferRepo {
  create(record: TransferRecord): Promise<void>;
  appendOutbox(event: OutboxEvent): Promise<void>;
}
```

### `src/application/ports/idempotencyStore.ts`

```ts
export interface IdempotencyStore<T> {
  get(key: string): Promise<T | null>;
  set(key: string, value: T): Promise<void>;
}
```

### `src/application/ports/clock.ts`

```ts
export interface Clock {
  nowISO(): string;
}
```

### `src/application/usecases/transferMoney.ts`

```ts
import type { WalletRepo } from "../ports/walletRepo";
import type { TransferRepo } from "../ports/transferRepo";
import type { IdempotencyStore } from "../ports/idempotencyStore";
import type { UnitOfWork } from "../ports/unitOfWork";
import type { Clock } from "../ports/clock";
import { applyTransfer } from "../../domain/wallet";

export type TransferMoneyInput = {
  idempotencyKey: string;
  fromWalletId: string;
  toWalletId: string;
  amountCents: number;
};

export type TransferMoneyOutput =
  | { ok: true; transferId: string; fromBalance: number; toBalance: number }
  | { ok: false; code: "INVALID_AMOUNT" | "INSUFFICIENT_FUNDS" | "NOT_FOUND" | "CONFLICT"; message: string };

export function makeTransferMoney(deps: {
  uow: UnitOfWork;
  walletRepo: WalletRepo;
  transferRepo: TransferRepo;
  idempotency: IdempotencyStore<TransferMoneyOutput>;
  clock: Clock;
  idGen: () => string;
}) {
  const { uow, walletRepo, transferRepo, idempotency, clock, idGen } = deps;

  return async function transferMoney(input: TransferMoneyInput): Promise<TransferMoneyOutput> {
    // 1) idempotency fast-path
    const cached = await idempotency.get(input.idempotencyKey);
    if (cached) return cached;

    try {
      const result = await uow.within(async () => {
        const from = await walletRepo.getById(input.fromWalletId);
        const to = await walletRepo.getById(input.toWalletId);
        if (!from || !to) {
          return { ok: false, code: "NOT_FOUND", message: "Wallet not found" } as const;
        }

        const applied = applyTransfer({ from, to, amountCents: input.amountCents });
        if (!applied.ok) {
          const e = applied.error;
          if (e.type === "InvalidAmount") return { ok: false, code: "INVALID_AMOUNT", message: e.message } as const;
          if (e.type === "InsufficientFunds") return { ok: false, code: "INSUFFICIENT_FUNDS", message: e.message } as const;
          return { ok: false, code: "CONFLICT", message: e.message } as const;
        }

        const transferId = idGen();
        const createdAt = clock.nowISO();

        // persist balances
        await walletRepo.save(applied.fromNext);
        await walletRepo.save(applied.toNext);

        // persist transfer + outbox
        await transferRepo.create({
          id: transferId,
          fromWalletId: from.id,
          toWalletId: to.id,
          amountCents: input.amountCents,
          createdAt,
        });

        await transferRepo.appendOutbox({
          id: idGen(),
          type: "MoneyTransferred",
          payload: { transferId, fromWalletId: from.id, toWalletId: to.id, amountCents: input.amountCents, createdAt },
        });

        return {
          ok: true,
          transferId,
          fromBalance: applied.fromNext.balanceCents,
          toBalance: applied.toNext.balanceCents,
        } as const;
      });

      // 2) store idempotency after successful uow (or even for failures—your choice; here store all results)
      await idempotency.set(input.idempotencyKey, result);
      return result;
    } catch (err) {
      const fail: TransferMoneyOutput = { ok: false, code: "CONFLICT", message: "Transaction failed" };
      // store idempotency for failure? trade-off: usually NO for infra failures
      return fail;
    }
  };
}
```

---

## Infra skeleton (adapters)

### `src/infra/memory/memoryUnitOfWork.ts`

```ts
import type { UnitOfWork } from "../../application/ports/unitOfWork";

export class MemoryUnitOfWork implements UnitOfWork {
  async within<T>(fn: () => Promise<T>): Promise<T> {
    // TODO: simulate transaction by just executing.
    // Stretch: add rollback simulation via snapshot copies.
    return fn();
  }
}
```

### `src/infra/memory/memoryWalletRepo.ts`

```ts
import type { WalletRepo } from "../../application/ports/walletRepo";
import type { Wallet } from "../../domain/wallet";

export class MemoryWalletRepo implements WalletRepo {
  private store = new Map<string, Wallet>();

  seed(wallets: Wallet[]) {
    wallets.forEach(w => this.store.set(w.id, w));
  }

  async getById(id: string): Promise<Wallet | null> {
    return this.store.get(id) ?? null;
  }

  async save(wallet: Wallet): Promise<void> {
    this.store.set(wallet.id, wallet);
  }
}
```

### `src/infra/memory/memoryTransferRepo.ts`

```ts
import type { TransferRepo, TransferRecord, OutboxEvent } from "../../application/ports/transferRepo";

export class MemoryTransferRepo implements TransferRepo {
  public transfers: TransferRecord[] = [];
  public outbox: OutboxEvent[] = [];

  async create(record: TransferRecord): Promise<void> {
    this.transfers.push(record);
  }

  async appendOutbox(event: OutboxEvent): Promise<void> {
    this.outbox.push(event);
  }
}
```

### `src/infra/memory/memoryIdempotencyStore.ts`

```ts
import type { IdempotencyStore } from "../../application/ports/idempotencyStore";

export class MemoryIdempotencyStore<T> implements IdempotencyStore<T> {
  private store = new Map<string, T>();
  async get(key: string): Promise<T | null> { return this.store.get(key) ?? null; }
  async set(key: string, value: T): Promise<void> { this.store.set(key, value); }
}
```

### `src/infra/memory/systemClock.ts`

```ts
import type { Clock } from "../../application/ports/clock";
export class SystemClock implements Clock {
  nowISO(): string { return new Date().toISOString(); }
}
```

### `src/infra/compositionRoot.ts`

```ts
import { makeTransferMoney } from "../application/usecases/transferMoney";
import { MemoryUnitOfWork } from "./memory/memoryUnitOfWork";
import { MemoryWalletRepo } from "./memory/memoryWalletRepo";
import { MemoryTransferRepo } from "./memory/memoryTransferRepo";
import { MemoryIdempotencyStore } from "./memory/memoryIdempotencyStore";
import { SystemClock } from "./memory/systemClock";

export function buildApp() {
  const uow = new MemoryUnitOfWork();
  const walletRepo = new MemoryWalletRepo();
  const transferRepo = new MemoryTransferRepo();
  const idempotency = new MemoryIdempotencyStore<any>();
  const clock = new SystemClock();

  const idGen = () => `id_${Math.random().toString(16).slice(2)}`;

  const transferMoney = makeTransferMoney({ uow, walletRepo, transferRepo, idempotency, clock, idGen });

  return { transferMoney, walletRepo, transferRepo, idempotency };
}
```

---

## Tests (minimum to pass)

### 1) Domain unit: `test/domain.wallet.test.ts`

* valid transfer reduces/increases balances
* insufficient funds → error
* amount <= 0 → error
* same wallet → error (nếu bạn bật rule)

### 2) Application: `test/application.transferMoney.test.ts`

* happy path: persists wallets + transfer + outbox
* idempotency: gọi 2 lần cùng key → **chỉ 1 transfer record**, output y hệt
* not found wallet → NOT_FOUND (không persist gì)
* infra failure simulation (Stretch): UoW throws → CONFLICT

### 3) Infra contract: `test/infra.contract.walletRepo.test.ts`

Định nghĩa “contract” cho `WalletRepo` (chạy được với mọi adapter):

* `save` rồi `getById` trả đúng
* `save` overwrite đúng (last write wins)

> Contract test giúp bạn thay infra (Memory -> Postgres) mà app vẫn bền.

### 4) Dependency rules: `test/dependency-rules.test.ts`

Tối thiểu check:

* Không file nào trong `src/domain/**` import từ `src/infra/**` hoặc `src/application/**`
* Không file nào trong `src/domain/**` import `node:` builtins (fs, path, process)

Bạn có thể viết test “scan text” đơn giản (regex). Senior+ thì nên parse TS AST, nhưng regex đủ cho kata.

---

## Checks (Definition of Done)

* ✅ `domain` compile & test chạy mà **không cần** infra.
* ✅ Use-case chỉ phụ thuộc ports + domain.
* ✅ Infra có thể swap (Memory repo khác) mà không đổi domain/app.
* ✅ Idempotency đảm bảo không tạo transfer trùng.
* ✅ Có dependency-rules test bắt vi phạm boundary.

---

## Stretch (Senior+ → Staff-ish)

1. **Hard mode: rollback thật**
   `MemoryUnitOfWork` snapshot state repos và rollback nếu throw. (Gợi ý: UnitOfWork giữ list “participants” có `snapshot()/restore()`.)
2. **Outbox dispatcher**
   Viết `OutboxDispatcher` (infra) đọc `transferRepo.outbox` và publish qua `EventBus` port. Thêm test “event eventually published”.
3. **Anti-corruption boundary**
   Thêm adapter “FX Rate Service” giả lập (external). Domain không biết tỷ giá; application gọi port lấy rate rồi pass amount đã convert vào domain.
4. **Architecture guardrail tooling**
   Thay regex bằng ESLint import rule hoặc tool kiểu “dependency-cruiser” (nếu bạn thích), nhưng phải có test CI fail khi vi phạm.
5. **Observability ports**
   Thêm `Logger` + `Metrics` ports ở application, infra implement (pino/prom-client). Domain vẫn sạch.

---

## Gợi ý cách làm nhanh (không lạc đường)

1. Viết domain trước (wallet + applyTransfer) + test.
2. Viết ports + use-case (application) + test với memory adapters.
3. Viết composition root + controller (nếu muốn) sau cùng.
4. Viết dependency-rules test cuối để “đóng cổng”.