# Kata 37 — Schema Migration Safe: Expand/Contract migration plan

## Goal

1. Thiết kế & mô phỏng một migration theo **expand/contract**:

* Phase 1 (Expand): thêm schema mới **không phá** code cũ
* Phase 2 (Backfill): fill dữ liệu dần, idempotent
* Phase 3 (Dual-write + Dual-read): app đọc/ghi cả cũ & mới (tạm thời)
* Phase 4 (Cutover): chuyển read sang mới
* Phase 5 (Contract): xoá cột cũ sau khi an toàn

2. Có **rollback plan** rõ ràng cho từng phase.

3. Có tests chứng minh:

* app version cũ + schema mới vẫn chạy (backward compatibility)
* app version mới + schema cũ (trong giới hạn phase) vẫn chạy (forward compatibility)
* backfill idempotent + resumable
* cutover không làm sai dữ liệu

---

## Constraints

* ✅ Không downtime (assume rolling deploy).
* ✅ Không đổi kiểu dữ liệu/cột “in-place” gây break.
* ✅ Mỗi phase phải có: **migration script**, **deploy action**, **verify**, **rollback**.
* ✅ Backfill phải chạy theo batch + checkpoint.
* ❌ Không “ALTER TABLE ... DROP COLUMN” ngay lập tức.

---

## Context (bài toán demo thực tế)

Bạn có bảng `users` hiện tại:

### Before (v1)

* `id`
* `full_name` (vd: `"Nguyen Van A"`)
* `created_at`

Business muốn chuẩn hoá thành:

* `first_name`
* `last_name`

Yêu cầu:

* trong quá trình rollout, v1 app vẫn dùng `full_name`
* v2 app dùng `first_name/last_name`
* cuối cùng bỏ `full_name`

---

## Deliverables

`kata-37/`

* `src/db.ts` (fake relational-ish table + schema flags)
* `src/app_v1.ts` (old app: read/write full_name)
* `src/app_v2.ts` (new app: read/write first/last + compatibility)
* `src/migrations.ts` (expand/contract + backfill)
* `src/backfill.ts` (batch backfill + checkpoint)
* `src/plan.md` (1 page: phases + verify + rollback)
* `test/kata37.test.ts` (phases simulation)

> Mình dùng fake DB để test logic; trong đời thật bạn thay bằng SQL migrations.

---

## “Nhìn vào làm được” Plan table (5 phases)

| Phase          | DB change                                           | App change         | Verify                | Rollback                                      |
| -------------- | --------------------------------------------------- | ------------------ | --------------------- | --------------------------------------------- |
| 1 Expand       | Add nullable `first_name`, `last_name`              | v1 unchanged       | v1 still works        | revert app only                               |
| 2 Backfill     | Fill new cols from `full_name` in batches           | v1 still           | new cols populated    | rerun backfill idempotent                     |
| 3 Dual-write   | v2 writes both (full + first/last)                  | deploy v2          | no drift between cols | rollback to v1 safe (full_name still correct) |
| 4 Cutover read | v2 reads from new (fallback old)                    | deploy v2 read-new | compare read parity   | toggle back to old read                       |
| 5 Contract     | drop `full_name` (or stop writing, then drop later) | v2 only new        | no code path uses old | postpone drop if unsure                       |

---

## Starter Skeleton

### `src/db.ts`

```ts
export type UserRow = {
  id: string;
  // old
  full_name?: string;

  // new
  first_name?: string | null;
  last_name?: string | null;

  created_at: number;
};

export type Schema = {
  has_full_name: boolean;
  has_first_last: boolean;
};

export class FakeUserTable {
  private rows = new Map<string, UserRow>();
  public schema: Schema = { has_full_name: true, has_first_last: false };

  insert(row: UserRow) {
    this.rows.set(row.id, { ...row });
  }

  get(id: string): UserRow | null {
    const r = this.rows.get(id);
    return r ? { ...r } : null;
  }

  update(id: string, patch: Partial<UserRow>) {
    const cur = this.rows.get(id);
    if (!cur) throw new Error("USER_NOT_FOUND");
    this.rows.set(id, { ...cur, ...patch });
  }

  allIds(): string[] {
    return [...this.rows.keys()];
  }
}
```

### `src/app_v1.ts` (legacy)

```ts
import type { FakeUserTable } from "./db";

export async function createUserV1(db: FakeUserTable, input: { id: string; full_name: string }) {
  // v1 expects full_name column
  if (!db.schema.has_full_name) throw new Error("SCHEMA_MISSING_full_name");

  db.insert({
    id: input.id,
    full_name: input.full_name,
    created_at: Date.now(),
  });
}

export async function getDisplayNameV1(db: FakeUserTable, id: string): Promise<string> {
  const u = db.get(id);
  if (!u) throw new Error("USER_NOT_FOUND");
  if (!db.schema.has_full_name) throw new Error("SCHEMA_MISSING_full_name");
  return u.full_name ?? "";
}
```

### `src/app_v2.ts` (compatible v2)

```ts
import type { FakeUserTable } from "./db";

function splitName(full: string): { first: string; last: string } {
  const s = full.trim().replace(/\s+/g, " ");
  const parts = s.split(" ");
  if (parts.length === 1) return { first: parts[0], last: "" };
  return { first: parts[0], last: parts.slice(1).join(" ") };
}

function joinName(first: string, last: string): string {
  return (first + " " + last).trim();
}

// Phase 3+: dual-write
export async function createUserV2(db: FakeUserTable, input: { id: string; full_name?: string; first_name?: string; last_name?: string }) {
  const now = Date.now();

  // v2 tries to write new columns if available; also keep full_name for rollback safety
  const firstLastAvailable = db.schema.has_first_last;
  const fullAvailable = db.schema.has_full_name;

  let first = input.first_name ?? "";
  let last = input.last_name ?? "";

  if ((!first || !last) && input.full_name) {
    const sp = splitName(input.full_name);
    first ||= sp.first;
    last ||= sp.last;
  }

  const full = input.full_name ?? joinName(first, last);

  const row: any = { id: input.id, created_at: now };

  if (fullAvailable) row.full_name = full;
  if (firstLastAvailable) { row.first_name = first; row.last_name = last; }

  // forward compatible: if schema missing new cols, still can insert using full_name
  if (!fullAvailable && !firstLastAvailable) throw new Error("NO_COMPATIBLE_SCHEMA");

  db.insert(row);
}

// Phase 4+: read new, fallback old
export async function getDisplayNameV2(db: FakeUserTable, id: string): Promise<string> {
  const u = db.get(id);
  if (!u) throw new Error("USER_NOT_FOUND");

  if (db.schema.has_first_last && (u.first_name != null || u.last_name != null)) {
    return joinName(u.first_name ?? "", u.last_name ?? "");
  }

  if (db.schema.has_full_name) return u.full_name ?? "";

  return "";
}
```

### `src/migrations.ts`

```ts
import type { FakeUserTable } from "./db";

// Phase 1: Expand
export function expand_addFirstLast(db: FakeUserTable) {
  db.schema.has_first_last = true;
  // existing rows will just have null/undefined first/last
}

// Phase 5: Contract (done last)
export function contract_dropFullName(db: FakeUserTable) {
  // In real DB: DROP COLUMN full_name.
  // Here: mark as absent and erase data to simulate.
  db.schema.has_full_name = false;

  for (const id of db.allIds()) {
    const u = db.get(id)!;
    delete (u as any).full_name;
    db.update(id, u);
  }
}
```

### `src/backfill.ts`

```ts
import type { FakeUserTable } from "./db";

function splitName(full: string): { first: string; last: string } {
  const s = full.trim().replace(/\s+/g, " ");
  const parts = s.split(" ");
  if (parts.length === 1) return { first: parts[0], last: "" };
  return { first: parts[0], last: parts.slice(1).join(" ") };
}

export type BackfillState = {
  cursor: number; // index in ids list
};

export function backfillBatch(db: FakeUserTable, state: BackfillState, batchSize: number): BackfillState {
  if (!db.schema.has_first_last) throw new Error("SCHEMA_MISSING_first_last");
  if (!db.schema.has_full_name) throw new Error("SCHEMA_MISSING_full_name");

  const ids = db.allIds().sort();
  const start = state.cursor;
  const end = Math.min(ids.length, start + batchSize);

  for (let i = start; i < end; i++) {
    const id = ids[i];
    const u = db.get(id)!;

    // idempotent: only fill if missing
    if ((u.first_name == null || u.first_name === "") && (u.last_name == null || u.last_name === "")) {
      const full = u.full_name ?? "";
      if (full.trim().length > 0) {
        const sp = splitName(full);
        db.update(id, { first_name: sp.first, last_name: sp.last });
      } else {
        db.update(id, { first_name: "", last_name: "" });
      }
    }
  }

  return { cursor: end };
}
```

### `src/plan.md` (template bắt buộc – bạn điền ngày/thời gian)

```md
# Expand/Contract Migration Plan: users.full_name -> users.first_name/last_name

## Phase 1 (Expand)
- DB: add nullable first_name, last_name
- Deploy: no app change required (v1 keeps working)
- Verify:
  - v1 create/read still OK
  - new cols exist
- Rollback: revert DB migration not required; just rollback app if needed

## Phase 2 (Backfill)
- Run backfill job in batches with checkpoint
- Verify:
  - %filled metric increases
  - spot-check correctness
- Rollback:
  - stop job; safe to resume (idempotent)

## Phase 3 (Dual-write)
- Deploy v2 writing both full_name and first/last
- Verify:
  - write parity checks (join(first,last) == full_name)
- Rollback:
  - rollback to v1 (full_name preserved)

## Phase 4 (Cutover read)
- Switch reads to new columns with fallback to full_name
- Verify:
  - compare response parity via shadow reads or sampling
- Rollback:
  - flip feature flag to read old

## Phase 5 (Contract)
- Stop writing full_name
- Wait >= N days
- Drop column full_name
- Verify:
  - no code path reads full_name
- Rollback:
  - postpone drop; if already dropped, restore from backup (expensive)
```

---

## Tests (vitest) — mô phỏng rolling deploy

### `test/kata37.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { FakeUserTable } from "../src/db";
import { createUserV1, getDisplayNameV1 } from "../src/app_v1";
import { createUserV2, getDisplayNameV2 } from "../src/app_v2";
import { expand_addFirstLast, contract_dropFullName } from "../src/migrations";
import { backfillBatch } from "../src/backfill";

describe("kata37 - schema migration safe (expand/contract)", () => {
  it("Phase1 Expand: v1 app still works with new columns added", async () => {
    const db = new FakeUserTable();
    await createUserV1(db, { id: "u1", full_name: "Nguyen Van A" });

    expand_addFirstLast(db); // add new cols

    // v1 still reads/writes full_name OK
    await createUserV1(db, { id: "u2", full_name: "Tran Thi B" });
    expect(await getDisplayNameV1(db, "u2")).toBe("Tran Thi B");
  });

  it("Phase2 Backfill: fills first/last idempotently and resumably", async () => {
    const db = new FakeUserTable();
    await createUserV1(db, { id: "u1", full_name: "Nguyen Van A" });
    await createUserV1(db, { id: "u2", full_name: "Tran Thi B" });

    expand_addFirstLast(db);

    let st = { cursor: 0 };
    st = backfillBatch(db, st, 1);
    st = backfillBatch(db, st, 1);
    expect(st.cursor).toBe(2);

    const u1 = db.get("u1")!;
    expect(u1.first_name).toBe("Nguyen");
    expect(u1.last_name).toBe("Van A");

    // rerun should not change (idempotent)
    const st2 = backfillBatch(db, { cursor: 0 }, 2);
    expect(st2.cursor).toBe(2);
    expect(db.get("u1")!.first_name).toBe("Nguyen");
  });

  it("Phase3/4: v2 dual-write + read-new-fallback-old works during transition", async () => {
    const db = new FakeUserTable();

    // start with old schema only
    await createUserV1(db, { id: "u1", full_name: "Nguyen Van A" });

    // expand + backfill
    expand_addFirstLast(db);
    backfillBatch(db, { cursor: 0 }, 10);

    // v2 creates user (dual-write)
    await createUserV2(db, { id: "u2", first_name: "Tran", last_name: "Thi B" });

    // v2 reads from new
    expect(await getDisplayNameV2(db, "u2")).toBe("Tran Thi B");

    // v2 reads fallback old if needed
    expect(await getDisplayNameV2(db, "u1")).toBe("Nguyen Van A");
  });

  it("Phase5 Contract: after dropping full_name, v2 still works; v1 fails (as expected)", async () => {
    const db = new FakeUserTable();
    await createUserV1(db, { id: "u1", full_name: "Nguyen Van A" });

    expand_addFirstLast(db);
    backfillBatch(db, { cursor: 0 }, 10);

    contract_dropFullName(db);

    // v2 should still read name via first/last
    expect(await getDisplayNameV2(db, "u1")).toBe("Nguyen Van A");

    // v1 should now fail because schema missing full_name
    await expect(getDisplayNameV1(db, "u1")).rejects.toThrow("SCHEMA_MISSING_full_name");
  });
});
```

---

## TODO / Decisions bạn phải chốt

1. **Name splitting**: heuristic (first token = first_name, rest = last_name) — đủ cho kata.
2. **Parity check** (Stretch): trong Phase 3, assert `full_name == join(first,last)` ở write path (log metric nếu mismatch).
3. **When to drop**: Phase 5 chỉ làm khi telemetry cho thấy **0 traffic v1** + đủ thời gian rollback window.

---

## Definition of Done (Checks)

Bạn pass kata này khi:

* v1 chạy được qua Phase 1 & 2 (expand/backfill).
* v2 chạy được xuyên Phase 3 & 4 (dual-write/read cutover).
* Phase 5 drop `full_name` xong:

  * v2 vẫn đọc đúng
  * v1 fail (expected) => chứng minh contract change là thật.
* Backfill chạy batch + resumable + idempotent.

---

## Stretch (Senior+)

1. **Feature flag cutover**

* `READ_FROM_NEW=true/false` để rollback Phase 4 không cần redeploy.

2. **Online verification**

* shadow read: đọc cả 2, compare, log mismatch rate.

3. **Write path guard**

* nếu `first/last` present mà `full_name` khác ⇒ reject hoặc normalize.

4. **Big table safety**

* add index concurrently (đời thật)
* backfill theo `WHERE first_name IS NULL` + limit
* checkpoint bằng primary key (không dùng offset)