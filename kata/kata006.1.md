# Bonus 06.1 — Versioned Update API

### Goal

* Thêm method `update(...)` vào repo contract:

  * check `expectedVersion`
  * apply `mutate(current)`
  * bump `version`
  * persist
* Service layer chỉ mô tả “muốn đổi gì”, không còn “cách đổi”.

---

## Spec

### Repo contract (mở rộng)

```ts
update(
  id: TId,
  expectedVersion: number,
  mutate: (current: TEntity) => TEntity
): Result<TEntity, NotFoundError | ConflictError>;
```

### Rules

1. Nếu không tìm thấy `id` → `NotFoundError`
2. Nếu `current.version !== expectedVersion` → `ConflictError`
3. `mutate` chạy trên entity hiện tại (đã đúng domain type)
4. Repo là nơi duy nhất bump version: `version + 1`
5. `mutate` **không được phép** thay `id` (có check runtime đơn giản)
6. Repo không throw; nếu `mutate` throw (bug), bạn có thể:

   * để throw (programmer error) **hoặc**
   * catch và map thành `ConflictError` (không khuyến khích vì che bug)
     → Bài này chọn: **để throw** (đúng tinh thần “bug phải crash sớm”).

---

## Implementation drop-in (dựa trên Kata 06)

> Bạn có thể thay thế repo cũ bằng version này hoặc add method vào class cũ.

```ts
/* kata06_bonus_1.ts */

// ---------- Result ----------
export type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

export const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
export const err = <E>(error: E): Result<never, E> => ({ ok: false, error });

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
  createdAt: number;
  email: Email;
  name: string;
  tags: Tag[];
  marketingOptIn: boolean;
};

// ---------- Repo errors ----------
export type NotFoundError = { type: "not_found"; id: string };
export type ConflictError = { type: "conflict"; message: string };
export type RepoError = NotFoundError | ConflictError;

// ---------- Query objects ----------
export type UserProfileQuery = {
  tag?: Tag;
  emailPrefix?: string;
  order?: "created_desc" | "created_asc";
};

export type FindOptions = { limit: number; cursor?: string };

// ---------- Repository contract (extended) ----------
export interface Repository<TEntity, TId, TQuery> {
  getById(id: TId): Result<TEntity, NotFoundError>;
  find(query: TQuery, options: FindOptions): Result<{ items: TEntity[]; nextCursor?: string }, never>;
  save(entity: TEntity, expectedVersion?: number): Result<TEntity, ConflictError>;
  delete(id: TId, expectedVersion?: number): Result<void, NotFoundError | ConflictError>;

  // ✅ BONUS 06.1
  update(
    id: TId,
    expectedVersion: number,
    mutate: (current: TEntity) => TEntity
  ): Result<TEntity, NotFoundError | ConflictError>;
}

// ---------- In-memory repo ----------
type Stored<TEntity> = { entity: TEntity };

function encodeCursor(index: number): string {
  // opaque enough for kata
  return Buffer.from(String(index), "utf8").toString("base64");
}
function decodeCursor(cursor: string | undefined): number {
  if (!cursor) return 0;
  const s = Buffer.from(cursor, "base64").toString("utf8");
  const n = Number(s);
  return Number.isFinite(n) && n >= 0 ? n : 0;
}

export class InMemoryUserProfileRepo implements Repository<UserProfile, UserId, UserProfileQuery> {
  private byId = new Map<string, Stored<UserProfile>>();

  getById(id: UserId): Result<UserProfile, NotFoundError> {
    const key = id as unknown as string;
    const found = this.byId.get(key);
    if (!found) return err({ type: "not_found", id: key });
    return ok(found.entity);
  }

  find(query: UserProfileQuery, options: FindOptions): Result<{ items: UserProfile[]; nextCursor?: string }, never> {
    const limit = Math.max(1, Math.min(100, options.limit));
    const start = decodeCursor(options.cursor);

    let items = Array.from(this.byId.values()).map((s) => s.entity);

    if (query.tag) {
      const tag = query.tag as unknown as string;
      items = items.filter((u) => u.tags.some((t) => (t as unknown as string) === tag));
    }
    if (query.emailPrefix) {
      const p = query.emailPrefix.toLowerCase();
      items = items.filter((u) => ((u.email as unknown) as string).startsWith(p));
    }

    const order = query.order ?? "created_desc";
    items.sort((a, b) => (order === "created_asc" ? a.createdAt - b.createdAt : b.createdAt - a.createdAt));

    const page = items.slice(start, start + limit);
    const next = start + limit < items.length ? encodeCursor(start + limit) : undefined;

    return ok({ items: page, nextCursor: next });
  }

  save(entity: UserProfile, expectedVersion?: number): Result<UserProfile, ConflictError> {
    const key = entity.id as unknown as string;
    const found = this.byId.get(key);

    if (!found) {
      // create path
      if (expectedVersion !== undefined && expectedVersion !== 0) {
        return err({ type: "conflict", message: `expectedVersion=${expectedVersion} for create must be 0` });
      }
      const created: UserProfile = { ...entity, version: 1 };
      this.byId.set(key, { entity: created });
      return ok(created);
    }

    // update path
    const current = found.entity;
    if (expectedVersion !== undefined && current.version !== expectedVersion) {
      return err({ type: "conflict", message: `version mismatch: current=${current.version}, expected=${expectedVersion}` });
    }

    const updated: UserProfile = { ...entity, version: current.version + 1 };
    this.byId.set(key, { entity: updated });
    return ok(updated);
  }

  delete(id: UserId, expectedVersion?: number): Result<void, NotFoundError | ConflictError> {
    const key = id as unknown as string;
    const found = this.byId.get(key);
    if (!found) return err({ type: "not_found", id: key });

    const current = found.entity;
    if (expectedVersion !== undefined && current.version !== expectedVersion) {
      return err({ type: "conflict", message: `version mismatch: current=${current.version}, expected=${expectedVersion}` });
    }

    this.byId.delete(key);
    return ok(undefined);
  }

  // ✅ BONUS 06.1 implementation
  update(
    id: UserId,
    expectedVersion: number,
    mutate: (current: UserProfile) => UserProfile
  ): Result<UserProfile, NotFoundError | ConflictError> {
    const key = id as unknown as string;
    const found = this.byId.get(key);
    if (!found) return err({ type: "not_found", id: key });

    const current = found.entity;
    if (current.version !== expectedVersion) {
      return err({ type: "conflict", message: `version mismatch: current=${current.version}, expected=${expectedVersion}` });
    }

    const next = mutate(current);

    // Guard: prevent id changes (common foot-gun)
    const nextId = next.id as unknown as string;
    if (nextId !== key) {
      return err({ type: "conflict", message: "mutate must not change entity.id" });
    }

    const updated: UserProfile = { ...next, version: current.version + 1 };
    this.byId.set(key, { entity: updated });
    return ok(updated);
  }
}

// ---------- Service using update(...) ----------
export class UserProfileService {
  constructor(private repo: Repository<UserProfile, UserId, UserProfileQuery>) {}

  create(input: Omit<UserProfile, "version" | "createdAt">, nowMs: number): Result<UserProfile, ConflictError> {
    const entity: UserProfile = { ...input, version: 0, createdAt: nowMs };
    return this.repo.save(entity, 0);
  }

  updateName(id: UserId, expectedVersion: number, newName: string): Result<UserProfile, RepoError> {
    return this.repo.update(id, expectedVersion, (cur) => ({ ...cur, name: newName }));
  }

  addTag(id: UserId, expectedVersion: number, tag: Tag): Result<UserProfile, RepoError> {
    return this.repo.update(id, expectedVersion, (cur) => {
      const raw = tag as unknown as string;
      const exists = cur.tags.some((t) => (t as unknown as string) === raw);
      return exists ? cur : { ...cur, tags: [...cur.tags, tag] };
    });
  }
}

// ---------- Demo / checks ----------
const repo = new InMemoryUserProfileRepo();
const svc = new UserProfileService(repo);

const u = "usr_1a2b3c4d5e6f" as UserId;
const e = "max@example.com" as Email;
const tDev = "dev" as Tag;

const c1 = svc.create(
  { id: u, email: e, name: "Max", tags: [], marketingOptIn: true },
  Date.now()
);
console.log("create:", c1);

if (c1.ok) {
  const v1 = c1.value.version;

  const u1 = svc.updateName(u, v1, "Max Nguyen");
  console.log("updateName ok:", u1);

  // stale version should conflict
  const u2 = svc.updateName(u, v1, "Stale");
  console.log("updateName stale:", u2);

  // add tag using latest version if previous succeeded
  if (u1.ok) {
    const u3 = svc.addTag(u, u1.value.version, tDev);
    console.log("addTag:", u3);
  }
}
```

---

## What you should verify (Checklist)

* `updateName` success ⇒ version increments
* `updateName` with stale version ⇒ conflict
* `mutate` cố tình đổi `id` ⇒ conflict (guard works)
* Service code **không bump version** và **không gọi save trực tiếp** cho update nữa

---

## Stretch (nếu muốn đóng gói “siêu production”)

1. `update` nhận `mutate: (cur) => Result<next, DomainError>` để mutate có thể fail theo domain rule, vẫn không throw
2. `update` trả `{ entity, changed: boolean }` để tối ưu write khi no-op
3. Add `updateMany` trong “Unit of Work” style