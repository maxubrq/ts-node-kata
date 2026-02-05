# Kata 68 — SSRF Prevention: allowlist + block internal ranges

## Goal

1. Implement `safeFetch(url, options)` chỉ cho phép gọi ra ngoài theo policy:

   * Allowlist hostname (exact hoặc suffix rule có kiểm soát)
   * Resolve DNS và **chặn private/internal IP ranges**
   * Chặn **localhost / loopback / link-local / multicast / reserved**
   * Chặn **userinfo** (`user:pass@host`)
   * Chặn schemes lạ (`file:`, `gopher:`, `ftp:`…)
   * Port policy (chỉ 80/443 hoặc allowlist port)
   * Redirect policy (không follow hoặc follow nhưng re-validate từng hop)
2. Chống **DNS rebinding**:

   * Re-resolve hostname (hoặc pin IP) trước khi connect / mỗi redirect hop
   * Reject nếu DNS trả về IP “bad”
3. Có **timeout + abort** end-to-end.
4. Tests cover các bypass kinh điển.

---

## Context

Bạn có endpoint:

* `GET /v1/proxy?url=...` dùng để fetch metadata từ một số domain đối tác.

Đây là SSRF sink điển hình: attacker đưa `http://169.254.169.254/latest/meta-data` hoặc `http://localhost:2375`…

---

## Constraints

* ❌ Không được “block by substring” kiểu `if (url.includes("169.254"))`.
* ❌ Không được chỉ check hostname string mà không resolve DNS.
* ✅ Must parse URL bằng `new URL()`.
* ✅ Must enforce allowlist trước (host), và enforce **IP range block** sau khi resolve.
* ✅ Must enforce redirect policy (default: **NO redirects**).
* ✅ Must return structured deny reasons + HTTP mapping.
* ✅ Không log full URL query nếu có thể chứa secrets (log safe parts).

---

## Deliverables

Tạo `kata-68/`:

* `src/ssrf/policy.ts`
* `src/ssrf/ip.ts`
* `src/ssrf/resolve.ts`
* other
* `src/ssrf/guard.ts`
* `src/ssrf/safeFetch.ts` (wrapper fetch)
* `src/server.ts` (proxy endpoint)
* `test/kata68.url-parse.test.ts`
* `test/kata68.ip-range.test.ts`
* `test/kata68.redirect.test.ts`
* `test/kata68.dns-rebinding.test.ts`

Tooling: `vitest`, Node 20+ (`fetch` built-in). DNS resolution có thể mock trong tests.

---

## Spec: SSRF Policy

### Allowed schemes

* `https:` (recommended)
* `http:` (optional, but if allowed still enforce IP blocks)

### Allowed hosts (choose one strategy)

* **Exact allowlist**: `api.github.com`, `example-partner.com`
* **Suffix allowlist**: `*.partner.example` but **must avoid** `evilpartner.example.com` pitfalls by enforcing dot boundary.

### Port policy

* Allow ports: `443` (and maybe `80`)
* Reject others by default.

### Redirect policy

* Default: `maxRedirects = 0` (no follow)
* Stretch: allow up to 3 redirects but **must re-validate** each hop URL + re-resolve + re-check IP.

### Blocked IP ranges (minimum)

* Loopback: `127.0.0.0/8`, `::1/128`
* Private:

  * `10.0.0.0/8`
  * `172.16.0.0/12`
  * `192.168.0.0/16`
* Link-local / metadata:

  * `169.254.0.0/16` (includes cloud metadata)
  * `fe80::/10` (IPv6 link-local)
* Multicast/reserved (at least):

  * `224.0.0.0/4`
  * `0.0.0.0/8`
  * `100.64.0.0/10` (CGNAT) *(khuyến nghị block)*
* IPv6 ULA: `fc00::/7`
* (Stretch) deny “IPv4-mapped IPv6” patterns

---

## Error mapping (proxy endpoint)

* invalid url / scheme / parse fail → `400` reason `INVALID_URL`
* host not allowed → `403` reason `HOST_NOT_ALLOWED`
* resolved IP is blocked → `403` reason `IP_BLOCKED`
* port not allowed → `403` reason `PORT_NOT_ALLOWED`
* redirect blocked → `403` reason `REDIRECT_NOT_ALLOWED`
* DNS fail → `502` reason `DNS_RESOLUTION_FAILED`
* timeout → `504` reason `UPSTREAM_TIMEOUT`

---

## Starter skeleton (điền TODO)

### `src/ssrf/policy.ts`

```ts
export type SsrfPolicy = {
  allowedSchemes: readonly ("https:" | "http:")[];
  allowedHosts: readonly string[];       // exact or suffix entries
  allowSubdomainsOf?: readonly string[]; // e.g. ["partner.example"]
  allowedPorts: readonly number[];       // e.g. [443, 80]
  maxRedirects: number;                  // default 0
  timeoutMs: number;                     // e.g. 1500
};

export function isHostAllowed(hostname: string, policy: SsrfPolicy): boolean {
  const h = hostname.toLowerCase();

  // exact allowlist
  if (policy.allowedHosts.map(x => x.toLowerCase()).includes(h)) return true;

  // suffix allowlist with dot-boundary
  if (policy.allowSubdomainsOf) {
    for (const suffix of policy.allowSubdomainsOf) {
      const s = suffix.toLowerCase();
      if (h === s) return true;
      if (h.endsWith("." + s)) return true;
    }
  }
  return false;
}
```

### `src/ssrf/ip.ts`

```ts
export type IpFamily = 4 | 6;

export function isIpLiteral(hostname: string): boolean {
  // TODO: detect IPv4 literal, IPv6 literal (bracketed handled by URL)
  return false;
}

export function parseIp(host: string): { ok: true; family: IpFamily; bytes: Uint8Array } | { ok: false } {
  // TODO: parse IPv4/IPv6 into bytes. (Bạn có thể dùng `net.isIP` + custom parse)
  return { ok: false };
}

export function isBlockedIp(ip: { family: IpFamily; bytes: Uint8Array }): boolean {
  // TODO: implement CIDR checks for the blocked ranges in spec
  return false;
}
```

### `src/ssrf/resolve.ts`

```ts
import dns from "node:dns/promises";
import type { IpFamily } from "./ip";

export type ResolvedAddress = { address: string; family: IpFamily };

export async function resolveHost(hostname: string): Promise<{ ok: true; addrs: ResolvedAddress[] } | { ok: false; reason: "DNS_RESOLUTION_FAILED" }> {
  try {
    // TODO: use dns.lookup(hostname, { all: true, verbatim: true })
    const out = await dns.lookup(hostname, { all: true, verbatim: true });
    const addrs = out.map(x => ({ address: x.address, family: (x.family === 6 ? 6 : 4) as IpFamily }));
    return { ok: true, addrs };
  } catch {
    return { ok: false, reason: "DNS_RESOLUTION_FAILED" };
  }
}
```

### `src/ssrf/guard.ts`

```ts
import { isHostAllowed, type SsrfPolicy } from "./policy";
import { parseIp, isBlockedIp } from "./ip";
import { resolveHost } from "./resolve";

export type GuardFailReason =
  | "INVALID_URL"
  | "SCHEME_NOT_ALLOWED"
  | "HOST_NOT_ALLOWED"
  | "USERINFO_NOT_ALLOWED"
  | "PORT_NOT_ALLOWED"
  | "DNS_RESOLUTION_FAILED"
  | "IP_BLOCKED"
  | "REDIRECT_NOT_ALLOWED";

export type GuardResult =
  | { ok: true; url: URL; resolvedIps: string[] }
  | { ok: false; reason: GuardFailReason; status: number; detail: string };

export async function guardUrl(raw: string, policy: SsrfPolicy): Promise<GuardResult> {
  let url: URL;
  try {
    url = new URL(raw);
  } catch {
    return { ok: false, reason: "INVALID_URL", status: 400, detail: "Malformed URL" };
  }

  // scheme
  if (!policy.allowedSchemes.includes(url.protocol as any)) {
    return { ok: false, reason: "SCHEME_NOT_ALLOWED", status: 400, detail: "Scheme not allowed" };
  }

  // userinfo (username:password@)
  if (url.username || url.password) {
    return { ok: false, reason: "USERINFO_NOT_ALLOWED", status: 400, detail: "Userinfo not allowed" };
  }

  // host allowlist
  const hostname = url.hostname; // normalized (no brackets)
  if (!isHostAllowed(hostname, policy)) {
    return { ok: false, reason: "HOST_NOT_ALLOWED", status: 403, detail: "Host not allowed" };
  }

  // port policy
  const port = url.port ? Number(url.port) : (url.protocol === "https:" ? 443 : 80);
  if (!policy.allowedPorts.includes(port)) {
    return { ok: false, reason: "PORT_NOT_ALLOWED", status: 403, detail: "Port not allowed" };
  }

  // DNS resolve + IP block
  const resolved = await resolveHost(hostname);
  if (!resolved.ok) {
    return { ok: false, reason: "DNS_RESOLUTION_FAILED", status: 502, detail: "DNS resolution failed" };
  }

  const ips: string[] = [];
  for (const a of resolved.addrs) {
    const parsed = parseIp(a.address);
    if (!parsed.ok) continue; // should not happen, but safe
    if (isBlockedIp({ family: parsed.family, bytes: parsed.bytes })) {
      return { ok: false, reason: "IP_BLOCKED", status: 403, detail: "Resolved IP is blocked" };
    }
    ips.push(a.address);
  }

  // IMPORTANT: if DNS returns both v4/v6, block if any is blocked (fail-closed)
  return { ok: true, url, resolvedIps: ips };
}
```

### `src/ssrf/safeFetch.ts`

```ts
import { guardUrl } from "./guard";
import type { SsrfPolicy } from "./policy";

export type SafeFetchResult =
  | { ok: true; status: number; body: string }
  | { ok: false; status: number; reason: string };

export async function safeFetch(rawUrl: string, policy: SsrfPolicy, init?: RequestInit): Promise<SafeFetchResult> {
  const g = await guardUrl(rawUrl, policy);
  if (!g.ok) return { ok: false, status: g.status, reason: g.reason };

  const ac = new AbortController();
  const t = setTimeout(() => ac.abort(), policy.timeoutMs);

  try {
    // Default: do not follow redirects automatically
    const res = await fetch(g.url, { ...init, redirect: "manual", signal: ac.signal });

    // Redirect handling
    if (res.status >= 300 && res.status < 400) {
      if (policy.maxRedirects <= 0) {
        return { ok: false, status: 403, reason: "REDIRECT_NOT_ALLOWED" };
      }
      // Stretch: implement manual redirect loop:
      // - read Location
      // - resolve relative vs absolute
      // - re-run guardUrl for each hop
      // - decrement redirect budget
      return { ok: false, status: 403, reason: "REDIRECT_NOT_ALLOWED" };
    }

    const body = await res.text();
    return { ok: true, status: res.status, body };
  } catch (e: any) {
    if (e?.name === "AbortError") return { ok: false, status: 504, reason: "UPSTREAM_TIMEOUT" };
    return { ok: false, status: 502, reason: "UPSTREAM_FETCH_FAILED" };
  } finally {
    clearTimeout(t);
  }
}
```

---

## Server wiring (minimal)

`GET /v1/proxy?url=`:

* call `safeFetch(url, policy)`
* if ok: return `{status, bodyPreview}` (truncate body to 2KB)
* if fail: return `{error: reason}` với status tương ứng
* Log: `reason`, `hostname`, `port`, `resolvedIpsCount` (không log full query string)

---

## Tests — bắt buộc (table-driven)

### 1) URL parsing & allowlist (`test/kata68.url-parse.test.ts`)

* deny:

  * `file:///etc/passwd` → `SCHEME_NOT_ALLOWED`
  * `http://user:pass@api.github.com/` → `USERINFO_NOT_ALLOWED`
  * `https://evil.com/` → `HOST_NOT_ALLOWED`
* allow:

  * `https://api.github.com/repos/...` (nếu nằm allowlist)

### 2) Internal ranges (`test/kata68.ip-range.test.ts`)

Mock `resolveHost()` để trả IP cụ thể:

* host allowed nhưng resolve ra `127.0.0.1` → `IP_BLOCKED`
* `169.254.169.254` → `IP_BLOCKED`
* `10.1.2.3` → `IP_BLOCKED`
* public IP (vd `93.184.216.34`) → allow

### 3) Redirect handling (`test/kata68.redirect.test.ts`)

Mock fetch trả 302 Location:

* `maxRedirects=0` → `REDIRECT_NOT_ALLOWED`
* (Stretch) nếu bạn implement redirects: mỗi hop phải re-guard.

### 4) DNS rebinding (`test/kata68.dns-rebinding.test.ts`)

Mô phỏng hostname allowlisted `partner.example`:

* lần resolve 1 trả public IP
* lần resolve 2 (cho hop redirect hoặc re-check) trả `127.0.0.1`
* Expect fail-closed `IP_BLOCKED`

> Gợi ý testing: inject `resolveHost` vào `guardUrl` qua dependency injection thay vì import trực tiếp (Senior+ design).

---

## Checks (Definition of Done)

Bạn pass kata nếu:

1. Parse URL đúng, chặn scheme lạ, chặn userinfo.
2. Enforce host allowlist (exact/suffix có dot-boundary).
3. Resolve DNS và chặn toàn bộ internal ranges (fail-closed nếu có IP xấu).
4. Không follow redirects (hoặc follow có re-validation).
5. Timeout/Abort hoạt động.
6. Tests cover bypass chính.

---

## Stretch (Senior+ thật sự)

1. **Manual redirect loop (max 3)**

   * `Location` có thể relative → resolve bằng `new URL(location, currentUrl)`
   * Re-run guard cho từng hop
   * Re-resolve DNS mỗi hop
2. **IP pinning**

   * Resolve DNS → chọn 1 IP public → connect chỉ tới IP đó (tránh rebinding giữa resolve và connect).
     *Trong Node fetch mặc định khó pin, nhưng bạn có thể mô phỏng bằng custom agent (hard).*
3. **Block “alternative IP notations”**

   * IPv4 dạng decimal/hex/octal (`2130706433` = 127.0.0.1)
   * IPv6 compressed / IPv4-mapped IPv6
4. **Egress allowlist theo path**

   * host ok nhưng chỉ allow một số path prefix (đối tác API).
5. **Observability**

   * counter `ssrf_block_total{reason=...}`
   * log `hostname`, `port`, `policyId`, `decision` (không log query)
