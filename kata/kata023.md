# Kata 23 — ETag & Caching

**Implement Conditional GET**: `ETag`, `If-None-Match`, `304 Not Modified`, cache correctness (strong ETag).

## Goal

Bạn build endpoint giả lập:

`GET /users/:id`

Yêu cầu:

1. Server trả `ETag` cho resource
2. Client gửi `If-None-Match`:

* nếu match → trả **304** (không body)
* nếu không match → trả **200 + body**

3. ETag phải đổi khi resource đổi (update)
4. Có `Cache-Control` hợp lý (tối thiểu `private, max-age=0`)

---

## Constraints

* ❌ Không dùng `Last-Modified` thay ETag (có thể thêm stretch)
* ✅ Strong ETag (quoted string, ví dụ `"abc123"`)
* ✅ Không `any`
* ✅ ETag compute deterministic (không random)
* ✅ Handle multiple ETags trong header (comma-separated) + wildcard `*`

---

## Deliverables

File `kata23.ts` gồm:

1. In-memory store `users`
2. `computeEtag(user)` (strong)
3. `handleGetUser(req)` trả `{status, headers, body?}`
4. Tests:

* first GET returns 200 + ETag
* GET with If-None-Match same ETag returns 304
* update user then conditional GET returns 200 with new ETag
* If-None-Match: `*` returns 304 only when resource exists
* If-None-Match with list `"etag1", "etag2"` works

---

## Spec

### Domain

```ts
type User = {
  id: string;
  name: string;
  email: string;
  version: number;  // increments on update
};
```

### HTTP shapes

```ts
type HttpRequest = {
  path: string;                 // e.g. "/users/u1"
  headers: Record<string, string | undefined>;
};

type HttpResponse = {
  status: number;
  headers: Record<string, string>;
  body?: unknown;
};
```

### Rules

* If user not found → 404 (no ETag)
* If found:

  * compute etag
  * if request has If-None-Match and it matches etag (or `*`) → 304
  * else 200 with body + ETag
* 304 must not include body

---

## Starter skeleton (điền TODO)

```ts
/* kata23.ts */
import crypto from "node:crypto";

// ---------- Domain ----------
type User = { id: string; name: string; email: string; version: number };

// ---------- HTTP ----------
type HttpRequest = {
  path: string;
  headers: Record<string, string | undefined>;
};

type HttpResponse = {
  status: number;
  headers: Record<string, string>;
  body?: unknown;
};

// ---------- Store ----------
const users = new Map<string, User>();

function seed() {
  users.clear();
  users.set("u1", { id: "u1", name: "Max", email: "max@example.com", version: 1 });
}

function updateUser(id: string, patch: Partial<Pick<User, "name" | "email">>) {
  const u = users.get(id);
  if (!u) return;
  users.set(id, { ...u, ...patch, version: u.version + 1 });
}

// ---------- ETag (strong) ----------
function computeEtag(u: User): string {
  // TODO: deterministic stable string -> hash -> quoted
  // Usually include version or canonical JSON of fields
  const canonical = `${u.id}|${u.version}|${u.name}|${u.email}`;
  const h = crypto.createHash("sha256").update(canonical).digest("hex").slice(0, 16);
  return `"${h}"`;
}

// ---------- Parse If-None-Match ----------
function parseIfNoneMatch(v: string): string[] | "*" {
  // TODO:
  // - trim
  // - if exactly "*" return "*"
  // - split by comma, trim each, keep only quoted tokens like "abc"
  // - ignore weak ETags W/"..."
  return [];
}

function matchesIfNoneMatch(ifNoneMatch: string[] | "*", etag: string): boolean {
  if (ifNoneMatch === "*") return true;
  return ifNoneMatch.includes(etag);
}

// ---------- Handler ----------
export function handleGetUser(req: HttpRequest): HttpResponse {
  // parse id from path "/users/:id"
  const parts = req.path.split("/").filter(Boolean);
  const id = parts.length === 2 && parts[0] === "users" ? parts[1] : "";

  if (!id) {
    return { status: 400, headers: { "content-type": "application/json" }, body: { error: "bad_path" } };
  }

  const u = users.get(id);
  if (!u) {
    return { status: 404, headers: { "content-type": "application/json" }, body: { error: "not_found" } };
  }

  const etag = computeEtag(u);
  const inm = req.headers["if-none-match"];

  if (typeof inm === "string") {
    const parsed = parseIfNoneMatch(inm);
    if (matchesIfNoneMatch(parsed, etag)) {
      // 304: no body
      return {
        status: 304,
        headers: {
          etag,
          "cache-control": "private, max-age=0",
        },
      };
    }
  }

  return {
    status: 200,
    headers: {
      etag,
      "cache-control": "private, max-age=0",
      "content-type": "application/json",
    },
    body: { id: u.id, name: u.name, email: u.email, version: u.version },
  };
}

// ---------- Tests ----------
function assert(cond: unknown, msg: string) {
  if (!cond) throw new Error("ASSERT FAIL: " + msg);
}

function testBasic() {
  seed();
  const r1 = handleGetUser({ path: "/users/u1", headers: {} });
  assert(r1.status === 200, "first GET 200");
  const etag = r1.headers.etag;
  assert(typeof etag === "string" && etag.startsWith(`"`), "etag present and quoted");

  const r2 = handleGetUser({ path: "/users/u1", headers: { "if-none-match": etag } });
  assert(r2.status === 304, "conditional GET returns 304");
  assert(!("body" in r2), "304 has no body");
}

function testUpdateChangesEtag() {
  seed();
  const r1 = handleGetUser({ path: "/users/u1", headers: {} });
  const e1 = r1.headers.etag;

  updateUser("u1", { name: "Max 2" });
  const r2 = handleGetUser({ path: "/users/u1", headers: { "if-none-match": e1 } });
  assert(r2.status === 200, "etag changed => 200");
  const e2 = r2.headers.etag;
  assert(e2 !== e1, "etag differs after update");
}

function testWildcard() {
  seed();
  const r = handleGetUser({ path: "/users/u1", headers: { "if-none-match": "*" } });
  assert(r.status === 304, "if-none-match * => 304 when exists");

  const r404 = handleGetUser({ path: "/users/u404", headers: { "if-none-match": "*" } });
  assert(r404.status === 404, "still 404 when not exists");
}

function testListHeader() {
  seed();
  const r1 = handleGetUser({ path: "/users/u1", headers: {} });
  const etag = r1.headers.etag;

  const r2 = handleGetUser({
    path: "/users/u1",
    headers: { "if-none-match": `"nope", ${etag}, "nope2"` },
  });
  assert(r2.status === 304, "list contains etag => 304");
}

function run() {
  testBasic();
  testUpdateChangesEtag();
  testWildcard();
  testListHeader();
  console.log("ALL TESTS PASSED");
}

run();
```

---

## Tasks bạn phải làm

1. Implement `parseIfNoneMatch()` đúng:

* handle whitespace
* handle `*`
* handle comma-separated list
* ignore weak ETags `W/"..."` (stretch: support them)

2. Ensure 304 response:

* có `etag` + `cache-control`
* **không có body**

3. Optional: ensure `content-type` absent in 304 (không bắt buộc, nhưng cleaner)
4. Add one test: If-None-Match with weak etag should not match strong etag (hoặc bạn hỗ trợ weak match theo RFC, nhưng phải consistent)

---

## Definition of Done

* Tests pass
* ETag deterministic, changes on update
* Conditional GET returns 304 correctly
* Header parsing robust

---

## Stretch (Senior+)

1. **ETag signing**: include HMAC to prevent forging (rarely needed for GET, but useful in multi-tenant caches)
2. Add `If-Match` for conditional **PUT** (optimistic concurrency)
3. Add `Vary` header when response depends on auth/locale
4. Integrate with `Cache-Control: public, max-age=...` for truly cacheable resources (careful with user-specific data)