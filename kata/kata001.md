# Kata 01 — Type-level Guardrails (Required keys by rules)

## Goal

Thiết kế **type utilities** để:

1. Ép object **phải có đủ field** theo “rule”
2. **Không được** có field thừa (strict object)
3. Có thể đổi rule theo **mode/variant** (ví dụ: `create` vs `update`)

## Context

Bạn đang viết SDK nội bộ / service layer. Bạn muốn các payload kiểu:

* `createUser` phải có: `email`, `name`
* `updateUser` phải có: `id` và **ít nhất 1** field trong `{email, name, bio}`

Tất cả phải fail ngay khi compile nếu sai.

---

## Constraints

* ❌ Không dùng `any`
* ❌ Không dùng `as any` / type assertion để “lách”
* ✅ Được dùng conditional types, mapped types, generics
* ✅ Có thể dùng `unknown` nếu cần
* ✅ Object literal phải bị bắt lỗi khi có key lạ

---

## Deliverables

Bạn tạo 1 file `kata01.ts` gồm:

1. Type utilities
2. Ví dụ usage
3. Các “test compile-time” (dùng `// @ts-expect-error`)

---

## Tasks

### Task A — Strict object (no extra keys)

Tạo type:

* `Strict<T>`: chỉ cho phép đúng keys của `T` (object literal có key lạ phải lỗi)

**Expectations**

* `Strict<{a: number}>` cho `{a:1}` pass
* `{a:1, b:2}` fail

---

### Task B — Require keys by rule

Tạo type:

* `RequireKeys<T, K extends keyof T>`: ép `K` là required, các keys khác giữ nguyên

**Expectations**

* `RequireKeys<User, "email"|"name">` bắt buộc có `email` + `name`

---

### Task C — AtLeastOne

Tạo type:

* `AtLeastOne<T, K extends keyof T = keyof T>`:

  * trong tập key `K`, phải có **ít nhất 1 key xuất hiện**
  * keys ngoài `K` giữ nguyên

**Expectations**

* `AtLeastOne<Pick<User, "email"|"name"|"bio">>`: phải có ít nhất 1 trong 3

---

### Task D — Build rules for payloads

Cho type `User`:

```ts
type User = {
  id: string;
  email: string;
  name: string;
  bio?: string;
  age?: number;
};
```

Bạn định nghĩa:

* `CreateUserInput`:

  * required: `email`, `name`
  * optional: `bio`, `age`
  * không cho key thừa

* `UpdateUserInput`:

  * required: `id`
  * optional update fields: `email`, `name`, `bio`, `age`
  * nhưng phải có **ít nhất 1 field để update** (ngoài `id`)
  * không cho key thừa

---

## Starter skeleton (bạn điền TODO)

```ts
/* kata01.ts */

// ---------- Domain ----------
type User = {
  id: string;
  email: string;
  name: string;
  bio?: string;
  age?: number;
};

// ---------- TODO: Utilities ----------

// TODO A: Strict<T>
type Strict<T extends object> = T & {
  // TODO: forbid extra keys
};

// TODO B: RequireKeys<T, K>
type RequireKeys<T, K extends keyof T> =
  // TODO
  never;

// TODO C: AtLeastOne<T, K>
type AtLeastOne<T, K extends keyof T = keyof T> =
  // TODO
  never;

// ---------- TODO: Payloads ----------
type CreateUserInput =
  // TODO: Strict + RequireKeys combo
  never;

type UpdateUserInput =
  // TODO: Strict + required id + at least one update field
  never;

// ---------- Compile-time checks ----------

// ---- Strict tests ----
const okStrict: Strict<{ a: number }> = { a: 1 };
// @ts-expect-error - extra key not allowed
const badStrict: Strict<{ a: number }> = { a: 1, b: 2 };

// ---- CreateUserInput tests ----
const okCreate: CreateUserInput = { email: "a@b.com", name: "Max" };
const okCreate2: CreateUserInput = { email: "a@b.com", name: "Max", bio: "hi", age: 18 };

// @ts-expect-error - missing required name
const badCreate1: CreateUserInput = { email: "a@b.com" };

// @ts-expect-error - extra key not allowed
const badCreate2: CreateUserInput = { email: "a@b.com", name: "Max", unknownKey: 123 };

// ---- UpdateUserInput tests ----
const okUpdate1: UpdateUserInput = { id: "u1", bio: "new bio" };
const okUpdate2: UpdateUserInput = { id: "u1", email: "x@y.com", age: 20 };

// @ts-expect-error - must include at least one field to update
const badUpdate1: UpdateUserInput = { id: "u1" };

// @ts-expect-error - extra key not allowed
const badUpdate2: UpdateUserInput = { id: "u1", name: "N", extra: true };

// @ts-expect-error - missing required id
const badUpdate3: UpdateUserInput = { email: "x@y.com" };
```

---

## Checks (Definition of Done)

* `tsc --noEmit` chạy qua **không có lỗi ngoài các dòng `@ts-expect-error`**
* Không dùng `any` / `as any`
* `UpdateUserInput` thật sự không cho `{ id }` đứng một mình

---

## Stretch (nâng cấp nếu còn sức)

1. Làm thêm `Exact<A,B>` để ép type A không chỉ “assignable” mà còn “exact same keys”
2. Thêm rule: `CreateUserInput` cấm `id` (không cho client set id)
3. Thêm mode `replaceUser` bắt buộc đủ mọi field (PUT semantics)