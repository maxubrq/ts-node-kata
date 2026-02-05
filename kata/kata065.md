# Kata 65 — Input Sanitization: sanitize đúng chỗ, đúng ngữ cảnh

## Goal

1. Xây một module `sanitizers/` theo **contextual encoding**:

   * SQL: **parameterize** (không “escape string” thủ công)
   * HTML output: **escape**
   * URL query: **encodeURIComponent**
   * HTTP header value: **validate charset + strip CRLF**
   * File path: **normalize + allowlist + deny traversal**
   * Log: **strip control chars + redact patterns**
2. Viết một mini service có các endpoint “dễ dính”:

   * `GET /v1/search?q=...` (SQL-ish query)
   * `GET /v1/page?title=...` (HTML render)
   * `GET /v1/redirect?next=...` (open redirect)
   * `GET /v1/download?file=...` (path traversal)
   * `GET /v1/proxy?url=...` (SSRF *chỉ sanitize nhẹ ở kata này; full SSRF là kata 68*)
3. Tests: payload injection thực tế + đảm bảo không false-positive phá UX.

---

## Context (những “sink” thực sự)

Bạn không sanitize để “input sạch đẹp”. Bạn sanitize để **bảo vệ sink**:

* **DB query** (SQL injection / NoSQL injection)
* **HTML output** (XSS)
* **Redirect** (open redirect)
* **Filesystem** (path traversal)
* **Headers** (response splitting)
* **Logs** (log injection / forging)

---

## Constraints (ép đúng tư duy)

* ❌ Không được “sanitize input globally” kiểu `req.body = clean(req.body)` cho mọi thứ.
* ❌ Không được viết “SQL escape” tay.
* ✅ SQL query phải dùng **prepared/parameterized** (mô phỏng cũng được).
* ✅ HTML output phải escape theo HTML context.
* ✅ Redirect phải enforce allowlist (relative only hoặc allowlist host).
* ✅ File path phải chống `../` và symlink escape (stretch).
* ✅ Header phải chặn `\r\n`.
* ✅ Có test cho mỗi sink.

---

## Deliverables

Tạo `kata-65/`:

* `src/sanitizers/html.ts`
* `src/sanitizers/url.ts`
* `src/sanitizers/header.ts`
* `src/sanitizers/path.ts`
* `src/sanitizers/log.ts`
* `src/db/query.ts` (parameterized query mock)
* `src/server.ts`
* `test/kata65.sql.test.ts`
* `test/kata65.xss.test.ts`
* `test/kata65.redirect.test.ts`
* `test/kata65.path.test.ts`
* `test/kata65.header-log.test.ts`

Tooling gợi ý: `vitest`, `supertest` (nếu express), hoặc node http.

---

## Spec: Rule set (bạn phải implement)

### 1) SQL “sanitization” (đúng nghĩa là parameterization)

* `searchUsers(q)` phải gọi:

  * `db.query("SELECT ... WHERE name ILIKE $1", [`%${q}%`])`
* Không string concat vào query.

**Attack payload**: `q = "'; DROP TABLE users; --"`

* Kết quả: query string không đổi, params chứa payload.

### 2) HTML escape (XSS)

Endpoint render HTML (đơn giản):

* `/v1/page?title=...` trả HTML `<h1>${title}</h1>`

Bạn phải escape:

* `& < > " '`

**Attack payload**: `<img src=x onerror=alert(1)>`

* Response phải render như text, không thành tag.

### 3) Redirect guard (open redirect)

`/v1/redirect?next=...`

* Policy: **chỉ cho relative path** bắt đầu bằng `/`
* Reject:

  * `https://evil.com`
  * `//evil.com`
  * `\evil.com`
  * `/%2F%2Fevil.com` (encoded tricks)
  * `javascript:alert(1)`

Return 400 nếu invalid.

### 4) File path guard (path traversal)

`/v1/download?file=...`

* Base dir: `/srv/files`
* Policy:

  * chỉ cho filename dạng allowlist: `[a-zA-Z0-9._-]+`
  * không cho `/` hoặc `\`
  * resolve path và đảm bảo nó nằm trong base dir

Reject:

* `../secret.txt`
* `%2e%2e%2fsecret`
* `..\\secret`
* `a/../../b`

### 5) Header value guard (response splitting)

Một endpoint set header từ input (để test):

* `/v1/header?name=X-Note&value=...`
* Policy:

  * Header name allowlist: `X-Note` only (kata)
  * Header value: reject nếu chứa `\r` hoặc `\n` hoặc control chars (0x00–0x1F, 0x7F)

Attack:

* `value=ok%0d%0aSet-Cookie:%20pwn=1`

### 6) Log sanitization (log forging)

Bạn log `q` vào structured log.

* Policy:

  * strip `\r\n\t` hoặc encode chúng
  * limit length (e.g. 200 chars)
  * redact patterns (stretch): JWT-like `xxxxx.yyyyy.zzzzz`

Attack:

* `q="ok\n{\"level\":\"fatal\",\"msg\":\"pwn\"}"`
  Log output không được tạo dòng giả / field giả.

---

## Starter skeleton (điền TODO)

### `src/sanitizers/html.ts`

```ts
export function escapeHtml(input: string): string {
  // TODO: replace & < > " '
  return input;
}
```

### `src/sanitizers/url.ts`

```ts
export function validateRelativeRedirect(next: string): { ok: true; value: string } | { ok: false; error: string } {
  // TODO:
  // - trim
  // - reject empty
  // - reject any scheme (://) or startsWith("//")
  // - reject backslashes
  // - decode once (careful) and reject encoded // tricks
  // - allow only strings starting with "/" and not containing control chars
  return { ok: false, error: "TODO" };
}
```

### `src/sanitizers/path.ts`

```ts
import path from "node:path";

export function safeJoin(baseDir: string, file: string): { ok: true; fullPath: string } | { ok: false; error: string } {
  // TODO:
  // - allowlist filename chars only
  // - reject any slash/backslash
  // - resolve = path.resolve(baseDir, file)
  // - ensure resolved startsWith(path.resolve(baseDir) + path.sep)
  return { ok: false, error: "TODO" };
}
```

### `src/sanitizers/header.ts`

```ts
export function validateHeaderValue(value: string): { ok: true; value: string } | { ok: false; error: string } {
  // TODO: reject \r \n and CTLs
  return { ok: false, error: "TODO" };
}
```

### `src/sanitizers/log.ts`

```ts
export function sanitizeForLog(value: string, maxLen = 200): string {
  // TODO:
  // - replace \r \n \t with escaped sequences like "\\n"
  // - strip other control chars
  // - truncate to maxLen
  return value;
}
```

### `src/db/query.ts` (mock DB)

```ts
export type Db = {
  query: (sql: string, params: unknown[]) => Promise<{ sql: string; params: unknown[]; rows: any[] }>;
};

export function createMockDb(): Db {
  return {
    async query(sql, params) {
      return { sql, params, rows: [] };
    },
  };
}

export async function searchUsers(db: Db, q: string) {
  // TODO: MUST be parameterized
  // Example: "SELECT ... WHERE name ILIKE $1", [`%${q}%`]
  return db.query("TODO", []);
}
```

---

## Server wiring (minimal)

Trong `src/server.ts`:

* `GET /v1/search?q=...` → gọi `searchUsers(db,q)` và return `{sql, params}`
* `GET /v1/page?title=...` → return HTML với `escapeHtml`
* `GET /v1/redirect?next=...` → validateRelativeRedirect, nếu ok set `Location` & 302
* `GET /v1/download?file=...` → safeJoin, nếu ok return `{path}`
* `GET /v1/header?value=...` → validateHeaderValue, set `X-Note`
* Log mọi request q/title/next/file bằng `sanitizeForLog`

---

## Tests — bắt buộc

### `test/kata65.sql.test.ts`

* payload `q="'; DROP TABLE users; --"`
* assert:

  * response.sql không chứa payload (không concat)
  * response.params[0] chứa payload

### `test/kata65.xss.test.ts`

* title `<img src=x onerror=alert(1)>`
* assert response body chứa `&lt;img` và không có `<img`

### `test/kata65.redirect.test.ts`

* allow: `/home`, `/orders/123?x=1`
* deny: `https://evil.com`, `//evil.com`, `/\evil`, `/%2F%2Fevil.com`, `javascript:alert(1)`

### `test/kata65.path.test.ts`

* allow: `report.pdf`, `a_b-1.txt`
* deny: `../secret`, `%2e%2e%2fsecret`, `a/../../b`, `..\\x`

### `test/kata65.header-log.test.ts`

* header value with CRLF injection must be rejected
* log sanitize:

  * input contains `\n` → output contains literal `\\n` not newline

---

## Checks (Definition of Done)

Bạn pass kata nếu:

1. SQL dùng parameterization, không concat.
2. HTML output escape đúng 5 ký tự.
3. Redirect chỉ allow relative, chặn encoded tricks.
4. Path traversal bị chặn cả raw và encoded.
5. Header splitting bị chặn (`\r\n` + CTLs).
6. Logs không bị forged (no multiline injection), có length cap.
7. Tests cover all sinks + attacks.

---

## Stretch (Senior+ thực chiến)

1. **Contextual sanitization matrix**
   Viết `docs/sinks.md`: input → sink → mitigation (encode vs validate vs parameterize).
2. **NoSQL injection drill**
   Mô phỏng Mongo query: reject object operators (`$ne`, `$gt`) nếu endpoint chỉ nhận string.
3. **Unicode confusables**
   Normalize (NFKC) cho header/path allowlist để chặn bypass bằng ký tự giống nhau.
4. **Symlink escape defense** (FS)
   Sau resolve, dùng `fs.realpath` để đảm bảo không thoát baseDir qua symlink.
5. **Security regression harness**
   Tạo một file payload list và chạy paramized tests (giống mini-fuzz).