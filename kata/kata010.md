# Kata 10 — TS Error Budget (Zero `any` / Zero `as`)

## Goal

Refactor module hiện có (dưới đây mình đưa “module xấu” sẵn) để đạt:

* `any` = 0
* `as` / type assertion = 0
* Không throw trong domain (optional, recommended)
* Compile-time checks pass
* Runtime behavior giữ nguyên (hoặc rõ ràng hơn)

> Nếu bạn muốn rule “thực tế production”: cho phép **1 assertion duy nhất** trong “branding factory” (như Kata 03). Nhưng kata này mặc định **0 tuyệt đối**.

---

## Constraints

* ❌ Không dùng `any`
* ❌ Không dùng `as` (kể cả `as const` — trong kata này cũng tính là “as”)
* ❌ Không `!` non-null assertion
* ✅ Được dùng:

  * `unknown`
  * type guards (`x is ...`)
  * discriminated unions
  * conditional checks runtime
  * `satisfies` (không phải assertion)
* ✅ Có thể tách code thành nhiều function nhỏ

---

## Deliverables

File `kata10.ts` gồm:

1. **Bad module** (giữ nguyên behavior)
2. **Refactor module** đạt budget
3. `@ts-expect-error` checks + demo cases

---

## The “Bad Module” bạn phải refactor (copy nguyên)

Bạn bắt đầu từ đoạn này:

```ts
/* kata10.ts - BAD MODULE (do not keep as-is in final) */

export function buildReport(input: any) {
  const users = input.users as any[];
  const includeInactive = input.includeInactive as boolean;

  const rows = users
    .filter((u) => includeInactive ? true : u.active)
    .map((u) => {
      const id = u.id as string;
      const email = (u.email as string).toLowerCase();
      const tags = (u.tags as string[] | undefined) || [];
      const createdAt = new Date(u.createdAt as any).getTime();

      return {
        id,
        email,
        isPro: (u.plan as any)?.tier === "pro",
        tagCount: tags.length,
        createdAt,
      };
    });

  return {
    total: rows.length,
    proCount: rows.filter((r) => r.isPro).length,
    rows,
  };
}
```

### Behavior requirements (giữ nguyên logic)

* Input shape mong đợi (nhưng input đến là `unknown`):

  * `users`: array user objects
  * `includeInactive`: boolean (optional, default false)
* Với mỗi user:

  * `id`: string (required)
  * `email`: string (required) → lowercased
  * `active`: boolean (optional, default false)
  * `tags`: string[] (optional, default [])
  * `createdAt`: ISO string hoặc number epoch (required-ish: nếu parse fail → bỏ user hoặc set 0? bạn chọn nhưng phải **consistent**)
  * `plan`: optional object `{ tier: "free"|"pro" }` (tier khác → treat as free)
* Output report:

  * `total`
  * `proCount`
  * `rows: {id,email,isPro,tagCount,createdAt}[]`

---

## Nhiệm vụ refactor (phần “GOOD MODULE”)

Bạn viết lại `buildReport(input: unknown): Result<Report, ValidationError[]>` (recommended) hoặc trả `Report` + defaulting (tuỳ bạn), nhưng phải:

### 1) Không `any` / không `as`

* Tất cả parsing dùng type guards + narrowing

### 2) Parse, normalize ở boundary

* `email` lowercase, trim
* `tags` normalize: string[] only, bỏ non-string
* `createdAt` parse: chấp nhận number hoặc string parseable

### 3) Error budget = 0

* Nếu input sai, trả errors (accumulate) hoặc skip row (nhưng phải quyết định rõ)

---

## Starter skeleton “GOOD MODULE” (điền TODO)

Bạn có thể copy skeleton này ngay dưới bad module (hoặc thay hẳn bad module bằng good module).

```ts
// ---------- Result + Validation ----------
type ValidationError = { field: string; message: string };

type Result<T> =
  | { ok: true; value: T }
  | { ok: false; errors: ValidationError[] };

const ok = <T>(value: T): Result<T> => ({ ok: true, value });
const fail = (...errors: ValidationError[]): Result<never> => ({ ok: false, errors });

// ---------- Domain ----------
type PlanTier = "free" | "pro";

type UserRow = {
  id: string;
  email: string;
  isPro: boolean;
  tagCount: number;
  createdAt: number;
};

type Report = {
  total: number;
  proCount: number;
  rows: UserRow[];
};

// ---------- Narrowing helpers (no assertions) ----------
function isRecord(x: unknown): x is Record<string, unknown> {
  return typeof x === "object" && x !== null;
}

function isString(x: unknown): x is string {
  return typeof x === "string";
}

function isBoolean(x: unknown): x is boolean {
  return typeof x === "boolean";
}

function isNumber(x: unknown): x is number {
  return typeof x === "number" && Number.isFinite(x);
}

function isArray(x: unknown): x is unknown[] {
  return Array.isArray(x);
}

// ---------- Parsers ----------
function parseIncludeInactive(x: unknown): { includeInactive: boolean; errors: ValidationError[] } {
  // default false
  if (x === undefined) return { includeInactive: false, errors: [] };
  if (isBoolean(x)) return { includeInactive: x, errors: [] };
  return { includeInactive: false, errors: [{ field: "includeInactive", message: "must be boolean" }] };
}

function parsePlanTier(x: unknown): PlanTier {
  // treat unknown as free
  if (!isRecord(x)) return "free";
  const tier = x.tier;
  if (tier === "pro") return "pro";
  return "free";
}

function parseCreatedAt(x: unknown): { value?: number; error?: ValidationError } {
  // accept number epoch or parseable string
  if (isNumber(x)) return { value: x };
  if (isString(x)) {
    const t = Date.parse(x);
    if (!Number.isNaN(t)) return { value: t };
    return { error: { field: "createdAt", message: "invalid date string" } };
  }
  return { error: { field: "createdAt", message: "must be number or date string" } };
}

function parseTags(x: unknown): string[] {
  if (!isArray(x)) return [];
  const out: string[] = [];
  for (const v of x) {
    if (isString(v)) out.push(v);
  }
  return out;
}

function parseUser(u: unknown, index: number): Result<{ row: UserRow; active: boolean }> {
  if (!isRecord(u)) return fail({ field: `users[${index}]`, message: "must be object" });

  const id = u.id;
  const email = u.email;

  const errors: ValidationError[] = [];

  if (!isString(id) || id.length === 0) errors.push({ field: `users[${index}].id`, message: "must be non-empty string" });
  if (!isString(email) || email.length === 0) errors.push({ field: `users[${index}].email`, message: "must be non-empty string" });

  const active = isBoolean(u.active) ? u.active : false;

  const created = parseCreatedAt(u.createdAt);
  if (!created.value) errors.push({ field: `users[${index}].createdAt`, message: created.error ? created.error.message : "invalid" });

  if (errors.length) return { ok: false, errors };

  // safe because guards above ensured string
  const emailNorm = email.trim().toLowerCase();
  const tags = parseTags(u.tags);
  const tier = parsePlanTier(u.plan);

  const row: UserRow = {
    id,
    email: emailNorm,
    isPro: tier === "pro",
    tagCount: tags.length,
    createdAt: created.value ?? 0,
  };

  return ok({ row, active });
}

// ---------- GOOD MODULE ----------
export function buildReport(input: unknown): Result<Report> {
  if (!isRecord(input)) return fail({ field: "body", message: "must be object" });

  const usersRaw = input.users;
  if (!isArray(usersRaw)) return fail({ field: "users", message: "must be array" });

  const inc = parseIncludeInactive(input.includeInactive);

  const rows: UserRow[] = [];
  const errors: ValidationError[] = [...inc.errors];

  for (let i = 0; i < usersRaw.length; i += 1) {
    const parsed = parseUser(usersRaw[i], i);
    if (!parsed.ok) {
      errors.push(...parsed.errors);
      continue; // decision: skip invalid users but keep errors
    }
    if (inc.includeInactive || parsed.value.active) rows.push(parsed.value.row);
  }

  if (errors.length) return { ok: false, errors };

  const proCount = rows.reduce((n, r) => n + (r.isPro ? 1 : 0), 0);
  return ok({ total: rows.length, proCount, rows });
}

// ---------- Demo ----------
const demo1: unknown = {
  users: [
    { id: "u1", email: " MAX@EXAMPLE.COM ", active: true, tags: ["dev"], createdAt: "2020-01-01T00:00:00Z", plan: { tier: "pro" } },
    { id: "u2", email: "a@b.com", active: false, createdAt: 1700000000000, plan: { tier: "free" } },
  ],
  includeInactive: false,
};

console.log("demo1:", buildReport(demo1));

const demoBad: unknown = {
  users: [{ id: 123, email: null, createdAt: "nope", tags: [1, "ok"] }],
  includeInactive: "yes",
};

console.log("demoBad:", buildReport(demoBad));
```

---

## Your explicit “budget” checklist

Bạn pass kata nếu:

* Search trong file không có: `any`, `as`, `!` (non-null assertion)
* `buildReport` nhận `unknown`
* Có type guards + parsers tách nhỏ
* Có policy rõ: invalid user → skip + accumulate errors, hoặc fail-fast (nhưng phải consistent)

---

## Stretch (đúng Senior+/Lead)

1. Thay “skip invalid users” bằng **partial success**:

   * `{ ok:true, value: Report, warnings: ValidationError[] }`
2. Dùng `satisfies` để ensure output shape mà không assertion
3. Thêm `email` validation đơn giản, normalize tags unique
4. Thêm `cursor pagination` cho rows (nối Kata 06)