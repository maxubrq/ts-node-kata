# Kata 22 — Pagination Correctness

**Cursor-based pagination (không offset)**: stable, không skip/duplicate khi có insert/delete concurrent.

---

## Goal

Bạn build API:

`GET /orders?limit=50&cursor=...`

Yêu cầu:

1. Cursor-based, **deterministic ordering**
2. **Không duplicate / không skip** khi:

* có order mới được insert trong lúc client đang paginate
* có delete/updates (status)

3. Cursor encode/decode an toàn (opaque, signed hoặc at least base64 JSON)
4. Có test cases mô phỏng concurrent inserts và chứng minh correctness

---

## Constraints

* ❌ Không offset pagination
* ✅ Sort key phải **total order**: `(createdAt, id)` (id tie-breaker)
* ✅ Cursor phải chứa last seen sort key
* ✅ Limit enforcement (min/max)
* ✅ Không `any`
* ✅ Parse cursor từ `unknown`/string an toàn

---

## Spec

### Domain

```ts
type Order = {
  id: string;          // unique, monotonic-ish not guaranteed
  createdAt: number;   // epoch ms
  userId: string;
  amount: number;
  status: "created" | "paid" | "cancelled";
};
```

### Query

* filter: `userId` (required)
* orderBy: newest-first hoặc oldest-first (choose one; kata yêu cầu implement **oldest-first** để dễ reasoning)
* limit default 20, max 100
* cursor: optional

### Response

```ts
type Page<T> = {
  items: T[];
  nextCursor?: string;
};
```

---

## Core correctness rules

### Sorting

Oldest-first means:

* primary: `createdAt ASC`
* tie-breaker: `id ASC`

### Cursor semantics

Cursor represents **exclusive start**:

* “give me items strictly after (createdAt, id)”

So query condition:

* `(createdAt > lastCreatedAt) OR (createdAt == lastCreatedAt AND id > lastId)`

---

## Deliverables

File `kata22.ts` gồm:

1. In-memory “DB” array + helper `insertOrder`
2. `encodeCursor` / `decodeCursor`
3. `listOrders(args)` returns `Page<Order>`
4. Tests (simple runner):

* basic pagination
* tie-break correctness
* concurrent insert between pages (no duplicate/skip)
* invalid cursor handling

---

## Starter skeleton (điền TODO)

```ts
/* kata22.ts */

type OrderStatus = "created" | "paid" | "cancelled";

type Order = {
  id: string;
  createdAt: number;
  userId: string;
  amount: number;
  status: OrderStatus;
};

type Page<T> = { items: T[]; nextCursor?: string };

type ListArgs = {
  userId: string;
  limit?: number;
  cursor?: string;
};

type Cursor = { createdAt: number; id: string };

// ---------- Cursor codec (opaque) ----------
function encodeCursor(c: Cursor): string {
  // TODO: base64url encode JSON
  // NOTE: do NOT use Buffer.from(...).toString("base64") directly without url-safe tweak
  return "TODO";
}

function decodeCursor(s: string): Cursor | undefined {
  // TODO: return undefined if invalid
  return undefined;
}

function clampLimit(limit: number | undefined): number {
  const x = typeof limit === "number" && Number.isFinite(limit) ? Math.trunc(limit) : 20;
  return Math.max(1, Math.min(100, x));
}

// ---------- In-memory DB (sorted insert not required; we will sort on query) ----------
const db: Order[] = [];

function insertOrder(o: Order) {
  db.push(o);
}

function compareOrder(a: Order, b: Order): number {
  if (a.createdAt !== b.createdAt) return a.createdAt - b.createdAt;
  return a.id < b.id ? -1 : a.id > b.id ? 1 : 0;
}

function isAfter(o: Order, c: Cursor): boolean {
  // exclusive start
  return o.createdAt > c.createdAt || (o.createdAt === c.createdAt && o.id > c.id);
}

// ---------- TODO: listOrders ----------
function listOrders(args: ListArgs): Page<Order> {
  const limit = clampLimit(args.limit);
  const cursor = args.cursor ? decodeCursor(args.cursor) : undefined;

  // TODO:
  // 1) filter by userId
  // 2) sort by (createdAt asc, id asc)
  // 3) if cursor present: take only items AFTER cursor (exclusive)
  // 4) take first `limit`
  // 5) nextCursor = encodeCursor(lastItemSortKey) if more items remain
  return { items: [] };
}

// ---------- Tests ----------
function assert(cond: unknown, msg: string) {
  if (!cond) throw new Error("ASSERT FAIL: " + msg);
}

function seed() {
  db.length = 0;
  // createdAt ties to force tie-break
  insertOrder({ id: "a", createdAt: 1000, userId: "u1", amount: 10, status: "created" });
  insertOrder({ id: "b", createdAt: 1000, userId: "u1", amount: 20, status: "paid" });
  insertOrder({ id: "c", createdAt: 2000, userId: "u1", amount: 30, status: "paid" });
  insertOrder({ id: "d", createdAt: 3000, userId: "u1", amount: 40, status: "cancelled" });
  insertOrder({ id: "x", createdAt: 1500, userId: "u2", amount: 99, status: "paid" });
}

function testBasic() {
  seed();
  const p1 = listOrders({ userId: "u1", limit: 2 });
  assert(p1.items.map((o) => o.id).join(",") === "a,b", "p1 should be a,b");
  assert(!!p1.nextCursor, "p1 nextCursor exists");

  const p2 = listOrders({ userId: "u1", limit: 2, cursor: p1.nextCursor });
  assert(p2.items.map((o) => o.id).join(",") === "c,d", "p2 should be c,d");
  assert(!p2.nextCursor, "p2 nextCursor empty");
}

function testTieBreak() {
  seed();
  const p = listOrders({ userId: "u1", limit: 10 });
  assert(p.items.map((o) => o.id).join(",") === "a,b,c,d", "tie-break by id asc");
}

function testConcurrentInsertNoSkipNoDup() {
  seed();

  // page 1
  const p1 = listOrders({ userId: "u1", limit: 2 });
  const seen = new Set(p1.items.map((o) => o.id));

  // concurrent insert happens BETWEEN pages
  // insert createdAt=1000 id="bb" -> belongs between b and c in order
  insertOrder({ id: "bb", createdAt: 1000, userId: "u1", amount: 15, status: "paid" });

  // page 2 from cursor of page1 should include NEW item bb? depends on semantics
  // With "after cursor (a,b)" it SHOULD include items after b, so it SHOULD include bb.
  const p2 = listOrders({ userId: "u1", limit: 2, cursor: p1.nextCursor });

  for (const o of p2.items) {
    assert(!seen.has(o.id), "no duplicates across pages");
    seen.add(o.id);
  }

  // continue until done
  let cursor = p2.nextCursor;
  while (cursor) {
    const pn = listOrders({ userId: "u1", limit: 2, cursor });
    for (const o of pn.items) {
      assert(!seen.has(o.id), "no duplicates across pages");
      seen.add(o.id);
    }
    cursor = pn.nextCursor;
  }

  // must have all u1 orders including bb
  assert(seen.has("bb"), "should include concurrently inserted bb");
  assert(seen.size === 5, "u1 should have 5 orders total now");
}

function testInvalidCursor() {
  seed();
  const p = listOrders({ userId: "u1", limit: 2, cursor: "not-a-cursor" });
  // choose your behavior:
  // A) treat invalid cursor as 400 error (better for API)
  // B) treat as undefined (start from beginning)
  // For kata: implement A via throwing an AppError if you want.
  // Here we’ll accept B for simplicity: must still return deterministic first page.
  assert(p.items.map((o) => o.id).join(",") === "a,b", "invalid cursor fallback");
}

function run() {
  testBasic();
  testTieBreak();
  testConcurrentInsertNoSkipNoDup();
  testInvalidCursor();
  console.log("ALL TESTS PASSED");
}

run();
```

---

## Tasks bạn phải làm

1. Implement `encodeCursor/decodeCursor` base64url đúng (không dùng `as`)
2. Implement `listOrders` theo đúng condition “strictly after cursor”
3. Ensure `nextCursor` chỉ set khi còn items sau page hiện tại
4. Decide behavior invalid cursor:

* **Recommended production**: return 400 (AppError validation)
* Kata skeleton đang cho phép fallback; nếu bạn chọn 400 thì sửa tests theo.

---

## Definition of Done

* Tests pass
* Concurrent insert không skip/duplicate (theo semantics “after cursor”)
* Cursor opaque (client không cần hiểu), nhưng decode được server-side
* Sorting stable `(createdAt, id)`

---

## Stretch (Senior+)

1. **Backward pagination** (`prevCursor`)
2. Cursor signed (HMAC) để client không sửa được createdAt/id
3. Filter thêm `status` mà vẫn giữ correctness (cursor phải include filters)
4. “Snapshot consistency”: add `asOf` timestamp trong cursor để “freeze view” nếu muốn không thấy inserts mới