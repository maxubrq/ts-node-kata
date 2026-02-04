# Kata 06 — Generic Repository Contract (No persistence leakage)

## Goal

1. Thiết kế **repository interface** chuẩn cho domain (không nhắc SQL/ORM)
2. Áp dụng **Result pattern** (Kata 04) + typed errors
3. Hỗ trợ tối thiểu:

* `getById`
* `find` (query object, cursor pagination)
* `save` (upsert hoặc create/update rõ)
* `delete`
* **optimistic concurrency** (version)
* **idempotency** (optional stretch)

4. Viết 1 implementation **InMemoryRepository** để test domain logic, chứng minh contract đúng.

---

## Context

Bạn có domain `UserProfile` (từ Kata 05) và muốn lưu/đọc mà:

* Service không phụ thuộc vào DB/ORM
* Có thể swap DB mà không đụng domain/service logic
* Test dễ (in-memory)

---

## Constraints

* ❌ Không trả về “DB row” kiểu `{ _id, created_at, updated_at }` trong domain
* ❌ Không để query nhận raw SQL string
* ✅ Repo chỉ nói bằng **domain types** + query objects
* ✅ `save` phải enforce version để chống lost update
* ✅ Không throw trong repo: trả `Result`

---

## Deliverables

File `kata06.ts` gồm:

1. `Result<T,E>` + helpers tối thiểu (`ok/err`, `andThen`)
2. Domain primitives: `EntityId`, `Version`, `Clock` (optional)
3. `Repository<TEntity, TId, TQuery>` interface
4. `InMemoryRepository` implementation
5. Service `UserProfileService` dùng repo
6. Compile-time checks + demo/test

---

## Domain spec

### Entity base

Mọi entity có:

* `id: TId` (branded)
* `version: number` (optimistic lock)
* `data` (fields domain)

### Errors (repo level)

* `NotFoundError` `{ type:"not_found"; id: string }`
* `ConflictError` `{ type:"conflict"; message: string }` (version mismatch)
* `ValidationError` (chỉ nếu repo enforce invariants – optional)

Gộp `RepoError`.

---

## Repository contract spec

### Required methods

* `getById(id): Result<TEntity, NotFoundError>`
* `find(query, options): Result<{ items: TEntity[]; nextCursor?: string }, never>`
  (in-memory luôn ok; DB cũng thường ok nếu query object hợp lệ)
* `save(entity, expectedVersion?): Result<TEntity, ConflictError>`

  * Nếu `expectedVersion` provided, phải match `entity.version` hiện tại trong store
  * Khi save thành công: `version` tăng `+1`
* `delete(id, expectedVersion?): Result<void, NotFoundError | ConflictError>`

### Query object (không leak persistence)

Cho `UserProfile`:

* filter theo `tag?: string`
* search theo `emailPrefix?: string` (string normalized)
* order: `"created_desc" | "created_asc"`
* pagination: cursor-based (string cursor)

---

## Starter skeleton (điền TODO)

```ts
/* kata06.ts */

// ---------- Result ----------
export type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

export const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
export const err = <E>(error: E): Result<never, E> => ({ ok: false, error });

export function andThen<T, E, U, F>(r: Result<T, E>, fn: (v: T) => Result<U, F>): Result<U, E | F> {
  return r.ok ? fn(r.value) : r;
}

// ---------- Branding ----------
declare const __brand: unique symbol;
type Brand<T, Name extends string> = T & { readonly [__brand]: Name };

export type UserId = Brand<string, "UserId">;
export type Email = Brand<string, "Email">;
export type Tag = Brand<string, "Tag">;

// ---------- Domain entity ----------
export type UserProfile = {
  id: UserId;
  version: number;
  createdAt: number; // domain-level time (not DB columns)
  email: Email;
  name: string;
  tags: Tag[];
  marketingOptIn: boolean;
};

// ---------- Repo errors ----------
export type NotFoundError = { type: "not_found"; id: string };
export type ConflictError = { type: "conflict"; message: string };
export type RepoError = NotFoundError | ConflictError;

// ---------- Query objects (no SQL) ----------
export type UserProfileQuery = {
  tag?: Tag;
  emailPrefix?: string; // normalized lowercase prefix (not Email brand to allow partial)
  order?: "created_desc" | "created_asc";
};

export type FindOptions = {
  limit: number;      // 1..100
  cursor?: string;    // opaque
};

// ---------- Repository Contract ----------
export interface Repository<TEntity, TId, TQuery> {
  getById(id: TId): Result<TEntity, NotFoundError>;

  find(query: TQuery, options: FindOptions): Result<{ items: TEntity[]; nextCursor?: string }, never>;

  save(entity: TEntity, expectedVersion?: number): Result<TEntity, ConflictError>;

  delete(id: TId, expectedVersion?: number): Result<void, NotFoundError | ConflictError>;
}

// ---------- In-memory implementation ----------
type Stored<TEntity> = {
  entity: TEntity;
  // you may store additional internal fields here, but do NOT expose
};

export class InMemoryUserProfileRepo implements Repository<UserProfile, UserId, UserProfileQuery> {
  private byId = new Map<string, Stored<UserProfile>>();

  // cursor strategy: base64 of index (simple), ok for kata
  // TODO: implement encode/decode cursor helpers

  getById(id: UserId): Result<UserProfile, NotFoundError> {
    // TODO
    return err({ type: "not_found", id: String(id) });
  }

  find(query: UserProfileQuery, options: FindOptions): Result<{ items: UserProfile[]; nextCursor?: string }, never> {
    // TODO:
    // - filter by tag if provided
    // - filter by emailPrefix if provided (email is branded string, but stored raw string inside)
    // - sort by createdAt asc/desc
    // - cursor-based pagination
    return ok({ items: [] });
  }

  save(entity: UserProfile, expectedVersion?: number): Result<UserProfile, ConflictError> {
    // TODO:
    // - if id exists:
    //   - if expectedVersion provided and doesn't match stored.version => conflict
    //   - else overwrite with version+1
    // - if id not exist:
    //   - if expectedVersion provided and expectedVersion != 0 => conflict (or reject)
    //   - else insert with version=1 (or entity.version+1, choose consistent rule)
    return err({ type: "conflict", message: "TODO" });
  }

  delete(id: UserId, expectedVersion?: number): Result<void, NotFoundError | ConflictError> {
    // TODO:
    // - if not exist => not_found
    // - if expectedVersion provided and mismatch => conflict
    // - else delete
    return err({ type: "not_found", id: String(id) });
  }
}

// ---------- Service layer (no DB knowledge) ----------
export type CreateUserProfileInput = {
  id: UserId;
  email: Email;
  name: string;
  tags: Tag[];
  marketingOptIn: boolean;
};

export class UserProfileService {
  constructor(private repo: Repository<UserProfile, UserId, UserProfileQuery>) {}

  create(input: CreateUserProfileInput, nowMs: number): Result<UserProfile, ConflictError> {
    const entity: UserProfile = {
      id: input.id,
      version: 0,
      createdAt: nowMs,
      email: input.email,
      name: input.name,
      tags: input.tags,
      marketingOptIn: input.marketingOptIn,
    };

    // rule: create must fail if already exists
    // TODO: use repo.save with expectedVersion = 0 (or define a create method)
    return this.repo.save(entity, 0);
  }

  updateName(id: UserId, expectedVersion: number, newName: string): Result<UserProfile, RepoError> {
    // TODO:
    // - getById
    // - produce new entity with updated name
    // - save with expectedVersion
    return err({ type: "conflict", message: "TODO" });
  }

  listByTag(tag: Tag, limit: number, cursor?: string) {
    return this.repo.find({ tag, order: "created_desc" }, { limit, cursor });
  }
}

// ---------- Demo / checks ----------
const repo = new InMemoryUserProfileRepo();
const svc = new UserProfileService(repo);

// helpers to make branded ids for kata (ok to assert here)
const u = "usr_1a2b3c4d5e6f" as UserId;
const e = "max@example.com" as Email;
const tDev = "dev" as Tag;
const tTs = "typescript" as Tag;

const created = svc.create({ id: u, email: e, name: "Max", tags: [tDev, tTs], marketingOptIn: true }, Date.now());
console.log("created:", created);

// create again should conflict
const createdAgain = svc.create({ id: u, email: e, name: "Max2", tags: [], marketingOptIn: false }, Date.now());
console.log("createdAgain:", createdAgain);

// update with correct version
if (created.ok) {
  const upd1 = svc.updateName(u, created.value.version, "Max Nguyen");
  console.log("upd1:", upd1);

  // update with stale version should conflict
  const upd2 = svc.updateName(u, created.value.version, "Stale Update");
  console.log("upd2:", upd2);
}

// list by tag
console.log("list:", svc.listByTag(tDev, 10));
```

---

## Required behaviors (phải đạt)

1. `create` lần 2 với cùng id → `ConflictError`
2. `updateName` với `expectedVersion` đúng → ok và version tăng
3. `updateName` với `expectedVersion` cũ → conflict
4. `find` theo tag/emailPrefix hoạt động và có `nextCursor` khi còn dữ liệu
5. Service không biết DB/ORM, chỉ nói chuyện bằng repo interface

---

## Checks (Definition of Done)

* Repo interface không nhắc SQL/ORM/collection/table
* Không lộ internal persistence fields ra ngoài domain entity
* Cursor là opaque string (client không dựa vào semantics)
* `save/delete` enforce optimistic concurrency

---

## Stretch (Senior+)

1. **Split contract**: `create` và `update` tách riêng để semantics rõ:

   * `create(entity): Result<entity, Conflict>`
   * `update(id, expectedVersion, patchFn): Result<entity, NotFound|Conflict>`
2. **Idempotency key**: `saveWithIdempotency(key, op)` trả lại result cũ nếu retry
3. **Unit of Work**: `withTransaction(fn)` để compose multiple repo saves
4. **Domain events**: `save` trả `{ entity, events }` (nối sang outbox kata 33)