# Kata 07 — Conditional Types for API (Return type depends on options)

## Goal

Xây 1 API “get user(s)” mà TypeScript suy ra return type theo input options:

* `byId` + `{ include: ... }` → trả **một user** (hoặc `null`)
* `list` + `{ cursor, limit, include }` → trả **page** `{ items, nextCursor }`
* `select` fields → output **chỉ có fields đó**
* `include` relations → output **có thêm** relation tương ứng
* `throwIfNotFound: true` → return type **không còn null**

Không dùng overload bừa bãi; trọng tâm là **conditional types + generics**.

---

## Constraints

* ❌ Không `any`
* ❌ Không “ép kiểu” để lách (`as unknown as ...`)
* ✅ Được dùng conditional types, mapped types, `infer`, generics
* ✅ Compile-time tests bằng `// @ts-expect-error`
* ✅ Runtime implementation có thể stub (kata này focus type)

---

## Domain model (fixed)

```ts
type User = {
  id: string;
  email: string;
  name: string;
  createdAt: number;
};

type Membership = {
  tier: "free" | "pro";
  expiresAt?: number;
};

type UserRelations = {
  membership: Membership;
  // stretch: add "orders" later
};
```

---

## API spec (bạn phải đạt)

Bạn thiết kế function:

```ts
declare function userQuery<Spec extends QuerySpec>(spec: Spec): QueryResult<Spec>;
```

Trong đó `QuerySpec` hỗ trợ 2 mode:

### Mode A — byId

```ts
{ mode: "byId"; id: string; select?: readonly (keyof User)[]; include?: readonly (keyof UserRelations)[]; throwIfNotFound?: boolean }
```

Return:

* mặc định: `SelectedUser & IncludedRelations | null`
* nếu `throwIfNotFound: true`: `SelectedUser & IncludedRelations` (không null)

### Mode B — list

```ts
{ mode: "list"; limit: number; cursor?: string; select?: readonly (keyof User)[]; include?: readonly (keyof UserRelations)[] }
```

Return:

```ts
{ items: (SelectedUser & IncludedRelations)[]; nextCursor?: string }
```

---

## Tasks

### Task A — `PickBySelect`

Nếu `select` không có → lấy full `User`
Nếu `select` có `["id","email"]` → output chỉ có `id,email`

### Task B — `AddIncludes`

Nếu `include` có `"membership"` → output có thêm `{ membership: Membership }`

### Task C — `NullableByThrowFlag`

Nếu `throwIfNotFound: true` → byId return **non-null**
Ngược lại → byId return **nullable**

### Task D — `QueryResult<Spec>`

Tùy `spec.mode` suy ra return type khác nhau (byId vs list)

---

## Starter skeleton (điền TODO)

```ts
/* kata07.ts */

// ---------- Domain ----------
type User = {
  id: string;
  email: string;
  name: string;
  createdAt: number;
};

type Membership = { tier: "free" | "pro"; expiresAt?: number };

type UserRelations = {
  membership: Membership;
};

// ---------- Spec types ----------
type ByIdSpec = {
  mode: "byId";
  id: string;
  select?: readonly (keyof User)[];
  include?: readonly (keyof UserRelations)[];
  throwIfNotFound?: boolean;
};

type ListSpec = {
  mode: "list";
  limit: number;
  cursor?: string;
  select?: readonly (keyof User)[];
  include?: readonly (keyof UserRelations)[];
};

type QuerySpec = ByIdSpec | ListSpec;

// ---------- TODO A: PickBySelect ----------
type PickBySelect<
  Base,
  Select extends readonly (keyof Base)[] | undefined
> =
  // TODO:
  // - if Select is undefined -> Base
  // - else -> Pick<Base, Select[number]>
  never;

// ---------- TODO B: AddIncludes ----------
type AddIncludes<
  Base,
  Include extends readonly (keyof UserRelations)[] | undefined
> =
  // TODO:
  // - if Include undefined -> Base
  // - else -> Base & { [K in Include[number]]: UserRelations[K] }
  never;

// ---------- Helper: final shape for an item ----------
type UserShape<Spec extends { select?: any; include?: any }> =
  AddIncludes<PickBySelect<User, Spec["select"]>, Spec["include"]>;

// ---------- TODO C: NullableByThrowFlag (byId only) ----------
type ByIdReturn<Spec extends ByIdSpec> =
  // TODO:
  // if Spec["throwIfNotFound"] is true -> UserShape<Spec>
  // else -> UserShape<Spec> | null
  never;

// ---------- TODO D: QueryResult ----------
type QueryResult<Spec extends QuerySpec> =
  // TODO:
  // if mode "byId" -> ByIdReturn<Spec>
  // if mode "list" -> { items: UserShape<Spec>[]; nextCursor?: string }
  never;

// ---------- API (runtime can be stubbed) ----------
declare function userQuery<Spec extends QuerySpec>(spec: Spec): QueryResult<Spec>;

// ---------- Compile-time checks ----------

// byId basic: nullable
const u0 = userQuery({ mode: "byId", id: "u1" });
// u0 should be User | null

// byId with select narrows fields
const u1 = userQuery({ mode: "byId", id: "u1", select: ["id", "email"] as const });
// u1 should be {id,email} | null

// @ts-expect-error - name not selected
u1?.name;

// byId include membership
const u2 = userQuery({ mode: "byId", id: "u1", include: ["membership"] as const });
// u2 should have membership when non-null
u2?.membership.tier;

// byId throwIfNotFound true => non-null
const u3 = userQuery({ mode: "byId", id: "u1", throwIfNotFound: true });
// u3 should be User (not null)
u3.email;

// @ts-expect-error - u3 is not nullable
u3?.email;

// list basic
const p0 = userQuery({ mode: "list", limit: 10 });
// p0.items: User[]
p0.items[0].id;

// list with select + include
const p1 = userQuery({
  mode: "list",
  limit: 10,
  select: ["id"] as const,
  include: ["membership"] as const,
});
// items element should have id + membership only
p1.items[0].id;
p1.items[0].membership.tier;
// @ts-expect-error - email not selected
p1.items[0].email;
```

---

## Definition of Done

* Không `any`
* Tất cả `@ts-expect-error` đúng (TS phải báo lỗi ở đó)
* `userQuery(...)` suy ra type đúng theo `mode/select/include/throwIfNotFound`

---

## Stretch (rất “SDK-grade”)

1. `include` phụ thuộc `mode` (vd list không cho include “heavy”)
2. `select` = `"*"` hoặc `{ user: [...], membership: [...] }` (nested select)
3. Thêm `mode: "byEmail"` và reuse toàn bộ hệ type inference
4. Make `throwIfNotFound` default theo env (dev vs prod) mà type vẫn đúng