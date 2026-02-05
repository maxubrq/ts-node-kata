# Kata 39 — Data Validation in DB: Constraints vs App validation

## Goal

1. Xây một mini-system có **cùng một domain rule**, nhưng enforce ở 2 nơi:

* App-layer validation (schema/type checks)
* DB-layer constraints (simulated constraints)

2. Chứng minh bằng tests:

* app validation có thể bị bypass (multi-writer / bug / script)
* DB constraint là “last line of defense”
* nhưng DB constraint cũng có trade-off: error UX, migration pain, coupling schema

3. Viết **Decision Matrix** (1 trang):

* rule nào nên ở DB
* rule nào nên ở app
* rule nào nên ở cả hai

---

## Constraints

* ✅ Có ít nhất 3 loại rule khác nhau:

  1. **Shape/format** (email format / id pattern)
  2. **Uniqueness** (unique email per tenant)
  3. **Cross-field / invariant** (status transition / totals)
* ✅ Có 2 writers:

  * `AppWriter` (có validation)
  * `ScriptWriter` (bỏ validation, simulate direct DB write)
* ✅ DB phải có constraint checks khi insert/update (throw error codes).
* ✅ Có mapping lỗi ra Problem+JSON style (nối Kata 26) (nhẹ thôi).
* ❌ Không “làm màu”: phải chứng minh bypass thực sự bằng test.

---

## Context (Scenario demo)

Bảng `users` multi-tenant:

User:

* `tenant_id`
* `user_id`
* `email`
* `status`: `ACTIVE | DISABLED`
* `created_at`

Rules:

1. Email phải lowercase + match regex đơn giản.
2. Email unique **trong 1 tenant**: `(tenant_id, email)` unique.
3. Status transition: `DISABLED` không được quay lại `ACTIVE` (immutable once disabled).

---

## Deliverables

`kata-39/`

* `src/db.ts` (fake DB + constraints)
* `src/validators.ts` (app validation)
* `src/writers.ts` (AppWriter vs ScriptWriter)
* `src/errors.ts` (DB error codes + Problem mapping)
* `src/decision.md` (decision matrix 1 trang)
* `test/kata39.test.ts` (bypass + constraints + trade-offs)

---

## “Nhìn vào làm được” table

| Phần | Bạn làm gì     | Output                       | Checks                   |
| ---- | -------------- | ---------------------------- | ------------------------ |
| 1    | Define rules   | 3 rules above                | clear invariants         |
| 2    | App validation | `validateCreateUser`         | catches common bad input |
| 3    | DB constraints | unique + transition + format | blocks bypass            |
| 4    | Two writers    | app vs script                | bypass demonstrated      |
| 5    | Decision doc   | matrix + rationale           | grounded trade-offs      |

---

## Starter Skeleton (điền TODO)

### `src/errors.ts`

```ts
export type DbErrorCode =
  | "CHK_EMAIL_FORMAT"
  | "UQ_TENANT_EMAIL"
  | "CHK_STATUS_IMMUTABLE";

export class DbConstraintError extends Error {
  constructor(public code: DbErrorCode, message?: string) {
    super(message ?? code);
  }
}

export function toProblemJson(e: unknown) {
  if (e instanceof DbConstraintError) {
    // minimal Problem+JSON-like
    const map: Record<DbErrorCode, { status: number; title: string }> = {
      CHK_EMAIL_FORMAT: { status: 400, title: "Invalid email format" },
      UQ_TENANT_EMAIL: { status: 409, title: "Email already exists in tenant" },
      CHK_STATUS_IMMUTABLE: { status: 409, title: "Status transition not allowed" },
    };

    const m = map[e.code];
    return {
      status: m.status,
      body: {
        type: "about:blank",
        title: m.title,
        status: m.status,
        detail: e.message,
        code: e.code,
      },
    };
  }

  return {
    status: 500,
    body: { type: "about:blank", title: "Internal Error", status: 500 },
  };
}
```

### `src/db.ts`

```ts
import { DbConstraintError } from "./errors";

export type UserStatus = "ACTIVE" | "DISABLED";

export type UserRow = {
  tenant_id: string;
  user_id: string;
  email: string;
  status: UserStatus;
  created_at: number;
};

const EMAIL_RE = /^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$/;

export class FakeDb {
  users = new Map<string, UserRow>(); // key = tenant|user_id

  // for unique constraint
  private tenantEmailIndex = new Map<string, string>(); // key = tenant|email -> user_id

  insertUser(row: UserRow) {
    // --- DB constraints (last line of defense) ---

    // (1) email format + lowercase
    if (!EMAIL_RE.test(row.email) || row.email !== row.email.toLowerCase()) {
      throw new DbConstraintError("CHK_EMAIL_FORMAT", `email=${row.email}`);
    }

    // (2) unique (tenant_id, email)
    const uqKey = `${row.tenant_id}|${row.email}`;
    if (this.tenantEmailIndex.has(uqKey)) {
      throw new DbConstraintError("UQ_TENANT_EMAIL", `tenant=${row.tenant_id} email=${row.email}`);
    }

    const pk = `${row.tenant_id}|${row.user_id}`;
    this.users.set(pk, { ...row });
    this.tenantEmailIndex.set(uqKey, row.user_id);
  }

  updateUser(tenant_id: string, user_id: string, patch: Partial<UserRow>) {
    const pk = `${tenant_id}|${user_id}`;
    const cur = this.users.get(pk);
    if (!cur) throw new Error("USER_NOT_FOUND");

    const next: UserRow = { ...cur, ...patch };

    // (3) status transition immutability
    if (cur.status === "DISABLED" && next.status === "ACTIVE") {
      throw new DbConstraintError("CHK_STATUS_IMMUTABLE", `cannot re-activate user=${user_id}`);
    }

    // handle email updates with constraints as well
    if (patch.email && patch.email !== cur.email) {
      if (!EMAIL_RE.test(next.email) || next.email !== next.email.toLowerCase()) {
        throw new DbConstraintError("CHK_EMAIL_FORMAT", `email=${next.email}`);
      }

      // unique check for new email
      const newUqKey = `${tenant_id}|${next.email}`;
      if (this.tenantEmailIndex.has(newUqKey)) {
        throw new DbConstraintError("UQ_TENANT_EMAIL", `tenant=${tenant_id} email=${next.email}`);
      }

      // update unique index: delete old, add new
      const oldUqKey = `${tenant_id}|${cur.email}`;
      this.tenantEmailIndex.delete(oldUqKey);
      this.tenantEmailIndex.set(newUqKey, user_id);
    }

    this.users.set(pk, next);
  }

  // “Unsafe” direct write to simulate raw SQL bypassing app validation:
  // (Still goes through DB constraints, because DB always enforces.)
  scriptInsertUser(row: UserRow) {
    this.insertUser(row);
  }
}
```

### `src/validators.ts`

```ts
const EMAIL_RE = /^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$/;

export function validateCreateUser(input: { email: string }) {
  const email = input.email.trim();

  // app-layer policy: normalize
  const normalized = email.toLowerCase();

  if (!EMAIL_RE.test(normalized)) {
    return { ok: false as const, error: "INVALID_EMAIL" };
  }

  return { ok: true as const, email: normalized };
}
```

### `src/writers.ts`

```ts
import type { FakeDb } from "./db";
import { validateCreateUser } from "./validators";

export class AppWriter {
  constructor(private readonly db: FakeDb) {}

  createUser(input: { tenant_id: string; user_id: string; email: string }) {
    const v = validateCreateUser({ email: input.email });
    if (!v.ok) return { ok: false as const, error: v.error };

    // write
    this.db.insertUser({
      tenant_id: input.tenant_id,
      user_id: input.user_id,
      email: v.email,
      status: "ACTIVE",
      created_at: Date.now(),
    });

    return { ok: true as const };
  }

  disableUser(input: { tenant_id: string; user_id: string }) {
    this.db.updateUser(input.tenant_id, input.user_id, { status: "DISABLED" });
  }
}

// ScriptWriter simulates a second writer (cron, admin script, another service)
export class ScriptWriter {
  constructor(private readonly db: FakeDb) {}

  // intentionally NO validation/normalization
  createUserRaw(row: { tenant_id: string; user_id: string; email: string }) {
    this.db.scriptInsertUser({
      tenant_id: row.tenant_id,
      user_id: row.user_id,
      email: row.email, // can be uppercase/bad
      status: "ACTIVE",
      created_at: Date.now(),
    });
  }

  updateUserRaw(tenant_id: string, user_id: string, patch: any) {
    this.db.updateUser(tenant_id, user_id, patch);
  }
}
```

### `src/decision.md` (template bắt buộc)

```md
# Data validation placement: DB vs App

## Rule 1: Email format + lowercase
- App: YES (fast feedback, normalize, good UX)
- DB: YES (defense-in-depth against other writers)
- Why: multi-writer + bug-proofing

## Rule 2: Unique (tenant_id, email)
- App: MAYBE (pre-check for nicer errors, but race-prone)
- DB: YES (only DB can guarantee under concurrency)
- Why: concurrency & correctness

## Rule 3: Status immutability (DISABLED cannot become ACTIVE)
- App: YES (domain guardrail)
- DB: YES if critical (prevents bypass)
- Trade-off: DB constraint makes rare admin recovery harder; might need explicit “reactivate” flow

## What I accept
- DB errors are less friendly; app should map to Problem+JSON and add context
- Migrations: adding constraints must be phased (validate existing data first)
```

---

## Tests (vitest) — prove bypass & trade-offs

### `test/kata39.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { FakeDb } from "../src/db";
import { AppWriter, ScriptWriter } from "../src/writers";
import { toProblemJson } from "../src/errors";

describe("kata39 - constraints vs app validation", () => {
  it("app validation catches obvious invalid input early", async () => {
    const db = new FakeDb();
    const app = new AppWriter(db);

    const r = app.createUser({ tenant_id: "t1", user_id: "u1", email: "NOT_AN_EMAIL" });
    expect(r.ok).toBe(false);
    expect(db.users.size).toBe(0);
  });

  it("DB constraint blocks bypass from a second writer", async () => {
    const db = new FakeDb();
    const script = new ScriptWriter(db);

    try {
      script.createUserRaw({ tenant_id: "t1", user_id: "u1", email: "UPPER@MAIL.COM" });
      throw new Error("should not reach");
    } catch (e) {
      const p = toProblemJson(e);
      expect(p.status).toBe(400);
      expect((p.body as any).code).toBe("CHK_EMAIL_FORMAT");
    }

    expect(db.users.size).toBe(0);
  });

  it("unique constraint must be in DB (race-safe): prevents duplicate email in same tenant", async () => {
    const db = new FakeDb();
    const app = new AppWriter(db);

    expect(app.createUser({ tenant_id: "t1", user_id: "u1", email: "a@x.com" }).ok).toBe(true);

    // second user same email same tenant -> conflict
    try {
      app.createUser({ tenant_id: "t1", user_id: "u2", email: "a@x.com" });
      throw new Error("should not reach");
    } catch (e) {
      const p = toProblemJson(e);
      expect(p.status).toBe(409);
      expect((p.body as any).code).toBe("UQ_TENANT_EMAIL");
    }

    // same email different tenant should be allowed
    expect(app.createUser({ tenant_id: "t2", user_id: "u1", email: "a@x.com" }).ok).toBe(true);
  });

  it("status immutability enforced even if script tries to bypass", async () => {
    const db = new FakeDb();
    const app = new AppWriter(db);
    const script = new ScriptWriter(db);

    expect(app.createUser({ tenant_id: "t1", user_id: "u1", email: "a@x.com" }).ok).toBe(true);
    app.disableUser({ tenant_id: "t1", user_id: "u1" });

    // script attempts to re-activate -> DB blocks
    await expect(async () => {
      script.updateUserRaw("t1", "u1", { status: "ACTIVE" });
    }).rejects.toThrow("CHK_STATUS_IMMUTABLE");
  });
});
```

---

## Definition of Done (Checks)

Bạn pass kata này khi:

1. App validation bắt được input sai (format).
2. ScriptWriter bypass app nhưng DB vẫn chặn được (format + invariant).
3. Unique constraint đúng scope tenant: same tenant conflict, different tenant ok.
4. Status transition invariant được enforce ngay cả khi bypass.

---

## Stretch (Senior+)

1. **Pre-check unique ở app nhưng vẫn dựa DB**

* app query “email exists?” để UX tốt hơn, nhưng vẫn handle DB conflict vì race.

2. **Constraint rollout theo Expand/Contract**

* Phase A: add constraint NOT VALID / validate existing data (DB thật)
* Phase B: enforce for new writes
* Phase C: validate + make strict

3. **Error taxonomy**

* map constraint errors → Problem+JSON với field-level detail (`field=email`).

4. **Multi-writer reality**

* thêm “ServiceBWriter” dùng normalization khác → chỉ DB constraint giữ consistency.