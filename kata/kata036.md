# Kata 36 — N+1 Query Killer: batch load + DataLoader pattern

## Goal

1. Tạo một **DataLoader** tối giản (không cần lib) cho Node/TS:

* `load(key)` trả Promise
* gom nhiều `load()` trong cùng tick thành **1 batch query**
* cache trong scope request

2. Áp dụng để loại N+1:

* endpoint `GET /v1/orders` trả list order + user info
* baseline naive: 1 query list orders + N query get user
* optimized: 1 query list orders + 1 batch query users (IN)

3. Có tests/bench chứng minh:

* số lần query giảm từ `1 + N` xuống `1 + 1`
* kết quả data không đổi
* cache hoạt động (load cùng key nhiều lần chỉ query 1)

---

## Constraints

* ✅ Batch phải preserve ordering: output array align với keys.
* ✅ Support duplicate keys trong 1 batch (đừng query trùng; nhưng resolve đúng cho từng call).
* ✅ Cache theo request scope (nối Kata 27/30: context).
* ✅ Có instrumentation: đếm query calls.
* ❌ Không được dùng global cache xuyên requests (dễ leak & stale).

---

## Context (scenario demo)

Bạn có:

* `Order { orderId, userId, total }`
* `User { userId, name }`

Repo (fake DB) có:

* `listOrders(): Order[]`
* `getUserById(userId): User | null`
* `getUsersByIds(userIds[]): (User | null)[]`

Endpoint cần trả:

```json
[
  { "orderId": "ord_1", "total": 100, "user": { "userId": "usr_1", "name": "A" } },
  ...
]
```

---

## Deliverables

`kata-36/`

* `src/db.ts` (fake DB + query counter)
* `src/repos.ts` (OrdersRepo, UsersRepo)
* `src/dataloader.ts` (minimal DataLoader)
* `src/service.ts` (naive vs batched)
* `test/kata36.test.ts` (prove query count + correctness)

---

## “Nhìn vào làm được” table

| Phần | Bạn làm gì             | Output                           | Checks            |
| ---- | ---------------------- | -------------------------------- | ----------------- |
| 1    | Fake DB with counters  | counts for list/get/batch        | measurable        |
| 2    | Minimal DataLoader     | `load`, `loadMany`, cache, batch | single batch/tick |
| 3    | Naive implementation   | N+1 appears                      | counts = 1 + N    |
| 4    | Batched implementation | uses loader                      | counts = 1 + 1    |
| 5    | Tests                  | assert counts + same output      | pass              |

---

## Starter Skeleton (điền TODO)

### `src/db.ts`

```ts
export type OrderRow = { orderId: string; userId: string; total: number };
export type UserRow = { userId: string; name: string };

export class FakeDb {
  orders: OrderRow[];
  users: UserRow[];

  // instrumentation
  public q_listOrders = 0;
  public q_getUserById = 0;
  public q_getUsersByIds = 0;

  constructor(seed: { orders: OrderRow[]; users: UserRow[] }) {
    this.orders = seed.orders;
    this.users = seed.users;
  }

  async listOrders(): Promise<OrderRow[]> {
    this.q_listOrders++;
    // simulate IO
    await Promise.resolve();
    return this.orders.map((o) => ({ ...o }));
  }

  async getUserById(userId: string): Promise<UserRow | null> {
    this.q_getUserById++;
    await Promise.resolve();
    const u = this.users.find((x) => x.userId === userId);
    return u ? { ...u } : null;
  }

  async getUsersByIds(userIds: string[]): Promise<Array<UserRow | null>> {
    this.q_getUsersByIds++;
    await Promise.resolve();
    // Must return aligned with input order
    return userIds.map((id) => {
      const u = this.users.find((x) => x.userId === id);
      return u ? { ...u } : null;
    });
  }
}
```

### `src/repos.ts`

```ts
import type { FakeDb, OrderRow, UserRow } from "./db";

export class OrdersRepo {
  constructor(private readonly db: FakeDb) {}
  list() { return this.db.listOrders(); }
}

export class UsersRepo {
  constructor(private readonly db: FakeDb) {}
  getById(userId: string) { return this.db.getUserById(userId); }
  getByIds(userIds: string[]) { return this.db.getUsersByIds(userIds); }
}

export type { OrderRow, UserRow };
```

### `src/dataloader.ts` (minimal DataLoader)

```ts
export type BatchFn<K, V> = (keys: K[]) => Promise<V[]>;

export class DataLoader<K, V> {
  private cache = new Map<K, Promise<V>>();
  private queue: K[] = [];
  private resolvers = new Map<K, Array<(v: V) => void>>();
  private rejecters = new Map<K, Array<(e: unknown) => void>>();
  private scheduled = false;

  constructor(private readonly batchFn: BatchFn<K, V>) {}

  load(key: K): Promise<V> {
    // cache per request scope
    const cached = this.cache.get(key);
    if (cached) return cached;

    const p = new Promise<V>((resolve, reject) => {
      // collect resolvers for duplicate keys
      const rs = this.resolvers.get(key) ?? [];
      rs.push(resolve);
      this.resolvers.set(key, rs);

      const rj = this.rejecters.get(key) ?? [];
      rj.push(reject);
      this.rejecters.set(key, rj);
    });

    this.cache.set(key, p);
    this.queue.push(key);

    if (!this.scheduled) {
      this.scheduled = true;
      queueMicrotask(() => this.flush());
    }

    return p;
  }

  async loadMany(keys: K[]): Promise<V[]> {
    return Promise.all(keys.map((k) => this.load(k)));
  }

  private async flush() {
    this.scheduled = false;

    // TODO:
    // - take current queue snapshot and clear it
    // - dedupe keys for batch query (but resolve all duplicates)
    // - call batchFn with deduped keys
    // - map results back to each key
    // - resolve/reject for each key
    throw new Error("TODO");
  }
}
```

### `src/service.ts`

```ts
import type { OrderRow, UserRow } from "./repos";
import { OrdersRepo, UsersRepo } from "./repos";
import { DataLoader } from "./dataloader";

export type OrderDto = {
  orderId: string;
  total: number;
  user: { userId: string; name: string } | null;
};

export async function listOrdersNaive(orders: OrdersRepo, users: UsersRepo): Promise<OrderDto[]> {
  const os = await orders.list();
  const out: OrderDto[] = [];

  for (const o of os) {
    const u = await users.getById(o.userId); // N queries
    out.push({ orderId: o.orderId, total: o.total, user: u ? { userId: u.userId, name: u.name } : null });
  }

  return out;
}

export async function listOrdersBatched(orders: OrdersRepo, users: UsersRepo): Promise<OrderDto[]> {
  const os = await orders.list();

  const loader = new DataLoader<string, UserRow | null>(async (keys) => {
    // 1 batch query
    return users.getByIds(keys);
  });

  const out: OrderDto[] = [];
  for (const o of os) {
    const u = await loader.load(o.userId); // gets batched
    out.push({ orderId: o.orderId, total: o.total, user: u ? { userId: u.userId, name: u.name } : null });
  }
  return out;
}
```

---

## TODO bắt buộc trong `flush()`

Các bước đúng:

1. Snapshot queue + clear:

* `const keys = this.queue; this.queue = [];`

2. Dedupe keys (preserve first-seen order):

* `const uniq: K[] = []` + `Set`

3. Call `batchFn(uniq)`; ensure length match:

* nếu mismatch: reject all keys

4. Resolve per key:

* với mỗi `uniq[i]` → `result[i]`
* gọi toàn bộ resolvers của key, clear maps

5. Nếu batchFn throw → reject tất cả keys trong batch

---

## Tests (vitest)

### `test/kata36.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { FakeDb } from "../src/db";
import { OrdersRepo, UsersRepo } from "../src/repos";
import { listOrdersNaive, listOrdersBatched } from "../src/service";

function seedDb() {
  return new FakeDb({
    users: [
      { userId: "usr_1", name: "A" },
      { userId: "usr_2", name: "B" },
      { userId: "usr_3", name: "C" },
    ],
    orders: [
      { orderId: "ord_1", userId: "usr_1", total: 100 },
      { orderId: "ord_2", userId: "usr_2", total: 200 },
      { orderId: "ord_3", userId: "usr_1", total: 300 }, // duplicate userId
      { orderId: "ord_4", userId: "usr_3", total: 400 },
    ],
  });
}

describe("kata36 - n+1 killer", () => {
  it("naive causes N+1 queries", async () => {
    const db = seedDb();
    const orders = new OrdersRepo(db);
    const users = new UsersRepo(db);

    const out = await listOrdersNaive(orders, users);

    expect(out.length).toBe(4);
    expect(db.q_listOrders).toBe(1);
    expect(db.q_getUserById).toBe(4);      // N
    expect(db.q_getUsersByIds).toBe(0);
  });

  it("batched uses 1 batch query and caches duplicates", async () => {
    const db = seedDb();
    const orders = new OrdersRepo(db);
    const users = new UsersRepo(db);

    const out = await listOrdersBatched(orders, users);

    expect(out.length).toBe(4);
    expect(db.q_listOrders).toBe(1);
    expect(db.q_getUserById).toBe(0);
    expect(db.q_getUsersByIds).toBe(1);   // ✅ 1 batch

    // correctness
    expect(out[0].user?.name).toBe("A");
    expect(out[2].user?.name).toBe("A");
  });
});
```

---

## Checks (Definition of Done)

Bạn pass kata này khi:

1. Naive path: query count đúng `listOrders=1`, `getUserById=N`.
2. Batched path: `listOrders=1`, `getUsersByIds=1` (và `getUserById=0`).
3. Duplicate key (`usr_1`) không làm batch query trùng (vẫn 1 batch) và resolve đúng.
4. Data output giữa naive và batched **giống nhau**.

---

## Stretch (Senior+)

1. **Per-request cache invalidation**

* loader có `clear(key)` / `prime(key, value)`.

2. **Batch window**

* gom theo tick là cơ bản; thêm `maxBatchSize` để split.

3. **Error handling**

* batchFn trả lỗi cho 1 key: kiểu `Result` per key (không fail whole batch).

4. **Integration với AsyncLocalStorage**

* tạo loader từ context để mọi service layer dùng chung loader per request (nối Kata 27/30).