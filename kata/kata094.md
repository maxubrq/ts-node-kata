# Kata 94 — Cold Start: Optimize Boot Time (Boot/Init Latency)

## Goal (Senior+)

Bạn sẽ xây một service Node có cold start “cố tình chậm”, rồi:

1. **Đo cold start** theo định nghĩa rõ ràng (T0→Ready).
2. **Trace startup critical path** (module load, config, DI, connect DB, warm caches).
3. Tối ưu theo 3 nhóm:

   * **load-time** (require/import graph)
   * **init-time** (sync heavy work)
   * **ready-time** (network/IO: DB, remote calls)
4. Chứng minh cải thiện bằng số liệu + p95.
5. Thêm **startup regression gate** vào CI.

---

## Context

Bạn deploy serverless/container scale-to-zero. Khi scale up:

* request đầu tiên bị chậm (cold start)
* readiness chậm khiến autoscaling phản ứng tệ
* cost tăng do instance chạy lâu hơn

Bạn cần boot nhanh, nhưng vẫn **đúng** (config validated, health readiness đúng, không race).

---

## Constraints

* ✅ Node 20+ (khuyên) hoặc 18+.
* ✅ Không dùng framework nặng (khuyến nghị http core hoặc fastify minimal).
* ✅ Phải có **startup marker** rõ ràng: “Ready”.
* ✅ Phải có **measurement harness** chạy N lần và report mean/median/p95.
* ✅ Phải có ít nhất 3 “nguồn chậm” ban đầu (intentionally).
* ❌ Không “ăn gian” bằng cách bỏ validation/security.
* ❌ Không đo bằng 1 lần duy nhất.

---

## Deliverables

```
packages/kata-94/
  src/
    server.ts
    bootstrap.ts
    readiness.ts
    slow/
      big-import.ts
      schema-validate.ts
      fake-remote.ts
  bench/
    coldstart.ts
  scripts/
    run-many.ts
  test/
    readiness.spec.ts
  README.md
```

Bạn giao:

1. Service chạy được: `node dist/src/server.js`
2. `run-many` chạy 30–100 cold starts (spawn process) và xuất report JSON
3. Before/after tối ưu: số liệu + giải thích critical path
4. Regression gate rule

---

## Định nghĩa Cold Start (bạn phải dùng)

Chọn 1 definition và cố định:

### Definition A (khuyến nghị)

**T0 = process start**
**T_ready = khi HTTP server listen + readiness returns 200**

ColdStart = `T_ready - T0`

> Đây là metric thực dụng nhất cho autoscaling / k8s readiness.

---

## Starter skeleton (nhìn vào làm ngay)

### 1) `src/bootstrap.ts` (startup timeline markers)

```ts
// src/bootstrap.ts
export type Marker = { name: string; atNs: bigint };

export class BootTimeline {
  private t0 = process.hrtime.bigint();
  private marks: Marker[] = [{ name: "process_start", atNs: this.t0 }];

  mark(name: string) {
    this.marks.push({ name, atNs: process.hrtime.bigint() });
  }

  snapshot() {
    const base = this.t0;
    return this.marks.map(m => ({
      name: m.name,
      msFromStart: Number(m.atNs - base) / 1e6,
    }));
  }
}
```

### 2) Intentional slow modules

#### `src/slow/big-import.ts` (cố tình tạo load-time cost)

```ts
// src/slow/big-import.ts
// Simulate heavy module graph + parse time.
// (Do NOT do this in real code—this is the kata's "enemy".)
export const BIG_TABLE = Array.from({ length: 200_000 }, (_, i) => `k_${i}_${Math.sqrt(i)}`);
```

#### `src/slow/schema-validate.ts` (init-time cost)

```ts
// src/slow/schema-validate.ts
export function validateConfig(raw: Record<string, unknown>) {
  // Intentionally expensive validation (simulate heavy schema work)
  // TODO (later): optimize by doing only once, lazy parts, or move off critical path
  let s = 0;
  for (let i = 0; i < 2_000_000; i++) s = (s + i) | 0;

  if (typeof raw["PORT"] !== "number") throw new Error("PORT must be number");
  return { port: raw["PORT"] as number };
}
```

#### `src/slow/fake-remote.ts` (ready-time cost)

```ts
// src/slow/fake-remote.ts
export async function fetchRemoteFeatureFlags(): Promise<Record<string, boolean>> {
  // Simulate network call (startup blocker)
  await new Promise(r => setTimeout(r, 250));
  return { newRanking: true, debug: false };
}
```

### 3) `src/readiness.ts`

```ts
// src/readiness.ts
export type ReadyState = {
  ready: boolean;
  reason?: string;
};

export class Readiness {
  private state: ReadyState = { ready: false, reason: "booting" };

  setReady() { this.state = { ready: true }; }
  setNotReady(reason: string) { this.state = { ready: false, reason }; }

  get() { return this.state; }
}
```

### 4) `src/server.ts` (baseline slow startup)

```ts
// src/server.ts
import http from "node:http";
import { BootTimeline } from "./bootstrap";
import { Readiness } from "./readiness";

// INTENTIONALLY BAD: heavy import on critical path
import { BIG_TABLE } from "./slow/big-import";
import { validateConfig } from "./slow/schema-validate";
import { fetchRemoteFeatureFlags } from "./slow/fake-remote";

async function main() {
  const timeline = new BootTimeline();
  const readiness = new Readiness();

  timeline.mark("imports_done");

  // config validation (blocking)
  const cfg = validateConfig({ PORT: 3000 });
  timeline.mark("config_validated");

  // remote flags (blocking readiness)
  const flags = await fetchRemoteFeatureFlags();
  timeline.mark("flags_fetched");

  // use BIG_TABLE so it can't be tree-shaken (simulate real usage)
  const tableSize = BIG_TABLE.length;
  timeline.mark("big_table_used");

  const server = http.createServer((req, res) => {
    if (req.url === "/healthz") {
      res.writeHead(200).end("ok");
      return;
    }
    if (req.url === "/readyz") {
      const s = readiness.get();
      res.writeHead(s.ready ? 200 : 503, { "content-type": "application/json" });
      res.end(JSON.stringify({ ...s, tableSize, flags }));
      return;
    }
    res.writeHead(200).end("hello");
  });

  server.listen(cfg.port, () => {
    timeline.mark("server_listening");
    readiness.setReady();
    timeline.mark("ready");

    // Print measurement line for runner to parse
    console.log(JSON.stringify({
      type: "BOOT_REPORT",
      pid: process.pid,
      timeline: timeline.snapshot(),
    }));
  });
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

---

## Measurement harness (bắt buộc)

### `scripts/run-many.ts`

Chạy nhiều cold starts bằng `spawn`, parse `BOOT_REPORT`, compute p50/p95.

```ts
// scripts/run-many.ts
import { spawn } from "node:child_process";

type BootReport = { timeline: { name: string; msFromStart: number }[] };

function percentile(xs: number[], p: number) {
  const a = [...xs].sort((x, y) => x - y);
  const idx = Math.min(a.length - 1, Math.floor((p / 100) * (a.length - 1)));
  return a[idx];
}

async function runOnce(): Promise<number> {
  return new Promise((resolve, reject) => {
    const child = spawn(process.execPath, ["dist/src/server.js"], {
      stdio: ["ignore", "pipe", "pipe"],
      env: { ...process.env },
    });

    let buf = "";
    child.stdout.on("data", (d) => {
      buf += d.toString("utf8");
      const lines = buf.split("\n");
      buf = lines.pop() ?? "";
      for (const line of lines) {
        try {
          const obj = JSON.parse(line);
          if (obj?.type === "BOOT_REPORT") {
            const rep = obj as BootReport;
            const ready = rep.timeline.find(x => x.name === "ready");
            if (!ready) return reject(new Error("missing ready marker"));
            resolve(ready.msFromStart);
            child.kill("SIGKILL");
            return;
          }
        } catch {}
      }
    });

    child.on("error", reject);
    child.on("exit", (code) => {
      if (code && code !== 0) reject(new Error(`exit ${code}`));
    });
  });
}

async function main() {
  const n = Number(process.argv[2] ?? 30);
  const xs: number[] = [];
  for (let i = 0; i < n; i++) xs.push(await runOnce());

  const mean = xs.reduce((a, b) => a + b, 0) / xs.length;
  const p50 = percentile(xs, 50);
  const p95 = percentile(xs, 95);

  const report = { n, mean, p50, p95, samples: xs };
  console.log(JSON.stringify({ type: "COLDSTART_SUMMARY", report }, null, 2));
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

---

## Nhiệm vụ tối ưu (bạn phải làm)

Bạn phải đạt **ít nhất 2/3 nhóm** sau, và chứng minh effect bằng timeline + run-many.

### Group 1 — Import/Load-time

* **Lazy import** module nặng (`big-import`) sau readiness
* Hoặc tách BIG_TABLE thành file data load lazy
* Hoặc chuyển sang “compute-on-demand” thay vì init

**Expected:** giảm `imports_done → ready`

### Group 2 — Init-time (CPU blocking)

* Optimize schema validation:

  * validate “core required fields” trước, phần expensive làm sau
  * hoặc memoize/skip heavy loop (kata enemy)
* Chuyển một phần init sang background sau `server.listen`

**Expected:** giảm `config_validated` time

### Group 3 — Ready-time (IO)

* Remote flags:

  * không block readiness: start server, mark ready, fetch flags in background
  * hoặc dùng cached flags local + refresh async
* Readiness semantics:

  * `/readyz` chỉ 200 khi *critical deps* ok
  * feature flags thường **không critical** → không block ready

**Expected:** `ready` không còn chờ `flags_fetched`

---

## Correctness guard (bắt buộc)

* `test/readiness.spec.ts`:

  * trước flags: `/readyz` 200 (nếu bạn quyết định flags non-critical)
  * nếu bạn quyết định flags critical: `/readyz` phải 503 cho đến khi flags ok
* Bạn phải document “critical deps list”.

---

## Checks (Definition of Done)

Bạn pass nếu:

1. `run-many.ts 30` cho baseline và optimized.
2. Có report JSON với:

   * mean, p50, p95
   * timeline breakdown
3. Optimized cải thiện:

   * **p95 cold start giảm ≥ 30%** (mục tiêu hợp lý với baseline cố tình chậm)
4. Readiness semantics đúng và có test.

---

## Stretch (Senior+ → Staff-grade)

Chọn ≥ 5:

1. **Startup budget**: đặt ngân sách rõ:

   * `p95 cold start <= 200ms` (hoặc mục tiêu bạn chọn)
2. **Regression gate** trong CI:

   * fail nếu p95 > budget hoặc tăng > 10% so với last known
3. **Critical path visualization**:

   * in timeline dạng bảng (marker deltas)
4. **Config & DI trimming**:

   * tránh import toàn bộ container/registry khi không cần
5. **Precompile / Bundle experiment**:

   * so sánh `tsup` bundle vs node resolution (có số liệu)
6. **Warm path check**:

   * đo second start trong cùng process (khác cold start) để tránh hiểu sai
7. **Deployment realism**:

   * set `NODE_ENV=production`, disable source maps, compare effect

---

## README template (bạn phải điền)

* Definition cold start (T0→Ready)
* Baseline numbers (mean/p50/p95)
* Timeline hotspots
* Changes made (group 1/2/3)
* Optimized numbers + delta
* Readiness contract + tests
* Regression gate rule