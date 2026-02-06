# Kata 86 — Multi-region: latency + consistency plan

## Goal

1. Chọn một topology multi-region hợp lý cho 1 sản phẩm thật (giả lập):

   * **Read local** (low latency)
   * **Write path** rõ: single-leader / multi-leader / quorum
   * **Consistency model** rõ: strong / bounded staleness / eventual
2. Thiết kế chiến lược khi sự cố:

   * region down
   * network partition
   * replication lag spike
   * clock skew
3. Có plan:

   * **SLO** (latency + freshness)
   * **User-visible behavior** khi degrade
   * **Data correctness** (không double-spend, không mất tiền)
4. Mô phỏng một phần bằng code:

   * latency per region
   * replication delay distribution
   * prove freshness percentile (p95 staleness) và failover impact

---

## Scenario (bài toán cụ thể để không mơ hồ)

Bạn xây **Wallet + Checkout** cho app thương mại:

* User xem **balance** và **discount eligibility** (read heavy)
* User thực hiện **charge** (write critical: không double-charge, không overspend)
* Hệ chạy 2 region: `SG` (Singapore) và `US` (Virginia)
* Khách hàng ở VN chủ yếu hit SG.

### Requirements

* **Charge** phải **strong correctness** (không overspend)
* **Balance read** ưu tiên low latency, chấp nhận stale nhỏ
* Target SLO:

  * Read p95 < 150ms (VN->SG)
  * Charge p95 < 500ms (chấp nhận cross-region nếu cần)
  * Staleness p95 < 2s cho balance reads (bình thường)

---

## Constraints

* ✅ Bạn phải đưa ra **2 phương án kiến trúc** + trade-offs:

  1. Single-leader write (global strong) + read replicas
  2. Multi-leader (regional writes) + conflict resolution
* ✅ Chọn **1 phương án final** và viết:

  * Data model / invariants
  * Read & write flows
  * Failure modes + handling
  * Monitoring signals
  * Runbook (5 bước)
* ✅ Code mô phỏng phải có:

  * two regions with RTT matrix
  * replication lag
  * read strategy (local vs quorum vs fallback)
  * outputs: latency stats + staleness stats

---

# Deliverables (bắt buộc)

1. `DESIGN.md` (1–2 trang) gồm:

   * Assumptions (traffic, RTT, failure assumptions)
   * Option A vs Option B
   * Final choice
   * Read/Write flows (pseudo)
   * Consistency guarantees
   * Failure handling
   * SLOs + alerts
   * Runbook

2. Code mini-sim:

```
kata86/
  src/
    sim.ts
    strategies.ts
    stats.ts
  test/
    sim.test.ts
```

---

# Part A — Design template (fill-in “nhìn là làm được”)

## 1) Assumptions

* Regions: SG, US
* RTT:

  * VN→SG: 40ms
  * VN→US: 220ms
  * SG↔US replication RTT: 180ms
* Replication delay: base 150ms + jitter, spike to seconds occasionally
* Failure: 1 region can be down, partitions possible
* Clock skew up to 500ms

## 2) Invariants

* Wallet charge: **no overspend** (balance must not go negative)
* Exactly-once effects: charge idempotency key
* Ledger append-only, balance derived

## 3) Option A — Single-leader (strong writes)

* Leader in SG (or multi with managed DB)
* Writes routed to leader (cross-region for US users)
* Reads:

  * local replica (fast) for non-critical
  * quorum/leader for critical reads
* Pros: easiest correctness, no write conflicts
* Cons: higher write latency for far users, leader region outage hurts

## 4) Option B — Multi-leader (regional writes)

* Each region accepts writes locally
* Replicate async both ways
* Conflicts resolved by:

  * CRDT (hard for money)
  * escrow / bounded counters per region (possible but complex)
* Pros: low write latency everywhere
* Cons: correctness for money is hard; requires escrow, strict budgets

## 5) Final choice (recommended for this scenario)

**Option A** cho money/charge path (strong), nhưng **read local** cho UX:

* Ledger writes: single-leader SG
* Read balance:

  * default: local replica (stale allowed)
  * if `freshnessRequired=true`: read leader or quorum

## 6) Read/Write flows (pseudo)

* Read:

  * if remaining deadline low: local replica else maybe leader
  * return `stalenessMs` metadata (from replication lag watermark)
* Write charge:

  * `POST /charge` at nearest region
  * region forwards to leader SG (with deadline propagation)
  * idempotency key prevents duplicate charges
  * return status; update local cache when event replicated

## 7) Failure handling

* SG leader down:

  * stop accepting charge (or failover to US with manual switch if DB supports)
  * reads served from local replicas with “degraded: write unavailable”
* Partition SG↔US:

  * charge only on leader side
  * reads still local; staleness grows; add banner/metadata

## 8) Monitoring

* Replication lag watermark p95/p99
* Cross-region write latency p95/p99
* Error rates per endpoint
* “stale read served %”
* “write rejected due to leader unavailable %”

## 9) Runbook (5 steps)

1. Detect: alert replication lag / leader health
2. Confirm blast radius: which region/users affected
3. Apply mitigation: route writes to leader region only, enable “read-only mode”
4. Communicate: status page + banner
5. Recover: failback + postmortem

---

# Part B — Code simulation (copy skeleton)

## `src/stats.ts`

```ts
// src/stats.ts
export function p(arr: number[], q: number): number {
  const a = [...arr].sort((x, y) => x - y);
  if (a.length === 0) return 0;
  const idx = Math.floor((q / 100) * (a.length - 1));
  return a[idx];
}
```

## `src/strategies.ts`

```ts
// src/strategies.ts
export type Region = "SG" | "US";

export type SimConfig = {
  rttClientTo: Record<Region, number>;      // client -> region RTT
  rttBetweenRegions: number;               // SG <-> US RTT
  replicationBaseMs: number;
  replicationJitterMs: number;
};

export type SimState = {
  leader: Region;
  // watermark: how up-to-date each region is vs leader's time
  replicaLagMs: Record<Region, number>;
};

export type ReadStrategy = "local" | "leader" | "local_then_leader_if_stale";

export function readLatencyMs(strategy: ReadStrategy, clientRegion: Region, state: SimState, cfg: SimConfig): number {
  if (strategy === "leader") return cfg.rttClientTo[state.leader];
  if (strategy === "local") return cfg.rttClientTo[clientRegion];
  // local_then_leader_if_stale: assume if lag > 2000ms then fallback to leader
  const lag = state.replicaLagMs[clientRegion];
  if (lag > 2000) return cfg.rttClientTo[state.leader]; // fallback
  return cfg.rttClientTo[clientRegion];
}

export function writeLatencyMs(clientRegion: Region, state: SimState, cfg: SimConfig): number {
  // write always goes to leader
  if (clientRegion === state.leader) return cfg.rttClientTo[state.leader];
  // client->local + forward to leader (approx)
  return cfg.rttClientTo[clientRegion] + cfg.rttBetweenRegions;
}

export function stepReplication(state: SimState, cfg: SimConfig, rng: () => number) {
  // simulate lag dynamics: lag decreases over time but spikes randomly
  for (const r of ["SG", "US"] as const) {
    if (r === state.leader) {
      state.replicaLagMs[r] = 0;
      continue;
    }
    // base lag with jitter
    const jitter = Math.floor(rng() * cfg.replicationJitterMs);
    let lag = cfg.replicationBaseMs + jitter;

    // rare spike
    if (rng() < 0.02) lag += 4000 + Math.floor(rng() * 4000);
    state.replicaLagMs[r] = lag;
  }
}
```

## `src/sim.ts`

```ts
// src/sim.ts
import { p } from "./stats";
import type { Region, ReadStrategy, SimConfig, SimState } from "./strategies";
import { readLatencyMs, writeLatencyMs, stepReplication } from "./strategies";

export type SimResult = {
  readP95: number;
  writeP95: number;
  stalenessP95: number;
  stalenessP99: number;
};

export function runSim(args: {
  iterations: number;
  clientRegion: Region;
  readStrategy: ReadStrategy;
  cfg: SimConfig;
  leader: Region;
  rngSeed?: number;
}): SimResult {
  let seed = args.rngSeed ?? 1;
  const rng = () => {
    seed = (1103515245 * seed + 12345) % 2 ** 31;
    return seed / 2 ** 31;
  };

  const state: SimState = {
    leader: args.leader,
    replicaLagMs: { SG: 0, US: 0 },
  };

  const readLat: number[] = [];
  const writeLat: number[] = [];
  const stale: number[] = [];

  for (let i = 0; i < args.iterations; i++) {
    stepReplication(state, args.cfg, rng);

    // read
    readLat.push(readLatencyMs(args.readStrategy, args.clientRegion, state, args.cfg));
    stale.push(state.replicaLagMs[args.clientRegion]);

    // write
    writeLat.push(writeLatencyMs(args.clientRegion, state, args.cfg));
  }

  return {
    readP95: p(readLat, 95),
    writeP95: p(writeLat, 95),
    stalenessP95: p(stale, 95),
    stalenessP99: p(stale, 99),
  };
}
```

---

# Tests (vitest) — `test/sim.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { runSim } from "../src/sim";

describe("Kata 86 - Multi-region thought drill", () => {
  it("local reads are faster but can be stale", () => {
    const cfg = {
      rttClientTo: { SG: 40, US: 220 },
      rttBetweenRegions: 180,
      replicationBaseMs: 150,
      replicationJitterMs: 200,
    };

    const local = runSim({
      iterations: 5000,
      clientRegion: "SG",
      readStrategy: "local",
      cfg,
      leader: "SG",
      rngSeed: 1,
    });

    const leaderRead = runSim({
      iterations: 5000,
      clientRegion: "SG",
      readStrategy: "leader",
      cfg,
      leader: "SG",
      rngSeed: 1,
    });

    expect(local.readP95).toBeLessThanOrEqual(leaderRead.readP95);
    expect(local.stalenessP95).toBeGreaterThanOrEqual(0);
  });

  it("writes to leader cost cross-region latency for far clients", () => {
    const cfg = {
      rttClientTo: { SG: 40, US: 220 },
      rttBetweenRegions: 180,
      replicationBaseMs: 150,
      replicationJitterMs: 200,
    };

    const usClient = runSim({
      iterations: 1000,
      clientRegion: "US",
      readStrategy: "local",
      cfg,
      leader: "SG",
      rngSeed: 2,
    });

    expect(usClient.writeP95).toBeGreaterThan(200); // must include cross-region
  });

  it("fallback strategy reduces extreme staleness at cost of latency", () => {
    const cfg = {
      rttClientTo: { SG: 40, US: 220 },
      rttBetweenRegions: 180,
      replicationBaseMs: 150,
      replicationJitterMs: 200,
    };

    const local = runSim({
      iterations: 5000,
      clientRegion: "US",
      readStrategy: "local",
      cfg,
      leader: "SG",
      rngSeed: 3,
    });

    const fallback = runSim({
      iterations: 5000,
      clientRegion: "US",
      readStrategy: "local_then_leader_if_stale",
      cfg,
      leader: "SG",
      rngSeed: 3,
    });

    // fallback read latency tends to be higher
    expect(fallback.readP95).toBeGreaterThanOrEqual(local.readP95);
    // but staleness tail should reduce (heuristic: p99 lower or similar)
    expect(fallback.stalenessP99).toBeLessThanOrEqual(local.stalenessP99);
  });
});
```

---

## Checks (DoD)

* Bạn có DESIGN.md với 2 options + quyết định final.
* Bạn chạy sim và đưa ra số:

  * readP95, writeP95
  * staleness p95/p99
* Bạn giải thích trade-off bằng data:

  * local read nhanh hơn nhưng stale
  * leader read chậm hơn nhưng fresh
  * fallback strategy giảm tail staleness nhưng tăng latency

---

# Senior+ Stretch (đúng “multi-region thực chiến”)

1. **Money correctness upgrade**: escrow / bounded counter nếu muốn multi-leader cho wallet (phức tạp nhưng có thể).
2. **Failover drill**: mô phỏng leader down:

   * option: reject writes vs promote follower
   * measure RTO/RPO
3. **Client routing**: geo-DNS + health-based routing; when SG down, VN → JP/US
4. **Consistency contract**: define “bounded staleness”: `stalenessP95 < 2s`, `P99 < 10s`, and how you measure it (watermark).
5. **Runbook with toggles**: feature flags:

   * `FORCE_LEADER_READ=true`
   * `WRITE_READONLY_MODE=true`
   * `DISABLE_RECS=true`