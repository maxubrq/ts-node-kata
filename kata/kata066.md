# Kata 66 — Dependency Risk: audit + lockfile strategy

## Goal

1. Thiết lập **chính sách phụ thuộc** (dependency policy) cho một repo Node/TS:

   * direct vs transitive
   * runtime vs dev
   * allowed registries
   * update cadence
2. Thiết kế **lockfile strategy**:

   * khi nào commit lockfile
   * pinning rules (`^`/`~`/exact)
   * reproducible installs trong CI
3. Thiết lập **security gates** trong CI:

   * vulnerability scan (severity threshold + exceptions)
   * license allowlist/denylist
   * detect malicious patterns (typosquatting, new maintainer, sudden version jump)
4. Viết một **incident drill**: nhận alert CVE → triage → patch → verify → postmortem note.

---

## Context

Bạn có repo `orders-api` (TypeScript, Node 20+), dùng:

* `express`/`fastify`
* `zod`
* `pino`
* `vitest`

Repo sẽ deploy production, nên yêu cầu:

* **reproducible builds**
* **minimal dependency surface**
* **fast response** khi có vulnerability

---

## Constraints

* ✅ Output phải là **artifact** (docs + scripts + CI config).
* ✅ Có 2 mode:

  * **developer mode**: nhanh, nhưng vẫn an toàn tối thiểu
  * **CI mode**: strict, reproducible
* ❌ Không “auto-fix everything” vô điều kiện (vì có breaking changes).
* ✅ Phải có **exception process** (temporary allow) có expiry.
* ✅ Phải chứng minh lockfile strategy bằng test: “install reproducible”.

---

## Deliverables (bạn phải nộp)

Tạo `kata-66/` với:

1. `docs/dependency-policy.md`
2. `docs/lockfile-strategy.md`
3. `docs/security-gates.md`
4. `docs/incident-drill.md`
5. `scripts/audit.mjs`
6. `scripts/license-check.mjs` *(mock được nếu không muốn kéo tool)*
7. `scripts/lockfile-verify.mjs`
8. `.github/workflows/deps.yml` *(hoặc pipeline tương đương)*
9. `test/kata66.repro.test.ts` *(test logic verify scripts; không cần mạng)*

---

## Phần A — Dependency Policy (viết vào `docs/dependency-policy.md`)

### Rule set tối thiểu (bạn áp dụng cho repo)

**1) Dependency hygiene**

* `dependencies`: chỉ runtime thật sự.
* `devDependencies`: test/build/lint.
* Cấm import dev-only vào runtime (stretch: ESLint rule).

**2) New dependency intake checklist**

* Direct deps phải có lý do rõ (1–2 câu): *“why we need it”*.
* Prefer built-in Node libs trước.
* Prefer “boring” libs: stable, maintained, broad usage.
* Check:

  * last release recency
  * maintainer bus factor
  * download trend (chỉ cần ghi “looked at”)
  * known alternatives

**3) Registry & integrity**

* Chỉ dùng npm registry chính (trừ khi org private registry).
* Bật `npm` integrity (mặc định).
* Cấm `postinstall` scripts trừ allowlist (stretch).

**4) Versioning**

* `dependencies` (runtime): **exact pin** hoặc `~` (khuyến nghị: exact cho services).
* `devDependencies`: `^` OK nhưng phải lockfile.
* Không dùng “latest” tag trong CI.

---

## Phần B — Lockfile Strategy (viết vào `docs/lockfile-strategy.md`)

### Yêu cầu kata (bạn chọn 1 chiến lược và ghi rõ)

**Option 1 (khuyến nghị cho service):**

* Commit lockfile (`package-lock.json` hoặc `pnpm-lock.yaml`).
* CI dùng install **frozen**:

  * npm: `npm ci`
  * pnpm: `pnpm install --frozen-lockfile`
  * yarn: `yarn install --immutable`
* Update dependencies qua PR định kỳ (weekly) + security hotfix.

**Pinning**

* runtime deps: exact (vd: `"zod": "3.24.1"`)
* dev deps: `^` ok nhưng lockfile đảm bảo reproducible

**Reproducible guarantee**

* Một commit → cài đặt ra đúng tree.
* Nếu lockfile thay đổi mà `package.json` không thay đổi → phải có lý do (security refresh, dedupe, lockfile version bump) và CI sẽ bắt.

---

## Phần C — Security Gates (viết vào `docs/security-gates.md`)

### Gate 1: Vulnerability scan

* Chạy `npm audit --json` (hoặc `pnpm audit`, `yarn npm audit`) trong CI.
* Policy:

  * fail build nếu có `critical` hoặc `high` trong runtime deps
  * dev deps: allow `high` nếu không ảnh hưởng runtime? (bạn phải quyết định, ghi rõ)
* Exception file: `security/exceptions.json` gồm:

  * package, advisory id, reason, owner, expiresAt (date)

### Gate 2: License policy

* Allowlist: `MIT`, `Apache-2.0`, `BSD-2-Clause`, `BSD-3-Clause`, `ISC`
* Denylist: `GPL-*` (nếu bạn muốn tránh viral), `AGPL-*`
* Nếu unknown license → fail

*(Nếu bạn không muốn tích hợp tool thật, bạn mock bằng cách parse `node_modules/*/package.json` trong local — nhưng trong CI thực tế sẽ cần tool.)*

### Gate 3: Risk heuristics (supply chain)

Implement check trong script:

* Reject package name giống typosquatting với list top deps (stretch).
* Flag nếu dependency mới có `postinstall`/`install` script (warn/fail).
* Flag nếu major bump trong lockfile mà không có PR note.

---

## Phần D — Scripts (nhìn vào làm được)

### `scripts/audit.mjs` (skeleton)

```js
import fs from "node:fs";

function loadExceptions() {
  try {
    return JSON.parse(fs.readFileSync("security/exceptions.json", "utf8"));
  } catch {
    return { exceptions: [] };
  }
}

function isExpired(ex) {
  return Date.now() > new Date(ex.expiresAt).getTime();
}

function main() {
  // TODO:
  // 1) read audit json from stdin OR from file `audit.json` (kata-friendly)
  // 2) compute findings by severity and whether runtime/dev
  // 3) apply exceptions (must not be expired)
  // 4) enforce thresholds and exit(1) on fail
  //
  // Output: summary lines that are safe (no giant dumps)
}

main();
```

### `scripts/lockfile-verify.mjs` (skeleton)

```js
import fs from "node:fs";
import crypto from "node:crypto";

function sha256(path) {
  return crypto.createHash("sha256").update(fs.readFileSync(path)).digest("hex");
}

function main() {
  // TODO:
  // - verify lockfile exists
  // - record hash to `artifacts/lockfile.sha`
  // - in CI, compare to committed hash or ensure lockfile unchanged after install
  // Kata-friendly approach:
  //   - run install step
  //   - git diff --exit-code lockfile  (in CI)
}

main();
```

### `scripts/license-check.mjs` (skeleton)

```js
import fs from "node:fs";
import path from "node:path";

const ALLOW = new Set(["MIT","Apache-2.0","BSD-2-Clause","BSD-3-Clause","ISC"]);
const DENY_PREFIX = ["GPL", "AGPL"];

function isDenied(lic) {
  if (!lic) return true;
  if (DENY_PREFIX.some(p => lic.startsWith(p))) return true;
  return !ALLOW.has(lic);
}

function main() {
  // TODO:
  // - walk node_modules (shallow is fine) and read package.json license
  // - collect denied/unknown
  // - fail if any
  // Note: for kata, you may scan a mocked `fixtures/node_modules`.
}

main();
```

---

## CI Workflow (deliverable `.github/workflows/deps.yml`)

Phải có các step logic:

1. Checkout
2. Setup Node (pin version)
3. Install **reproducible**
4. Run:

   * `node scripts/lockfile-verify.mjs`
   * `npm audit --json > audit.json` (hoặc fixture) + `node scripts/audit.mjs < audit.json`
   * `node scripts/license-check.mjs`
5. Upload audit summary artifact (optional)

---

## Tests (kata-friendly, không cần mạng)

Bạn tạo `fixtures/audit.high-runtime.json`, `fixtures/audit.high-dev.json`, `fixtures/audit.with-exception.json`.

### `test/kata66.repro.test.ts` (ý tưởng)

* spawn `node scripts/audit.mjs` với fixture input
* assert exit code đúng (pass/fail)
* assert expired exception fails
* assert runtime high fails, dev-only high passes (nếu policy bạn chọn)

---

## Incident Drill (viết vào `docs/incident-drill.md`)

Bạn phải mô phỏng scenario:

**Alert**: “CVE trong `lodash` transitive used by `some-lib` severity HIGH, affects prototype pollution”.

Checklist phải có:

1. **Triage**

   * Is it runtime path? reachable? exploitable in our usage?
   * Which versions affected?
2. **Decide**

   * Hotfix now vs scheduled
3. **Patch**

   * Bump direct dep / add override / resolutions / npm `overrides`
4. **Verify**

   * run tests + e2e smoke
   * confirm audit clean
   * check lockfile diff contains only expected packages
5. **Release**

   * canary → full
6. **Postmortem note**

   * root cause (dependency chain)
   * preventive follow-up (reduce deps, pin, add gate)

---

## Checks (Definition of Done)

Bạn pass kata nếu:

1. Có đầy đủ 4 docs (policy/lockfile/gates/incident).
2. Có scripts chạy được với fixtures và trả exit code đúng.
3. CI file thể hiện install frozen + audit + license + lockfile verify.
4. Có exception process có expiry và test chứng minh expiry bị reject.
5. Có quyết định rõ ràng về runtime vs dev vulnerability threshold.

---

## Stretch (Senior+ “Ops & supply chain”)

1. **npm/pnpm overrides policy**

   * Document khi nào dùng overrides, và cách tránh “dependency hell”.
2. **Provenance / integrity**

   * Bật `npm config set fund false`? (optional)
   * Sigstore/provenance (chỉ cần note nếu không implement).
3. **Detect install scripts**

   * Fail nếu có package mới thêm `postinstall` trừ allowlist.
4. **Renovate/Dependabot strategy**

   * Group updates, automerge patch-level dev deps, manual for runtime majors.
5. **SBOM**

   * Generate SBOM (CycloneDX) và lưu artifact (chỉ cần skeleton).