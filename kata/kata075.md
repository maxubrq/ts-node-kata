# Kata 75 — Monolith Modularization: Module Boundaries & Rules

## Goal

1. Modularize một monolith thành các **modules** có boundary rõ:

   * mỗi module có **public API** (facade)
   * nội bộ module **không bị gọi xuyên tường**
   * modules giao tiếp qua **contracts** (types/ports/events), không import lẫn bậy
2. Enforce rules bằng **architecture tests**:

   * allowed dependencies matrix
   * cấm import “deep path” vào internal
   * cấm circular deps giữa modules
3. Có **integration tests** ở mức module contract (để refactor nội bộ không phá người dùng).

---

## Context (bài toán)

Bạn có monolith kiểu “API server” đang phình ra. Bạn sẽ tạo 4 modules:

1. `identity` (users, auth-ish)
2. `wallet` (balance)
3. `transfer` (chuyển tiền)
4. `notification` (send receipt)

Flow:

* `transfer` cần:

  * đọc user từ `identity`
  * update balance trong `wallet`
  * phát event để `notification` gửi biên nhận
* Rule: `wallet` **không được** biết `identity` và `notification`
* `notification` chỉ nghe event, không gọi trực tiếp `transfer`

---

## Constraints (ép đúng kiểu production)

* ✅ Mỗi module phải có:

  * `index.ts` (Public API duy nhất)
  * `internal/` (không module khác được import)
  * `contracts/` (types/events shared) *hoặc* export types từ `index.ts`
* ✅ Import rules:

  * module A chỉ được import từ module B qua `@modules/B` (alias vào `B/index.ts`)
  * cấm `@modules/B/internal/...`
* ✅ Dependency matrix (bạn phải enforce):

  * `identity` → (none)
  * `wallet` → (none)
  * `transfer` → identity, wallet, contracts/events
  * `notification` → contracts/events
* ✅ Cấm circular dependency giữa modules
* ✅ Có 2 kiểu integration:

  * module-level contract tests
  * end-to-end flow test (transfer triggers notification)

---

## Deliverables (repo shape)

```
kata-75/
  src/
    modules/
      identity/
        index.ts
        internal/
          userRepo.ts
          userService.ts
        contracts.ts
      wallet/
        index.ts
        internal/
          walletRepo.ts
          walletService.ts
        contracts.ts
      transfer/
        index.ts
        internal/
          transferService.ts
          outbox.ts
        contracts.ts
      notification/
        index.ts
        internal/
          notificationHandler.ts
        contracts.ts
    shared/
      eventBus.ts
      di.ts
  test/
    arch.boundaries.test.ts
    arch.dependency-matrix.test.ts
    arch.no-cycles.test.ts
    contracts.wallet.test.ts
    e2e.transfer-notify.test.ts
  tsconfig.json (paths alias)
  README.md
```

Alias gợi ý trong `tsconfig.json`:

* `@modules/*` -> `src/modules/*/index.ts`
* `@shared/*` -> `src/shared/*`

> Mấu chốt: **chỉ alias vào index.ts**, không alias vào internal.

---

## Module contracts (bạn phải đạt)

### `identity` public API

* `getUser(userId): Promise<User | null>`
* types: `User { id, email }`

### `wallet` public API

* `getWallet(walletId)`
* `debit(walletId, amountCents)`
* `credit(walletId, amountCents)`
* domain errors typed

### `transfer` public API

* `transferMoney({ userId, fromWalletId, toWalletId, amountCents, idempotencyKey })`
* emits event: `MoneyTransferred`

### `notification` public API

* `startNotificationConsumer()` subscribe event bus
* handler logs “receipt sent”

---

## Architecture rules (bạn phải implement & test)

1. **No deep imports**

   * Fail nếu có `@modules/wallet/internal/...`
2. **Dependency matrix**

   * transfer được import identity+wallet
   * identity không được import transfer
   * wallet không được import transfer/identity/notification
3. **No cycles**

   * detect circular import graph (module-level)

---

## Starter skeleton (điền TODO)

### `src/shared/eventBus.ts`

```ts
export type DomainEvent =
  | { type: "MoneyTransferred"; payload: { transferId: string; userId: string; amountCents: number } };

type Handler<E extends DomainEvent["type"]> = (e: Extract<DomainEvent, { type: E }>) => Promise<void> | void;

export class EventBus {
  private handlers: { [K in DomainEvent["type"]]?: Handler<K>[] } = {};

  on<E extends DomainEvent["type"]>(type: E, handler: Handler<E>) {
    (this.handlers[type] ??= []).push(handler as any);
  }

  async emit(e: DomainEvent) {
    const hs = this.handlers[e.type] ?? [];
    for (const h of hs) await h(e as any);
  }
}
```

### `src/modules/identity/index.ts`

```ts
export type User = { id: string; email: string };

export interface IdentityAPI {
  getUser(userId: string): Promise<User | null>;
}

// TODO: factory (composition root will inject repo)
export function createIdentityModule(): IdentityAPI {
  throw new Error("TODO");
}
```

### `src/modules/wallet/index.ts`

```ts
export type Wallet = { id: string; balanceCents: number };

export type WalletError =
  | { type: "NotFound" }
  | { type: "InsufficientFunds" };

export interface WalletAPI {
  getWallet(walletId: string): Promise<Wallet | null>;
  debit(walletId: string, amountCents: number): Promise<{ ok: true } | { ok: false; error: WalletError }>;
  credit(walletId: string, amountCents: number): Promise<{ ok: true } | { ok: false; error: WalletError }>;
}

export function createWalletModule(): WalletAPI {
  throw new Error("TODO");
}
```

### `src/modules/transfer/index.ts`

```ts
import type { IdentityAPI } from "@modules/identity";
import type { WalletAPI } from "@modules/wallet";
import type { EventBus } from "@shared/eventBus";

export type TransferInput = {
  userId: string;
  fromWalletId: string;
  toWalletId: string;
  amountCents: number;
  idempotencyKey: string;
};

export type TransferOutput =
  | { ok: true; transferId: string }
  | { ok: false; code: "NOT_FOUND" | "INSUFFICIENT_FUNDS" | "CONFLICT"; message: string };

export interface TransferAPI {
  transferMoney(input: TransferInput): Promise<TransferOutput>;
}

export function createTransferModule(deps: { identity: IdentityAPI; wallet: WalletAPI; bus: EventBus }): TransferAPI {
  // TODO: implement in internal/transferService.ts, exposed via facade
  throw new Error("TODO");
}
```

### `src/modules/notification/index.ts`

```ts
import type { EventBus } from "@shared/eventBus";

export interface NotificationAPI {
  start(): void;
}

export function createNotificationModule(deps: { bus: EventBus }): NotificationAPI {
  // TODO: subscribe to MoneyTransferred only (no imports from transfer!)
  throw new Error("TODO");
}
```

### `src/shared/di.ts` (composition root wiring)

```ts
import { EventBus } from "./eventBus";
import { createIdentityModule } from "@modules/identity";
import { createWalletModule } from "@modules/wallet";
import { createTransferModule } from "@modules/transfer";
import { createNotificationModule } from "@modules/notification";

export function buildMonolith() {
  const bus = new EventBus();

  const identity = createIdentityModule();
  const wallet = createWalletModule();
  const transfer = createTransferModule({ identity, wallet, bus });
  const notification = createNotificationModule({ bus });

  notification.start();

  return { identity, wallet, transfer, notification, bus };
}
```

---

## Architecture tests (đinh của kata)

### 1) No deep imports: `test/arch.boundaries.test.ts`

* scan `src/modules/**.ts`
* fail nếu regex match:

  * `@modules\/[^'"]+\/internal\/`
  * `\/modules\/[^/]+\/internal\/`
* fail nếu import relative “đi xuyên” vào module khác: `../wallet/internal/...`

### 2) Dependency matrix: `test/arch.dependency-matrix.test.ts`

Bạn tạo map allowed:

```ts
const allowed: Record<string, string[]> = {
  identity: [],
  wallet: [],
  transfer: ["identity", "wallet", "shared"],
  notification: ["shared"],
};
```

Scan imports:

* nếu file thuộc module X import từ module Y mà Y không nằm trong allowed[X] → fail.

### 3) No cycles: `test/arch.no-cycles.test.ts`

* build graph module-level:

  * node = module name
  * edge X->Y if any file in X imports Y
* detect cycle bằng DFS (stack)
* fail nếu có cycle.

> Đây là thứ giữ monolith sống thêm 3 năm.

---

## Contract tests (để module tự do refactor)

### `test/contracts.wallet.test.ts`

* `wallet.debit`:

  * insufficient funds returns typed error
  * debit+credit roundtrip

### `test/e2e.transfer-notify.test.ts`

* buildMonolith
* seed user + wallets
* call `transfer.transferMoney`
* assert:

  * balances changed
  * event emitted (you can spy bus.emit or handler called)
  * notification handler executed (set a flag)

---

## Checks (Definition of Done)

* ✅ Modules chỉ expose qua `index.ts`
* ✅ Architecture tests enforce:

  * no deep import
  * dependency matrix
  * no cycles
* ✅ Transfer flow works end-to-end
* ✅ Notification decoupled (event-driven), không import transfer
* ✅ Bạn có thể đổi internal implementation mà không đổi consumer code

---

## Stretch (Senior+ → cực cao)

1. **Public API freezing**

   * enforce that only `index.ts` is exported via TS `exports` field (package.json) hoặc path alias trick
2. **Domain events schema evolution**

   * versioned events: `MoneyTransferred.v1`
3. **Module integration via contracts package**

   * move contracts to `src/contracts/` and ban importing module types directly
4. **Runtime boundary guard**

   * throw if someone calls an internal function (only possible nếu lỡ export), detect via “private symbol” pattern
5. **Refactor to vertical slices**

   * convert one flow (transfer) into vertical slice with its own internal controllers/repo while still respecting global rules.