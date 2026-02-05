# Kata 62 — AuthN/AuthZ: RBAC + ABAC “nhỏ nhưng chuẩn”

## Goal

1. **AuthN (Authentication)**: xác thực người dùng từ `Authorization: Bearer <token>` (token mock/JWT-like nhưng bạn không cần crypto phức tạp ở kata này).
2. **AuthZ (Authorization)**: quyết định quyền truy cập bằng:

   * **RBAC**: role → permissions
   * **ABAC**: attributes (tenant, resource owner, plan, time, risk…) → constraints
3. **Deny by default**: thiếu thông tin / không match policy → **DENY**.
4. **Audit logging**: log “decision” **không lộ PII nhạy cảm** (hash userId/subject), có reason code.
5. **Test**: unit + integration-ish (HTTP handler) + policy tests.

---

## Context

Bạn có API nhỏ:

* `GET /v1/tenants/:tenantId/orders/:orderId`
* `POST /v1/tenants/:tenantId/orders`
* `POST /v1/tenants/:tenantId/orders/:orderId/refund`

Multi-tenant là bắt buộc.

Bạn có 3 role:

* `admin`
* `support`
* `member`

Và ABAC rules:

* Member chỉ đọc/ghi **trong tenant của mình**.
* Member chỉ refund nếu **owner** của order **và** `plan=pro`.
* Support có thể read tất cả trong tenant mình được assign, nhưng refund chỉ khi `riskScore < 50`.
* Admin có full access (nhưng vẫn phải đúng tenant boundary nếu bạn chọn policy strict).

---

## Constraints

* ❌ Không “if role === admin return true” rải rác ở handler.
* ❌ Không hardcode policy trong route handler.
* ✅ Policy tách riêng: `authorize(subject, action, resource, ctx)` trả về `Decision`.
* ✅ `Decision` phải có `reason` (machine-readable) để audit/metrics.
* ✅ Có middleware `requireAuth` + `requirePermission`.
* ✅ Không log raw token, không log email/PII trực tiếp.
* ✅ Có test:

  * policy table tests (RBAC + ABAC)
  * middleware tests (401/403)
  * audit log redaction test

---

## Deliverables

Tạo `kata-62/`:

* `src/auth/types.ts`
* `src/auth/authn.ts` (parse bearer token → Subject)
* `src/auth/rbac.ts` (roles → permissions)
* `src/auth/abac.ts` (attribute checks)
* `src/auth/policy.ts` (authorize core)
* `src/auth/middleware.ts`
* `src/logger.ts` (redaction)
* `src/server.ts` (minimal http app, có thể dùng express/fastify hoặc node http)
* `test/kata62.policy.test.ts`
* `test/kata62.http.test.ts`

---

## Domain model (đủ để làm)

### Actions

* `orders:read`
* `orders:create`
* `orders:refund`

### Subject (authenticated principal)

* `sub: UserId` (branded type optional)
* `roles: Role[]`
* `tenantId: string` (tenant “current”)
* `assignedTenants?: string[]` (cho support)
* `plan: "free" | "pro"`
* `riskScore: number` (0–100)

### Resource

* Order:

  * `tenantId`
  * `orderId`
  * `ownerUserId`
  * `amount`
  * `status: "paid" | "refunded"`

---

## Policy rules (Spec)

### RBAC baseline

* `admin` → allow: read, create, refund
* `support` → allow: read
* `member` → allow: read, create

*(refund không nằm trong support/member baseline; refund phải qua ABAC grant)*

### ABAC constraints

* **Tenant isolation**: `subject.tenantId === resource.tenantId` là điều kiện bắt buộc (trừ khi bạn implement cross-tenant admin; mặc định **không**).
* **member refund**: allow nếu:

  * `action=orders:refund`
  * `subject.plan === "pro"`
  * `resource.ownerUserId === subject.sub`
  * `resource.status === "paid"`
* **support read**: allow nếu:

  * `action=orders:read`
  * `resource.tenantId` nằm trong `subject.assignedTenants`
* **support refund**: allow nếu:

  * `action=orders:refund`
  * `subject.riskScore < 50`
  * `resource.status === "paid"`

### Precedence

* Deny-by-default
* Any hard failure (missing subject/resource attr) → deny với reason phù hợp
* Tenant mismatch → deny luôn (reason: `TENANT_MISMATCH`)

---

## Starter skeleton (điền TODO)

### `src/auth/types.ts`

```ts
export type Role = "admin" | "support" | "member";

export type Action = "orders:read" | "orders:create" | "orders:refund";

export type Subject = {
  sub: string; // keep string for kata; optionally brand
  roles: Role[];
  tenantId: string;
  assignedTenants?: string[];
  plan: "free" | "pro";
  riskScore: number; // 0..100
};

export type Order = {
  tenantId: string;
  orderId: string;
  ownerUserId: string;
  amount: number;
  status: "paid" | "refunded";
};

export type Resource =
  | { kind: "order"; order: Order }
  | { kind: "tenant"; tenantId: string };

export type Decision =
  | { ok: true; reason: "ALLOW"; via: "RBAC" | "ABAC" | "RBAC+ABAC" }
  | {
      ok: false;
      reason:
        | "UNAUTHENTICATED"
        | "FORBIDDEN"
        | "TENANT_MISMATCH"
        | "MISSING_ATTR"
        | "NOT_IN_ASSIGNED_TENANTS"
        | "NOT_OWNER"
        | "PLAN_REQUIRED"
        | "RISK_TOO_HIGH"
        | "ORDER_NOT_REFUNDABLE";
    };
```

### `src/auth/authn.ts`

```ts
import type { Subject } from "./types";

/**
 * Token format (kata):
 *  base64(json)
 * Example payload:
 * { "sub":"u1", "roles":["member"], "tenantId":"t1", "plan":"pro", "riskScore":10 }
 */
export function authenticate(authorizationHeader: string | undefined): Subject | null {
  // TODO:
  // - require "Bearer <token>"
  // - base64 decode (Buffer.from(token, "base64"))
  // - JSON.parse
  // - validate minimal fields exist (no need for full schema, but better if you do)
  // - return Subject
  return null;
}
```

### `src/auth/rbac.ts`

```ts
import type { Action, Role } from "./types";

const ROLE_PERMS: Record<Role, readonly Action[]> = {
  admin: ["orders:read", "orders:create", "orders:refund"],
  support: ["orders:read"],
  member: ["orders:read", "orders:create"],
};

export function rbacAllows(roles: Role[], action: Action): boolean {
  // TODO: if any role grants action
  return false;
}
```

### `src/auth/abac.ts`

```ts
import type { Action, Decision, Resource, Subject } from "./types";

export function abacAllows(subject: Subject, action: Action, resource: Resource): Decision | null {
  // Return:
  // - {ok:true, reason:"ALLOW", via:"ABAC"} if ABAC grants (even if RBAC doesn't)
  // - {ok:false, reason:...} if ABAC explicitly denies with a specific reason
  // - null if ABAC is not applicable (let RBAC decide)
  //
  // TODO implement rules:
  // - tenant isolation mandatory
  // - member refund constraints
  // - support assigned tenants read
  // - support refund with risk
  return null;
}
```

### `src/auth/policy.ts`

```ts
import type { Action, Decision, Resource, Subject } from "./types";
import { rbacAllows } from "./rbac";
import { abacAllows } from "./abac";

export function authorize(
  subject: Subject | null,
  action: Action,
  resource: Resource,
): Decision {
  // TODO:
  // - if no subject => UNAUTHENTICATED
  // - ABAC first for hard denies (e.g., TENANT_MISMATCH, missing attrs)
  // - if ABAC grants => ALLOW (via ABAC or RBAC+ABAC)
  // - else if RBAC allows => ALLOW (via RBAC) BUT still must respect tenant isolation (ABAC hard rule)
  // - else => FORBIDDEN
  return { ok: false, reason: "FORBIDDEN" };
}
```

### `src/auth/middleware.ts`

```ts
import type { Action, Resource, Subject } from "./types";
import { authenticate } from "./authn";
import { authorize } from "./policy";
import { logger } from "../logger";

export type Req = { headers: Record<string, string | undefined>; params: any; body?: any };
export type Res = { statusCode: number; body: any; end: (b: any) => void };
export type Next = () => Promise<void> | void;

export function requireAuth(req: Req): Subject | null {
  const sub = authenticate(req.headers["authorization"]);
  return sub;
}

export function requirePermission(action: Action, getResource: (req: Req) => Resource) {
  return async (req: Req, res: Res, next: Next) => {
    const subject = requireAuth(req);
    const resource = getResource(req);

    const decision = authorize(subject, action, resource);

    // TODO: audit log (no raw token, no PII)
    // log fields: action, resource.kind, tenantId, decision.ok, decision.reason, subject hash (optional)
    logger.info({ action, resource: resource.kind, ok: decision.ok, reason: decision.ok ? decision.reason : decision.reason }, "authz_decision");

    if (!decision.ok) {
      res.statusCode = decision.reason === "UNAUTHENTICATED" ? 401 : 403;
      res.end({ error: decision.reason });
      return;
    }

    await next();
  };
}
```

---

## HTTP wiring (minimal)

Trong `src/server.ts`, dựng 3 route, mỗi route dùng `requirePermission`:

* `GET /v1/tenants/:tenantId/orders/:orderId` → action `orders:read`, resource `order` (mock load order từ in-memory map)
* `POST /v1/tenants/:tenantId/orders` → action `orders:create`, resource `tenant`
* `POST /v1/tenants/:tenantId/orders/:orderId/refund` → action `orders:refund`, resource `order`

**Quan trọng**: resource phải có `tenantId` để tenant isolation check hoạt động.

---

## Tests bạn phải có

### 1) Policy table tests (quan trọng nhất)

Tạo bảng cases:

* member/pro đúng tenant, owner, status=paid → refund allow
* member/free → refund deny `PLAN_REQUIRED`
* member/pro nhưng không owner → `NOT_OWNER`
* member/pro owner nhưng status=refunded → `ORDER_NOT_REFUNDABLE`
* support read order trong assignedTenants → allow
* support read order không trong assignedTenants → deny `NOT_IN_ASSIGNED_TENANTS`
* support refund riskScore=70 → deny `RISK_TOO_HIGH`
* tenant mismatch bất kể role → deny `TENANT_MISMATCH`
* unauthenticated → `UNAUTHENTICATED`

### 2) Middleware/HTTP tests

* không có Authorization → 401
* có token nhưng forbidden → 403 + reason đúng
* allow case → 200

### 3) Audit log safety test

* đảm bảo log output **không chứa** token và không chứa `sub` raw (nếu bạn chọn hash).

---

## Checks (Definition of Done)

Bạn pass kata nếu:

1. **Tất cả quyết định quyền** nằm trong `authorize()` + policy modules, handler không có `if role`.
2. **Tenant isolation** enforced: mismatch luôn 403 với `TENANT_MISMATCH`.
3. RBAC và ABAC phối hợp đúng theo spec.
4. Audit log có reason codes, không leak token/PII.
5. Test cover đầy đủ các case quan trọng.

---

## Stretch (Senior+ “security-grade”)

1. **Policy-as-code**: viết DSL nhỏ:

   * `rule("tenant isolation").when(...).deny("TENANT_MISMATCH")`
   * dễ đọc, dễ review, dễ diff khi thay policy.
2. **Reason → Metrics**: đếm `authz_deny_total{reason=...}` để xem deny patterns.
3. **Contextual constraints**: thêm `ipReputation`, `mfaLevel`, `timeWindow` vào ABAC.
4. **Permission drift test**: snapshot file JSON mapping role→perms, CI fail nếu thay đổi mà không update “security review note”.
5. **Least privilege by route**: mỗi route declare required action + resource shape, viết test đảm bảo route không “forgot auth middleware”.