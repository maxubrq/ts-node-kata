# Kata 98 — Flaky Test Killer: Create Flakes, Kill Them Deterministically

## Goal (Senior+)

Bạn sẽ:

1. Tạo một test suite có **flake rate rõ ràng** (ví dụ fail 2–10%).
2. Phân loại flaky theo taxonomy (time, concurrency, env, randomness, order, shared state).
3. Build **flake reproducer**: chạy suite 200–1000 lần, thống kê fail rate.
4. Fix từng flake bằng chiến thuật deterministic.
5. Chứng minh:

   * fail rate → 0%
   * không cần “retry”
   * runtime không phình vô lý

---

## Constraints

* ✅ Node + TS, `vitest` khuyến nghị.
* ✅ Bạn phải tạo **ít nhất 4 test flaky** với 4 nguyên nhân khác nhau.
* ✅ Phải có script `flake-hunt` chạy lặp nhiều lần và in report.
* ✅ Fix theo nguyên tắc: **deterministic by design**, không “tăng timeout” bừa.
* ❌ Không dùng retry để che flake (retry=0).
* ❌ Không skip/flaky-ignore.

---

## Deliverables

```
packages/kata-98/
  src/
    time.ts
    rng.ts
    db.ts
    http.ts
    service.ts
  test/
    flaky/
      flaky.time.spec.ts
      flaky.random.spec.ts
      flaky.order.spec.ts
      flaky.race.spec.ts
    fixed/
      fixed.time.spec.ts
      fixed.random.spec.ts
      fixed.order.spec.ts
      fixed.race.spec.ts
  scripts/
    flake-hunt.ts
  README.md
```

Bạn giao:

1. 4 flaky tests (mỗi cái có note nguyên nhân)
2. 4 fixed tests tương ứng
3. `flake-hunt` report trước/sau
4. README: playbook diệt flake

---

# Part 1 — Flake taxonomy (bắt buộc ghi trong README)

Bạn phải ghi:

* **Time flake**: phụ thuộc `Date.now()`, timers, scheduling.
* **Random flake**: RNG không seed, data phụ thuộc entropy.
* **Order/Shared-state flake**: tests chạy khác thứ tự hoặc share global state.
* **Race/Concurrency flake**: promise không await, event loop timing, port collisions.

---

# Part 2 — Build the flaky surfaces (src)

### `src/time.ts`

```ts
export type Clock = { now(): number };

export const SystemClock: Clock = { now: () => Date.now() };

export function isExpired(createdAtMs: number, ttlMs: number, clock: Clock = SystemClock) {
  return clock.now() - createdAtMs >= ttlMs;
}
```

### `src/rng.ts`

```ts
export type Rng = { next(): number };

// BAD default: Math.random
export const SystemRng: Rng = { next: () => Math.random() };

// deterministic rng
export function xorshift32(seed: number): Rng {
  let x = seed >>> 0;
  return {
    next() {
      x ^= x << 13; x >>>= 0;
      x ^= x >>> 17; x >>>= 0;
      x ^= x << 5; x >>>= 0;
      return (x >>> 0) / 2 ** 32;
    },
  };
}
```

### `src/db.ts` (shared global state “tội lỗi”)

```ts
type Row = { id: string; value: number };

const GLOBAL: Row[] = []; // intentionally global

export function insert(row: Row) {
  GLOBAL.push(row);
}

export function count() {
  return GLOBAL.length;
}

export function reset() {
  GLOBAL.length = 0;
}
```

### `src/http.ts` (race/port collision surface)

```ts
import http from "node:http";

export async function startServer(port: number, handler: http.RequestListener) {
  const server = http.createServer(handler);
  await new Promise<void>((resolve, reject) => {
    server.listen(port, resolve);
    server.on("error", reject);
  });
  return server;
}
```

---

# Part 3 — Create 4 flaky tests (cố tình)

## Flaky #1 — Time boundary (deadline jitter)

### `test/flaky/flaky.time.spec.ts`

```ts
import { describe, it, expect } from "vitest";
import { isExpired } from "../../src/time";

describe("flaky time", () => {
  it("should not expire exactly at boundary (flaky)", async () => {
    const createdAt = Date.now();
    const ttl = 20;

    // jitter depends on scheduling
    await new Promise(r => setTimeout(r, 20));

    // sometimes true, sometimes false
    expect(isExpired(createdAt, ttl)).toBe(false);
  });
});
```

**Nguyên nhân**: boundary đúng bằng ttl + scheduling → thỉnh thoảng vượt.

---

## Flaky #2 — Randomness (non-seeded)

### `test/flaky/flaky.random.spec.ts`

```ts
import { describe, it, expect } from "vitest";

describe("flaky random", () => {
  it("rarely fails due to randomness", () => {
    // 2% fail
    const x = Math.random();
    expect(x).toBeGreaterThan(0.02);
  });
});
```

**Nguyên nhân**: RNG.

---

## Flaky #3 — Order/shared state (test order)

### `test/flaky/flaky.order.spec.ts`

```ts
import { describe, it, expect } from "vitest";
import { insert, count } from "../../src/db";

describe("flaky order/shared state", () => {
  it("assumes db is empty (flaky)", () => {
    expect(count()).toBe(0); // depends on other tests
  });

  it("writes to db", () => {
    insert({ id: "a", value: 1 });
    expect(count()).toBeGreaterThan(0);
  });
});
```

**Nguyên nhân**: shared global db.

---

## Flaky #4 — Race condition / missing await / port collision

### `test/flaky/flaky.race.spec.ts`

```ts
import { describe, it, expect } from "vitest";
import { startServer } from "../../src/http";

describe("flaky race", () => {
  it("starts server on fixed port (flaky)", async () => {
    // parallel runs / other tests can conflict
    const port = 3333;

    const server = await startServer(port, (req, res) => {
      res.end("ok");
    });

    // no proper cleanup timing -> can leak
    server.close();

    expect(true).toBe(true);
  });
});
```

**Nguyên nhân**: fixed port + cleanup race.

---

# Part 4 — Flake-hunting harness (bắt buộc)

### `scripts/flake-hunt.ts`

* chạy `vitest` N lần (ví dụ 200)
* thống kê pass/fail rate
* in top failing files

Gợi ý implementation: spawn `npx vitest run test/flaky --reporter=json` và parse exit code (đủ dùng).
Bạn phải chạy trước và sau fix, ghi số liệu vào README.

---

# Part 5 — Fix deterministically (4 test fixed)

## Fix #1 — Control time (inject clock / fake timers)

### `test/fixed/fixed.time.spec.ts`

Cách 1 (khuyến nghị): inject clock:

```ts
import { describe, it, expect } from "vitest";
import { isExpired, type Clock } from "../../src/time";

describe("fixed time", () => {
  it("boundary is deterministic with fake clock", () => {
    let t = 1000;
    const clock: Clock = { now: () => t };

    const createdAt = t;
    const ttl = 20;

    t = 1019;
    expect(isExpired(createdAt, ttl, clock)).toBe(false);

    t = 1020;
    expect(isExpired(createdAt, ttl, clock)).toBe(true);
  });
});
```

---

## Fix #2 — Seed randomness

### `test/fixed/fixed.random.spec.ts`

```ts
import { describe, it, expect } from "vitest";
import { xorshift32 } from "../../src/rng";

describe("fixed random", () => {
  it("deterministic random stream", () => {
    const rng = xorshift32(42);
    const xs = Array.from({ length: 5 }, () => rng.next());
    expect(xs).toMatchObject(xs); // stable, you can snapshot exact values if you want
    expect(xs[0]).toBeGreaterThanOrEqual(0);
    expect(xs[0]).toBeLessThan(1);
  });
});
```

---

## Fix #3 — Isolate state per test (reset in beforeEach)

### `test/fixed/fixed.order.spec.ts`

```ts
import { describe, it, expect, beforeEach } from "vitest";
import { insert, count, reset } from "../../src/db";

describe("fixed order/shared state", () => {
  beforeEach(() => reset());

  it("db is empty at start", () => {
    expect(count()).toBe(0);
  });

  it("writes are isolated", () => {
    insert({ id: "a", value: 1 });
    expect(count()).toBe(1);
  });
});
```

---

## Fix #4 — Avoid fixed ports + ensure cleanup awaited

### `test/fixed/fixed.race.spec.ts`

```ts
import { describe, it, expect } from "vitest";
import { startServer } from "../../src/http";

function listenRandomPort() {
  return 0; // Node picks an available port
}

describe("fixed race", () => {
  it("uses random port and awaits close", async () => {
    const server = await startServer(listenRandomPort(), (req, res) => res.end("ok"));

    await new Promise<void>((resolve) => server.close(() => resolve()));
    expect(true).toBe(true);
  });
});
```

---

# Part 6 — Checks (Definition of Done)

Bạn pass kata nếu:

1. `flake-hunt` trên `test/flaky` cho thấy **fail rate > 0%** (trước fix).
2. `flake-hunt` trên `test/fixed` cho thấy **fail rate = 0%** qua ít nhất **200 runs**.
3. Không dùng retries, không tăng timeout vô tội vạ.
4. README có bảng:

   * flake type → root cause → fix pattern → rule để ngăn tái phát.

---

# Stretch (Senior+ → Staff-grade)

Chọn ≥ 6:

1. **Quarantine pipeline**: nếu test fail, tự động rerun 20 lần để xác định flake vs real regression.
2. **Add flake tags**: mark known flake type (time/random/order/race).
3. **Deterministic scheduler**: wrap async jobs to control execution order in tests.
4. **Hermetic DB**: thay global store bằng per-test instance injected.
5. **CI parallelism proof**: chạy fixed tests với `--pool=forks --maxWorkers=...` vẫn stable.
6. **Anti-flake lint rules**: forbid `Date.now()` / `Math.random()` trong tests trừ khi injected.
7. **Postmortem template for flakes**: cause, fix, prevention guard.

---

## Bonus: “Flake Killer Playbook” (1 trang, bạn nên viết)

* Không retry.
* Freeze time.
* Seed RNG.
* Isolate state.
* No fixed ports.
* Always await async cleanup.
* Never assert on timing of real scheduler.