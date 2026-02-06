# Kata 72 — Dependency Inversion in TS: Composition Root

## Goal

1. Thiết kế **composition root** (entrypoint) để:

   * Build object graph (wiring deps)
   * Control **lifetime** (singleton/per-request)
   * Apply **cross-cutting concerns** (logging, metrics, tracing, timeouts)
2. Enforce **Dependency Inversion**:

   * application phụ thuộc **ports**
   * infra implements ports
   * only composition root imports infra
3. Có **test seam**: swap adapters dễ dàng trong test (fakes/in-memory).

---

## Context

Bạn có use-case `TransferMoney` (từ Kata 71) hoặc 1 use-case tương tự. Hiện dev hay làm bậy:

```ts
// ❌ anti-pattern: new infra inside app
export async function transferMoney(...) {
  const repo = new PostgresWalletRepo(...)
  ...
}
```

Bạn cần cấu trúc để chuyện này **không thể xảy ra**.

---

## Constraints

* ✅ `src/application/**` **không được import** `src/infra/**`
* ✅ Chỉ **1 nơi** được `new` infra adapter: `src/main/compositionRoot.ts` (hoặc `src/infra/compositionRoot.ts`, nhưng là entrypoint)
* ✅ Use-case factory style: `makeXxx(deps)` trả function/class đã được inject
* ✅ Có “lifetime policy” rõ:

  * DB pool: singleton
  * request-scoped: requestId, logger child, deadline
* ✅ Có **guardrail test**: fail nếu file app dùng `new SomeRepo` hoặc import infra
* ✅ Có **2 wiring profile**:

  * `buildProdApp()`
  * `buildTestApp()` (in-memory / fakes)
* ✅ Có **integration test** chứng minh swap deps không đổi app code.

---

## Deliverables (repo shape)

Tạo `kata-72/`:

```
kata-72/
  src/
    domain/...
    application/
      ports/...
      usecases/transferMoney.ts
      app.ts                // app "kernel" type
    infra/
      db/...
      http/...
      logging/...
    main/
      compositionRoot.ts    // ONLY wiring
      server.ts             // uses composition root
      requestContext.ts     // request-scoped deps
  test/
    wiring.prod.test.ts
    wiring.testprofile.test.ts
    guardrails.test.ts
  README.md
```

---

## Design spec (bạn phải đạt)

### 1) “App Kernel” interface

Application expose **một object** chứa các entrypoints (use-cases):

```ts
export type App = {
  transferMoney: (input: TransferMoneyInput, ctx: RequestContext) => Promise<TransferMoneyOutput>;
  // sau này thêm usecase khác không đổi wiring style
};
```

### 2) RequestContext (request-scoped deps)

RequestContext chứa:

* `requestId: string`
* `deadlineMs: number`
* `logger: Logger` (port)
* `traceId?: string` (optional)

> App functions nhận `ctx` để không dùng AsyncLocalStorage trong kata này (Stretch mới dùng ALS).

### 3) Composition root làm 3 việc

* Build singletons: dbPool, metrics registry, baseLogger, config
* Provide request factory: create RequestContext
* Wire ports → usecases → App

---

## Starter skeleton (điền TODO)

### `src/application/app.ts`

```ts
import type { TransferMoneyInput, TransferMoneyOutput } from "./usecases/transferMoney";

export type RequestContext = {
  requestId: string;
  deadlineMs: number;
  traceId?: string;
  logger: Logger;
};

export interface Logger {
  info(obj: Record<string, unknown>, msg?: string): void;
  error(obj: Record<string, unknown>, msg?: string): void;
  child(bindings: Record<string, unknown>): Logger;
}

export type App = {
  transferMoney(input: TransferMoneyInput, ctx: RequestContext): Promise<TransferMoneyOutput>;
};
```

### `src/application/usecases/transferMoney.ts`

(giữ kiểu factory injection)

```ts
import type { WalletRepo } from "../ports/walletRepo";
import type { TransferRepo } from "../ports/transferRepo";
import type { IdempotencyStore } from "../ports/idempotencyStore";
import type { UnitOfWork } from "../ports/unitOfWork";
import type { Clock } from "../ports/clock";
import type { RequestContext } from "../app";
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
  return async function transferMoney(input: TransferMoneyInput, ctx: RequestContext): Promise<TransferMoneyOutput> {
    ctx.logger.info({ input: { ...input, amountCents: input.amountCents } }, "transferMoney called");

    // TODO: same as kata-71, but:
    // - respect ctx.deadlineMs (timeout budget) (stretch: implement)
    // - log outcomes
    // - do NOT import infra anywhere

    const cached = await deps.idempotency.get(input.idempotencyKey);
    if (cached) return cached;

    try {
      const result = await deps.uow.within(async () => {
        const from = await deps.walletRepo.getById(input.fromWalletId);
        const to = await deps.walletRepo.getById(input.toWalletId);
        if (!from || !to) return { ok: false, code: "NOT_FOUND", message: "Wallet not found" } as const;

        const applied = applyTransfer({ from, to, amountCents: input.amountCents });
        if (!applied.ok) {
          const t = applied.error.type;
          if (t === "InvalidAmount") return { ok: false, code: "INVALID_AMOUNT", message: applied.error.message } as const;
          if (t === "InsufficientFunds") return { ok: false, code: "INSUFFICIENT_FUNDS", message: applied.error.message } as const;
          return { ok: false, code: "CONFLICT", message: applied.error.message } as const;
        }

        const transferId = deps.idGen();
        const createdAt = deps.clock.nowISO();

        await deps.walletRepo.save(applied.fromNext);
        await deps.walletRepo.save(applied.toNext);

        await deps.transferRepo.create({
          id: transferId,
          fromWalletId: from.id,
          toWalletId: to.id,
          amountCents: input.amountCents,
          createdAt,
        });

        await deps.transferRepo.appendOutbox({
          id: deps.idGen(),
          type: "MoneyTransferred",
          payload: { transferId, fromWalletId: from.id, toWalletId: to.id, amountCents: input.amountCents, createdAt },
        });

        return { ok: true, transferId, fromBalance: applied.fromNext.balanceCents, toBalance: applied.toNext.balanceCents } as const;
      });

      await deps.idempotency.set(input.idempotencyKey, result);
      ctx.logger.info({ result }, "transferMoney done");
      return result;
    } catch (e) {
      ctx.logger.error({ err: String(e) }, "transferMoney failed");
      return { ok: false, code: "CONFLICT", message: "Transaction failed" };
    }
  };
}
```

### `src/main/requestContext.ts`

```ts
import type { RequestContext, Logger } from "../application/app";

export type RequestContextFactory = (args: { requestId: string; traceId?: string; deadlineMs: number }) => RequestContext;

export function makeRequestContextFactory(baseLogger: Logger): RequestContextFactory {
  return ({ requestId, traceId, deadlineMs }) => {
    const logger = baseLogger.child({ requestId, traceId });
    return { requestId, traceId, deadlineMs, logger };
  };
}
```

### `src/main/compositionRoot.ts` (trọng tâm kata)

```ts
import type { App } from "../application/app";
import { makeTransferMoney } from "../application/usecases/transferMoney";
import { makeRequestContextFactory } from "./requestContext";

// infra (ONLY HERE)
import { MemoryUnitOfWork } from "../infra/memory/memoryUnitOfWork";
import { MemoryWalletRepo } from "../infra/memory/memoryWalletRepo";
import { MemoryTransferRepo } from "../infra/memory/memoryTransferRepo";
import { MemoryIdempotencyStore } from "../infra/memory/memoryIdempotencyStore";
import { SystemClock } from "../infra/memory/systemClock";
import { ConsoleLogger } from "../infra/logging/consoleLogger";

export type BuiltApp = {
  app: App;
  createCtx: ReturnType<typeof makeRequestContextFactory>;
  // expose for tests (optional)
  __infra: {
    walletRepo: MemoryWalletRepo;
    transferRepo: MemoryTransferRepo;
  };
};

export function buildTestApp(): BuiltApp {
  // SINGLETONS
  const baseLogger = new ConsoleLogger({ env: "test" });
  const createCtx = makeRequestContextFactory(baseLogger);

  const uow = new MemoryUnitOfWork();
  const walletRepo = new MemoryWalletRepo();
  const transferRepo = new MemoryTransferRepo();
  const idempotency = new MemoryIdempotencyStore<any>();
  const clock = new SystemClock();
  const idGen = () => `id_${Math.random().toString(16).slice(2)}`;

  // USECASES
  const transferMoney = makeTransferMoney({ uow, walletRepo, transferRepo, idempotency, clock, idGen });

  // APP KERNEL
  const app: App = {
    transferMoney,
  };

  return { app, createCtx, __infra: { walletRepo, transferRepo } };
}

/**
 * TODO: buildProdApp()
 * - change infra to "db pool" + "pino logger" style adapters
 * - still ONLY here imports infra
 */
export function buildProdApp(): BuiltApp {
  return buildTestApp(); // TODO replace later
}
```

### `src/infra/logging/consoleLogger.ts`

```ts
import type { Logger } from "../../application/app";

export class ConsoleLogger implements Logger {
  constructor(private base: Record<string, unknown> = {}) {}

  child(bindings: Record<string, unknown>): Logger {
    return new ConsoleLogger({ ...this.base, ...bindings });
  }

  info(obj: Record<string, unknown>, msg?: string): void {
    console.log(JSON.stringify({ level: "info", ...this.base, ...obj, msg }));
  }

  error(obj: Record<string, unknown>, msg?: string): void {
    console.error(JSON.stringify({ level: "error", ...this.base, ...obj, msg }));
  }
}
```

---

## Tests (bắt buộc)

### 1) Wiring works (test profile): `test/wiring.testprofile.test.ts`

* buildTestApp
* seed wallets qua `__infra.walletRepo.seed(...)`
* call `app.transferMoney(input, ctx)`
* assert:

  * balances updated
  * outbox event exists
  * calling again same idempotencyKey returns same transferId và không tăng số transfer

### 2) Prod wiring smoke: `test/wiring.prod.test.ts`

* `buildProdApp()` phải build được và pass 1 happy path (tạm thời có thể reuse memory, nhưng bạn phải để TODO rõ).

### 3) Guardrails: `test/guardrails.test.ts`

Fail nếu:

* bất kỳ file trong `src/application/**` chứa chuỗi `new Memory` / `new Postgres` / `from "../infra"` (regex scan).
* bất kỳ import path trong domain/app trỏ tới infra.

> Guardrail test là “khóa luật” — đây mới là Senior+.

---

## Checks (Definition of Done)

* ✅ Chỉ `compositionRoot.ts` import infra.
* ✅ Use-case nhận deps qua factory injection, không `new` infra.
* ✅ Có request-scoped context (logger child + deadline).
* ✅ Có 2 wiring profiles (prod/test).
* ✅ Guardrail test fail đúng khi vi phạm.

---

## Stretch (Senior+ → rất cao)

1. **Lifetime container mini (không dùng library)**
   Viết `createContainer()` hỗ trợ:

   * `singleton(token, factory)`
   * `scoped(token, factory)`
   * `resolve(token, scope)`
   * chống circular deps (detect stack)
2. **Deadline propagation thật**
   Tạo port `Deadline` hoặc helper `withDeadline(promise, ctx.deadlineMs)` và áp vào repo calls.
3. **Decorator ports**
   Implement wrapper:

   * `walletRepo = withMetrics(withLogging(walletRepo))`
   * `uow = withTracing(uow)`
     Tất cả wiring chỉ ở composition root.
4. **Static typing của deps graph**
   Dùng type-level trick để composition root fail compile nếu thiếu dep (no runtime surprise).
5. **No service locator**
   Cấm `container.get()` trong application; container chỉ dùng ở composition root.