# Kata 46 — Load Shedding: drop/deny chiến lược khi quá tải

## Goal

1. Implement **load shedding** (fail-fast) với nhiều chiến lược: **deny**, **drop**, **degrade**, dựa trên tín hiệu quá tải.
2. Overload signals (ít nhất 3):

   * queue depth / queue wait time (từ Kata 41/44/45)
   * latency tail (p95/p99) hoặc “timeout rate”
   * event-loop lag (từ Kata 16) / saturation (inflight gần trần)
3. Áp chính sách theo **route / class / tenant** (ưu tiên SLA), và chứng minh: khi overload, hệ **không chết** và route quan trọng vẫn sống.
4. Có **metrics**: shed rate theo reason, latency giảm, error budget, admitted vs rejected.
5. Có test deterministic (fake clock) + simulation chứng minh “stabilize”.

---

## Context

Khi hệ quá tải, mục tiêu không phải “phục vụ tất cả”, mà là:

* bảo vệ core (checkout/auth)
* giữ latency trong ngưỡng chấp nhận được
* tránh meltdown (queue phình, retry storm, GC thrash)

Load shedding là “cắt bớt” đúng chỗ để tàu không chìm.

---

## Constraints

* ❌ Không chỉ “rate limit” rồi gọi là shedding. Shedding là **phản ứng theo overload** và **ưu tiên SLA**.
* ✅ Phải có **ít nhất 3 policy** và 3 overload signals.
* ✅ Có **priority-aware**: P0 không bị shed (hoặc shed rất muộn), P2 shed trước.
* ✅ Có **hysteresis** (tránh flapping): vào overload và thoát overload có ngưỡng khác nhau hoặc cooldown.
* ✅ Có “reason codes” rõ ràng: `QUEUE_FULL`, `QUEUE_TIMEOUT_RISK`, `TAIL_LATENCY`, `EVENT_LOOP_LAG`, `CIRCUIT_OPEN`, `BUDGET_EXCEEDED`.
* ✅ Không leak resource: request bị deny không chiếm slot/queue.

---

## Deliverables

1. `src/shedder.ts` — LoadShedder engine (signals → decision)
2. `src/signals.ts` — signal collectors (queue depth, lag, tail latency)
3. `src/policies.ts` — strategies (deny/drop/degrade)
4. `src/sim.ts` — simulation: traffic mix + overload injection
5. `test/shedder.test.ts` — unit tests cho decisions + hysteresis
6. `README.md` — reasoning + trade-offs + tuning guide

---

## Bạn sẽ build cái gì (API)

```ts
export type TrafficClass = "P0" | "P1" | "P2";

export type RequestMeta = {
  route: string;          // e.g. "POST /checkout"
  klass: TrafficClass;    // SLA class
  tenant?: string;        // optional for fairness
  id?: string;
};

export type OverloadSignals = {
  now: number;

  // saturation style
  inflight: number;
  inflightCap: number;

  // queue style (per route/class, but you can pass aggregated)
  queueDepth: number;
  queueCap: number;
  queueWaitP95Ms: number;

  // latency style
  latencyP95Ms: number;
  errorRate: number;        // 0..1

  // runtime style
  eventLoopLagMs: number;
};

export type ShedDecision =
  | { action: "ALLOW" }
  | { action: "DENY"; reason: string; retryAfterMs?: number }
  | { action: "DEGRADE"; mode: "CACHE_ONLY" | "STALE_OK" | "SKIP_DOWNSTREAM"; reason: string };

export type LoadShedderConfig = {
  // thresholds + hysteresis
  enterOverload: {
    inflightRatio?: number;       // e.g. 0.9
    queueRatio?: number;          // e.g. 0.9
    queueWaitP95Ms?: number;      // e.g. 200
    latencyP95Ms?: number;        // e.g. 500
    eventLoopLagMs?: number;      // e.g. 50
    errorRate?: number;           // e.g. 0.2
  };
  exitOverload: {
    inflightRatio?: number;       // e.g. 0.7
    queueRatio?: number;          // e.g. 0.7
    queueWaitP95Ms?: number;      // e.g. 120
    latencyP95Ms?: number;        // e.g. 350
    eventLoopLagMs?: number;      // e.g. 30
    errorRate?: number;           // e.g. 0.1
  };
  cooldownMs: number;             // minimum time to stay in overload once entered

  // SLA-aware rules
  classRules: Record<TrafficClass, {
    // when overload, what to do first
    strategy: "DENY" | "DEGRADE";
    denyProbability?: number;     // optional probabilistic shedding
    degradeMode?: "CACHE_ONLY" | "STALE_OK" | "SKIP_DOWNSTREAM";
    retryAfterMs?: number;
  }>;

  // optional: route overrides
  routeRules?: Record<string, Partial<LoadShedderConfig["classRules"][TrafficClass]>>;
};

export class LoadShedder {
  constructor(cfg: LoadShedderConfig);

  updateSignals(s: OverloadSignals): void;
  decide(req: RequestMeta): ShedDecision;

  snapshot(): {
    inOverload: boolean;
    lastEnterAt?: number;
    reasons: Record<string, number>;
    deniedByClass: Record<TrafficClass, number>;
    degradedByClass: Record<TrafficClass, number>;
    allowedTotal: number;
  };
}
```

---

## Core semantics bắt buộc

### 1) Overload state machine (hysteresis + cooldown)

* Hệ có state: `NORMAL` hoặc `OVERLOADED`
* Enter OVERLOADED nếu **bất kỳ** enter threshold bị vượt (hoặc scoring ≥ threshold — stretch)
* Khi đã OVERLOADED:

  * phải giữ ít nhất `cooldownMs`
  * chỉ exit khi **tất cả** exit thresholds đã “an toàn” (hoặc scoring < threshold)

> Mục đích: tránh flapping “đang deny lại allow” mỗi 100ms.

### 2) Decision theo SLA class

* NORMAL: ALLOW (trừ các hard rejects như queue full)
* OVERLOADED:

  * P0: thường **ALLOW** hoặc **DEGRADE nhẹ** (tuỳ config), deny rất muộn
  * P1: degrade hoặc deny theo tỉ lệ
  * P2: deny trước (hoặc drop) để giải áp lực

### 3) Reason codes (bắt buộc)

Mỗi decision DENY/DEGRADE phải có reason:

* `INFLIGHT_SATURATION`
* `QUEUE_SATURATION`
* `QUEUE_WAIT_RISK`
* `TAIL_LATENCY`
* `EVENT_LOOP_LAG`
* `ERROR_BURST`

---

## Starter skeleton (nhìn vào code)

```ts
// src/shedder.ts
export type TrafficClass = "P0" | "P1" | "P2";
export type RequestMeta = { route: string; klass: TrafficClass; tenant?: string; id?: string };

export type OverloadSignals = {
  now: number;
  inflight: number; inflightCap: number;
  queueDepth: number; queueCap: number; queueWaitP95Ms: number;
  latencyP95Ms: number; errorRate: number;
  eventLoopLagMs: number;
};

export type ShedDecision =
  | { action: "ALLOW" }
  | { action: "DENY"; reason: string; retryAfterMs?: number }
  | { action: "DEGRADE"; mode: "CACHE_ONLY" | "STALE_OK" | "SKIP_DOWNSTREAM"; reason: string };

export type LoadShedderConfig = {
  enterOverload: Partial<{
    inflightRatio: number;
    queueRatio: number;
    queueWaitP95Ms: number;
    latencyP95Ms: number;
    eventLoopLagMs: number;
    errorRate: number;
  }>;
  exitOverload: Partial<{
    inflightRatio: number;
    queueRatio: number;
    queueWaitP95Ms: number;
    latencyP95Ms: number;
    eventLoopLagMs: number;
    errorRate: number;
  }>;
  cooldownMs: number;
  classRules: Record<TrafficClass, {
    strategy: "DENY" | "DEGRADE";
    denyProbability?: number; // 0..1
    degradeMode?: "CACHE_ONLY" | "STALE_OK" | "SKIP_DOWNSTREAM";
    retryAfterMs?: number;
  }>;
  routeRules?: Record<string, Partial<Record<TrafficClass, Partial<LoadShedderConfig["classRules"][TrafficClass]>>>>;
};

type Snapshot = {
  inOverload: boolean;
  lastEnterAt?: number;
  reasons: Record<string, number>;
  deniedByClass: Record<TrafficClass, number>;
  degradedByClass: Record<TrafficClass, number>;
  allowedTotal: number;
};

export class LoadShedder {
  private inOverload = false;
  private lastEnterAt?: number;
  private lastSignals?: OverloadSignals;

  private reasons: Record<string, number> = {};
  private deniedByClass: Snapshot["deniedByClass"] = { P0: 0, P1: 0, P2: 0 };
  private degradedByClass: Snapshot["degradedByClass"] = { P0: 0, P1: 0, P2: 0 };
  private allowedTotal = 0;

  constructor(private cfg: LoadShedderConfig, private deps: { rand?: () => number } = {}) {}

  updateSignals(s: OverloadSignals) {
    this.lastSignals = s;
    this.updateStateMachine(s);
  }

  snapshot(): Snapshot {
    return {
      inOverload: this.inOverload,
      lastEnterAt: this.lastEnterAt,
      reasons: { ...this.reasons },
      deniedByClass: { ...this.deniedByClass },
      degradedByClass: { ...this.degradedByClass },
      allowedTotal: this.allowedTotal,
    };
  }

  decide(req: RequestMeta): ShedDecision {
    const s = this.lastSignals;
    if (!s) return { action: "ALLOW" };

    // Hard fast-fail if queue already full (integration with bulkhead/priority queue)
    if (s.queueCap > 0 && s.queueDepth >= s.queueCap) {
      return this.deny(req.klass, "QUEUE_SATURATION", 100);
    }

    if (!this.inOverload) {
      this.allowedTotal++;
      return { action: "ALLOW" };
    }

    const reason = this.primaryReason(s);
    const rule = this.resolveRule(req);

    if (rule.strategy === "DEGRADE") {
      const mode = rule.degradeMode ?? "SKIP_DOWNSTREAM";
      return this.degrade(req.klass, mode, reason);
    }

    // DENY strategy (optionally probabilistic)
    const p = rule.denyProbability ?? 1;
    const rand = this.deps.rand?.() ?? Math.random();
    if (rand <= p) return this.deny(req.klass, reason, rule.retryAfterMs);

    this.allowedTotal++;
    return { action: "ALLOW" };
  }

  private resolveRule(req: RequestMeta) {
    const base = this.cfg.classRules[req.klass];
    const routeOverride = this.cfg.routeRules?.[req.route]?.[req.klass];
    return { ...base, ...(routeOverride ?? {}) };
  }

  private updateStateMachine(s: OverloadSignals) {
    const now = s.now;

    if (!this.inOverload) {
      const enter = this.isEnterOverload(s);
      if (enter) {
        this.inOverload = true;
        this.lastEnterAt = now;
      }
      return;
    }

    // in overload
    const sinceEnter = this.lastEnterAt ? now - this.lastEnterAt : Number.POSITIVE_INFINITY;
    if (sinceEnter < this.cfg.cooldownMs) return;

    const exit = this.isExitOverload(s);
    if (exit) {
      this.inOverload = false;
    }
  }

  private isEnterOverload(s: OverloadSignals): boolean {
    // TODO: if any enter threshold is breached => true
    // remember: ratios are inflight/cap, queueDepth/queueCap
    return this.anyBreach(s, this.cfg.enterOverload);
  }

  private isExitOverload(s: OverloadSignals): boolean {
    // TODO: require ALL exit thresholds to be safe => true
    return this.allSafe(s, this.cfg.exitOverload);
  }

  private anyBreach(s: OverloadSignals, t: LoadShedderConfig["enterOverload"]) {
    const inflightRatio = s.inflightCap > 0 ? s.inflight / s.inflightCap : 0;
    const queueRatio = s.queueCap > 0 ? s.queueDepth / s.queueCap : 0;

    if (t.inflightRatio !== undefined && inflightRatio >= t.inflightRatio) return true;
    if (t.queueRatio !== undefined && queueRatio >= t.queueRatio) return true;
    if (t.queueWaitP95Ms !== undefined && s.queueWaitP95Ms >= t.queueWaitP95Ms) return true;
    if (t.latencyP95Ms !== undefined && s.latencyP95Ms >= t.latencyP95Ms) return true;
    if (t.eventLoopLagMs !== undefined && s.eventLoopLagMs >= t.eventLoopLagMs) return true;
    if (t.errorRate !== undefined && s.errorRate >= t.errorRate) return true;

    return false;
  }

  private allSafe(s: OverloadSignals, t: LoadShedderConfig["exitOverload"]) {
    const inflightRatio = s.inflightCap > 0 ? s.inflight / s.inflightCap : 0;
    const queueRatio = s.queueCap > 0 ? s.queueDepth / s.queueCap : 0;

    if (t.inflightRatio !== undefined && inflightRatio > t.inflightRatio) return false;
    if (t.queueRatio !== undefined && queueRatio > t.queueRatio) return false;
    if (t.queueWaitP95Ms !== undefined && s.queueWaitP95Ms > t.queueWaitP95Ms) return false;
    if (t.latencyP95Ms !== undefined && s.latencyP95Ms > t.latencyP95Ms) return false;
    if (t.eventLoopLagMs !== undefined && s.eventLoopLagMs > t.eventLoopLagMs) return false;
    if (t.errorRate !== undefined && s.errorRate > t.errorRate) return false;

    return true;
  }

  private primaryReason(s: OverloadSignals): string {
    // pick the “most alarming” signal for reason code (simple priority order)
    const inflightRatio = s.inflightCap > 0 ? s.inflight / s.inflightCap : 0;
    const queueRatio = s.queueCap > 0 ? s.queueDepth / s.queueCap : 0;

    if (queueRatio >= (this.cfg.enterOverload.queueRatio ?? 1.1)) return "QUEUE_SATURATION";
    if (s.queueWaitP95Ms >= (this.cfg.enterOverload.queueWaitP95Ms ?? Infinity)) return "QUEUE_WAIT_RISK";
    if (s.latencyP95Ms >= (this.cfg.enterOverload.latencyP95Ms ?? Infinity)) return "TAIL_LATENCY";
    if (s.eventLoopLagMs >= (this.cfg.enterOverload.eventLoopLagMs ?? Infinity)) return "EVENT_LOOP_LAG";
    if (s.errorRate >= (this.cfg.enterOverload.errorRate ?? Infinity)) return "ERROR_BURST";
    if (inflightRatio >= (this.cfg.enterOverload.inflightRatio ?? Infinity)) return "INFLIGHT_SATURATION";

    return "OVERLOADED";
  }

  private deny(klass: TrafficClass, reason: string, retryAfterMs?: number): ShedDecision {
    this.reasons[reason] = (this.reasons[reason] ?? 0) + 1;
    this.deniedByClass[klass]++;
    return { action: "DENY", reason, retryAfterMs };
  }

  private degrade(klass: TrafficClass, mode: "CACHE_ONLY" | "STALE_OK" | "SKIP_DOWNSTREAM", reason: string): ShedDecision {
    this.reasons[reason] = (this.reasons[reason] ?? 0) + 1;
    this.degradedByClass[klass]++;
    return { action: "DEGRADE", mode, reason };
  }
}
```

---

## Policies bắt buộc (ít nhất 3)

### Policy 1 — Hard deny khi queue gần đầy

* Nếu `queueDepth/queueCap >= 1.0` ⇒ deny tất cả trừ P0 (hoặc P0 degrade)
* Reason: `QUEUE_SATURATION`

### Policy 2 — Probabilistic shedding theo class khi inflight/latency cao

* OVERLOADED:

  * P2: denyProbability = 1.0
  * P1: denyProbability = 0.5
  * P0: allow hoặc degrade
* Reason: `INFLIGHT_SATURATION` hoặc `TAIL_LATENCY`

### Policy 3 — Degrade mode (serve stale / skip downstream)

* OVERLOADED:

  * P0: `DEGRADE: STALE_OK` (serve cache + background refresh)
  * P1: `DEGRADE: CACHE_ONLY`
  * P2: deny
* Reason: `EVENT_LOOP_LAG` / `TAIL_LATENCY`

> “degrade” ở đây là contract thay đổi có chủ đích (ví dụ bỏ enrichment, trả dữ liệu cache, tắt expensive downstream).

---

## Test plan bắt buộc (deterministic)

### Test 1 — Enter overload + cooldown + exit (hysteresis)

* feed signals vượt enter threshold ⇒ inOverload=true
* hạ signals xuống safe nhưng chưa qua cooldown ⇒ vẫn overload
* qua cooldown và signals < exit threshold ⇒ exit overload

### Test 2 — SLA-aware decisions

* inOverload=true
* decide(P2) ⇒ DENY
* decide(P1) ⇒ DENY hoặc DEGRADE theo config
* decide(P0) ⇒ ALLOW hoặc DEGRADE, không DENY (trừ khi bạn config)

### Test 3 — Reason code correctness

* set eventLoopLag high ⇒ decision reason phải là `EVENT_LOOP_LAG` (hoặc theo priority bạn định nghĩa)

### Test 4 — Queue full hard reject

* set queueDepth==queueCap ⇒ P1/P2 DENY ngay, không phụ thuộc overload state

### Test 5 — Probabilistic shedding deterministic

* inject rand() fixed:

  * rand=0.4, denyProbability=0.5 ⇒ DENY
  * rand=0.6 ⇒ ALLOW

---

## Simulation bắt buộc (chứng minh “stabilize”)

Bạn viết `src/sim.ts` chạy 60s simulated time:

* Traffic mix: P0 50 rps, P1 200 rps, P2 500 rps (spike)
* Downstream capacity fixed: inflightCap=100
* Khi không shedding: queueDepth tăng vô hạn, latency p95 tăng > 5s
* Khi bật shedding: queueDepth ổn định quanh cap, latency p95 giảm mạnh, P0 success rate giữ cao

Bạn in ra mỗi 1s:

* `inOverload`, queueDepth, latencyP95, deniedByClass, degradedByClass

---

## Checks (Definition of Done)

* Có overload state machine + hysteresis + cooldown.
* Có ≥3 signals và ≥3 strategies.
* P0 vẫn phục vụ được trong overload (ALLOW hoặc DEGRADE), P2 bị shed trước.
* Metrics snapshot đầy đủ và reason codes rõ.
* Simulation cho thấy hệ không “runaway” (queue không phình vô hạn, latency không leo mãi).

---

## README bắt buộc (ngắn nhưng “ops-grade”)

1. Load shedding khác rate limit ở đâu
2. Signals bạn dùng và vì sao chọn chúng
3. Enter/exit thresholds + cooldown rationale
4. SLA class rules (P0/P1/P2)
5. Degrade modes (cache/stale/skip) và contract của response
6. Reason codes mapping
7. Alerting gợi ý: shed rate, deadline miss, queue depth
8. Failure modes: shed quá tay, flapping, unfair tenant
9. Tuning playbook: điều chỉnh thresholds theo p95 + saturation
10. Liên kết: Bulkhead (44) + Priority Queue (45) + Circuit Breaker (43)

---

## Stretch (Senior+ thật)

Chọn 1:

1. **Token bucket budget per class**: mỗi class có “admission budget” theo thời gian (giống error budget).
2. **Tenant fairness**: khi overload, deny theo tenant heavy hitter trước.
3. **Feedback controller**: mục tiêu giữ latencyP95 dưới X bằng cách điều chỉnh denyProbability tự động.
4. **Multi-stage shedding**: warn → degrade → deny theo mức overload.
5. **Integration thực**: middleware Express/Fastify trả `429`/`503` kèm `Retry-After` + trace reason.