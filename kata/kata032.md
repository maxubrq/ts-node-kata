# Kata 32 — Optimistic Locking: version field + retry policy

## Goal

1. Implement optimistic locking bằng `version`:

* mỗi record có `version: number`
* update phải kèm `expectedVersion`
* nếu mismatch ⇒ throw `CONFLICT_VERSION` (aka optimistic lock conflict)

2. Implement retry policy:

* retry khi conflict, có giới hạn attempts
* có backoff/jitter (tối thiểu linear)
* phân biệt retryable vs non-retryable errors

3. Có tests chứng minh:

* không có optimistic lock ⇒ lost update xảy ra (baseline test)
* có optimistic lock ⇒ lost update bị chặn
* retry policy giúp request “thắng” sau vài lần khi có contention
* retry dừng đúng lúc, không loop vô hạn

---

## Constraints

* ✅ Update API phải theo pattern: **read → compute → write(expectedVersion)**.
* ✅ Repo phải có method `updateWithVersion(...)`.
* ✅ Retry chỉ cho `CONFLICT_VERSION`, không retry cho lỗi domain khác.
* ✅ Có test mô phỏng 2 “concurrent” updates (không cần thread thật, chỉ cần interleaving).
* ❌ Không “lock” bằng mutex (kata này tập trung optimistic, không pessimistic).

---

## Context (scenario demo)

Bạn có `Wallet` (hoặc `Inventory`) và use-case `addFunds(walletId, amount)`.

Nếu 2 request cùng đọc balance=100 và cùng ghi:

* req1 +10 → 110
* req2 +20 → 120
  Kết quả đúng phải là **130**, nhưng lost update cho ra **110 hoặc 120**.

Optimistic locking ngăn chuyện đó.

---

## Deliverables

`kata-32/`

* `src/db.ts` (reuse snapshot DB từ kata31, mở rộng table)
* `src/repos.ts` (WalletRepo + optimistic update)
* `src/service.ts` (addFunds + retry wrapper)
* `test/kata32.test.ts` (baseline lost update + optimistic lock + retry)

---

## Bảng “nhìn vào làm được”

| Phần       | Bạn làm gì                       | Output                           | Checks                |
| ---------- | -------------------------------- | -------------------------------- | --------------------- |
| 1. Model   | thêm `version` vào row           | `WalletRow { balance, version }` | version init = 0      |
| 2. Repo    | `updateBalance(expectedVersion)` | throw conflict nếu mismatch      | conflict reproducible |
| 3. Service | `addFunds` theo read→write       | atomic per tx                    | no lost update        |
| 4. Retry   | `retryOnConflict(fn)`            | attempts/backoff                 | stops correctly       |
| 5. Tests   | simulate interleaving            | 4 tests                          | pass                  |

---

## Starter Skeleton (điền TODO)

### `src/db.ts` (mini DB cho kata này)

```ts
export type WalletRow = { walletId: string; balance: number; version: number };

export type DbState = {
  wallets: Map<string, WalletRow>;
};

function cloneState(s: DbState): DbState {
  return {
    wallets: new Map([...s.wallets.entries()].map(([k, v]) => [k, { ...v }])),
  };
}

export class Database {
  private state: DbState;

  constructor(seed?: Partial<DbState>) {
    this.state = { wallets: seed?.wallets ?? new Map() };
  }

  snapshot(): DbState {
    return cloneState(this.state);
  }

  beginTransaction(): Transaction {
    return new Transaction(this);
  }

  _getStateRef(): DbState {
    return this.state;
  }
  _setStateRef(next: DbState) {
    this.state = next;
  }
}

export class Transaction {
  private active = true;
  private readonly before: DbState;
  private working: DbState;

  constructor(private readonly db: Database) {
    const current = db._getStateRef();
    this.before = cloneState(current);
    this.working = cloneState(current);
  }

  state(): DbState {
    if (!this.active) throw new Error("TX_NOT_ACTIVE");
    return this.working;
  }

  commit() {
    if (!this.active) throw new Error("TX_NOT_ACTIVE");
    this.db._setStateRef(this.working);
    this.active = false;
  }

  rollback() {
    if (!this.active) throw new Error("TX_NOT_ACTIVE");
    this.db._setStateRef(this.before);
    this.active = false;
  }
}
```

### `src/tx.ts`

```ts
import type { Database, Transaction } from "./db";
import { makeWalletRepo, type UnitOfWork } from "./repos";

export async function withTransaction<T>(
  db: Database,
  fn: (tx: Transaction, uow: UnitOfWork) => Promise<T> | T
): Promise<T> {
  const tx = db.beginTransaction();
  const uow: UnitOfWork = { tx, wallets: makeWalletRepo(tx) };

  try {
    const out = await fn(tx, uow);
    tx.commit();
    return out;
  } catch (e) {
    try { tx.rollback(); } catch {}
    throw e;
  }
}
```

### `src/repos.ts`

```ts
import type { Transaction, WalletRow } from "./db";

export type UnitOfWork = {
  tx: Transaction;
  wallets: WalletRepo;
};

export interface WalletRepo {
  get(walletId: string): Promise<WalletRow | null>;

  // baseline (unsafe) update to demonstrate lost update
  setBalanceUnsafe(walletId: string, newBalance: number): Promise<void>;

  // optimistic locking update
  setBalanceWithVersion(walletId: string, input: {
    newBalance: number;
    expectedVersion: number;
  }): Promise<void>;
}

export function makeWalletRepo(tx: Transaction): WalletRepo {
  return {
    async get(walletId) {
      const row = tx.state().wallets.get(walletId);
      return row ? { ...row } : null;
    },

    async setBalanceUnsafe(walletId, newBalance) {
      const s = tx.state();
      const row = s.wallets.get(walletId);
      if (!row) throw new Error("WALLET_NOT_FOUND");
      row.balance = newBalance;
      // ❌ no version check, no increment (unsafe)
      s.wallets.set(walletId, row);
    },

    async setBalanceWithVersion(walletId, { newBalance, expectedVersion }) {
      const s = tx.state();
      const row = s.wallets.get(walletId);
      if (!row) throw new Error("WALLET_NOT_FOUND");

      // TODO: if row.version !== expectedVersion => throw conflict
      // TODO: else set balance and increment version
      throw new Error("TODO");
    },
  };
}
```

### `src/retry.ts`

```ts
function sleep(ms: number) {
  return new Promise((r) => setTimeout(r, ms));
}

export type RetryOptions = {
  maxAttempts: number;     // e.g. 5
  baseDelayMs: number;     // e.g. 5
  maxDelayMs?: number;     // e.g. 50
};

export async function retryOnOptimisticConflict<T>(
  fn: (attempt: number) => Promise<T>,
  opts: RetryOptions
): Promise<T> {
  const maxDelay = opts.maxDelayMs ?? 50;

  for (let attempt = 1; attempt <= opts.maxAttempts; attempt++) {
    try {
      return await fn(attempt);
    } catch (e: any) {
      const msg = e instanceof Error ? e.message : String(e);
      const isConflict = msg === "CONFLICT_VERSION";

      // TODO: only retry conflict; otherwise throw
      // TODO: if last attempt, throw
      // TODO: delay with simple backoff + jitter
      throw e;
    }
  }

  // unreachable
  throw new Error("RETRY_EXHAUSTED");
}
```

### `src/service.ts`

```ts
import type { Database } from "./db";
import { withTransaction } from "./tx";
import { retryOnOptimisticConflict } from "./retry";

export async function addFundsUnsafe(db: Database, walletId: string, amount: number) {
  return withTransaction(db, async (_tx, uow) => {
    const w = await uow.wallets.get(walletId);
    if (!w) throw new Error("WALLET_NOT_FOUND");
    const next = w.balance + amount;
    await uow.wallets.setBalanceUnsafe(walletId, next);
    return next;
  });
}

export async function addFundsOptimisticOnce(db: Database, walletId: string, amount: number) {
  return withTransaction(db, async (_tx, uow) => {
    const w = await uow.wallets.get(walletId);
    if (!w) throw new Error("WALLET_NOT_FOUND");

    const next = w.balance + amount;
    await uow.wallets.setBalanceWithVersion(walletId, {
      newBalance: next,
      expectedVersion: w.version,
    });

    return next;
  });
}

export async function addFundsWithRetry(db: Database, walletId: string, amount: number) {
  return retryOnOptimisticConflict(
    async () => addFundsOptimisticOnce(db, walletId, amount),
    { maxAttempts: 5, baseDelayMs: 5, maxDelayMs: 50 }
  );
}
```

---

## Tests (vitest) — mô phỏng concurrency bằng interleaving

### `test/kata32.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { Database } from "../src/db";
import { addFundsUnsafe, addFundsOptimisticOnce, addFundsWithRetry } from "../src/service";
import { withTransaction } from "../src/tx";

describe("kata32 - optimistic locking", () => {
  it("baseline: lost update can happen without optimistic lock", async () => {
    const db = new Database({
      wallets: new Map([["w1", { walletId: "w1", balance: 100, version: 0 }]]),
    });

    // simulate two requests interleaving:
    // T1 read
    let t1Read: any;
    await withTransaction(db, async (_tx, uow) => {
      t1Read = await uow.wallets.get("w1"); // 100
      // do not write yet
      return null;
    });

    // T2 read+write +20
    await addFundsUnsafe(db, "w1", 20); // writes 120

    // T1 write +10 based on stale read(100) -> overwrites to 110
    await withTransaction(db, async (_tx, uow) => {
      const next = t1Read.balance + 10; // 110
      await uow.wallets.setBalanceUnsafe("w1", next);
      return null;
    });

    expect(db.snapshot().wallets.get("w1")?.balance).toBe(110); // lost update
  });

  it("optimistic lock prevents stale write (conflict)", async () => {
    const db = new Database({
      wallets: new Map([["w1", { walletId: "w1", balance: 100, version: 0 }]]),
    });

    // T1 reads version 0
    const t1 = await withTransaction(db, async (_tx, uow) => uow.wallets.get("w1"));

    // T2 updates -> version becomes 1
    await addFundsOptimisticOnce(db, "w1", 20);

    // T1 attempts update with expectedVersion 0 => conflict
    await expect(
      withTransaction(db, async (_tx, uow) => {
        if (!t1) throw new Error("no");
        await uow.wallets.setBalanceWithVersion("w1", {
          newBalance: t1.balance + 10,
          expectedVersion: t1.version, // 0
        });
      })
    ).rejects.toThrow("CONFLICT_VERSION");
  });

  it("retry policy resolves contention eventually", async () => {
    const db = new Database({
      wallets: new Map([["w1", { walletId: "w1", balance: 100, version: 0 }]]),
    });

    // Create contention by updating wallet between attempts:
    // We'll do one update first, then run addFundsWithRetry which should read latest and succeed.
    await addFundsOptimisticOnce(db, "w1", 20); // 120, v1

    const out = await addFundsWithRetry(db, "w1", 10);
    expect(out).toBe(130);
    expect(db.snapshot().wallets.get("w1")?.balance).toBe(130);
  });

  it("retry does not retry non-conflict errors", async () => {
    const db = new Database({ wallets: new Map() });
    await expect(addFundsWithRetry(db, "missing", 10)).rejects.toThrow("WALLET_NOT_FOUND");
  });
});
```

---

## TODOs bạn phải hoàn thành (rõ ràng)

### 1) Implement optimistic update (repo)

Trong `setBalanceWithVersion`:

* nếu `row.version !== expectedVersion` → `throw new Error("CONFLICT_VERSION")`
* else:

  * `row.balance = newBalance`
  * `row.version = row.version + 1`

### 2) Implement retry policy (retry.ts)

* chỉ retry khi `CONFLICT_VERSION`
* nếu attempt == maxAttempts → throw conflict
* delay: `delay = min(maxDelay, baseDelayMs * attempt)` + jitter nhỏ (0..baseDelayMs)

  * jitter có thể dùng `Math.random()`

---

## Checks (Definition of Done)

Bạn pass kata này khi:

1. Baseline test chứng minh lost update **có thể** xảy ra khi không lock.
2. Optimistic lock làm stale write fail bằng `CONFLICT_VERSION`.
3. Retry:

* retry đúng loại lỗi
* dừng đúng số lần
* cuối cùng balance đúng (130).

4. Không có infinite loop.

---

## Stretch (Senior+)

1. **Retry budget + observability**:

* log mỗi conflict attempt (event: `db.optimistic_conflict`)
* metric `optimistic_conflicts_total`

2. **Idempotency**:

* kết hợp idempotency key để retry safe ở use-case create/update.

3. **Generalized helper**:

* `updateWithOptimisticLock(get, setWithVersion, mutateFn)` pattern tái sử dụng.

4. **Write skew / invariants**:

* thêm invariant: balance không vượt quá limit; test concurrent updates fail correctly.