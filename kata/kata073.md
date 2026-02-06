# Kata 73 — Feature Flagging: Runtime Flags + Kill Switch

## Goal

1. Xây **Feature Flag system tối thiểu nhưng chuẩn production**:

   * runtime evaluation theo **context** (tenant/user/route)
   * **kill switch** (global emergency off) có hiệu lực ngay
   * **staged rollout** (percentage rollout) ổn định theo user
2. Có **safe defaults** khi flag store/remote config lỗi.
3. Có **auditability**: log/metric khi flag quyết định hành vi (nhất là kill switch).

---

## Context

Bạn đang có use-case `transferMoney` (từ Kata 71/72). Bạn muốn rollout tính năng mới: **“fee_v2”** (tính phí chuyển tiền kiểu mới).

* Khi `fee_v2` ON: tính fee = `max(100, amount*1%)` (cents)
* Khi OFF: fee = 0
* Có `kill_switch_transfers`: nếu ON → **disable toàn bộ transfer** (trả lỗi 503/feature_disabled), bất kể flag khác.

Flag requirements:

* `fee_v2`:

  * enabled cho **tenant A**
  * rollout 10% cho các tenant khác (dựa theo userId stable hashing)
* `kill_switch_transfers`:

  * bật/tắt tức thì (runtime)
  * ưu tiên cao nhất

---

## Constraints

* ✅ Flag evaluation nằm trong **application layer** (policy), không nằm trong controller/infra.
* ✅ Flag values có thể update runtime mà **không restart process**.
* ✅ Có **FlagProvider** port và ít nhất 2 implementations:

  * InMemory provider (cho test)
  * File/Env polling provider (infra) (mô phỏng remote config)
* ✅ Có **deterministic percentage rollout** (không random mỗi request).
* ✅ Có **caching + TTL** (để tránh đọc file quá nhiều) nhưng kill switch phải “nhanh”.
* ✅ Có tests:

  * unit tests cho rollout hashing
  * application tests cho kill switch precedence
  * failure-mode tests: provider throw → fallback

---

## Deliverables (repo shape)

```
kata-73/
  src/
    application/
      flags/
        types.ts
        evaluator.ts
        flagService.ts
      usecases/
        transferMoneyWithFees.ts
    infra/
      flags/
        inMemoryProvider.ts
        filePollingProvider.ts
    main/
      compositionRoot.ts
  test/
    flags.rollout.test.ts
    flags.killSwitch.test.ts
    flags.failureMode.test.ts
  README.md
```

---

## Flag model (Spec)

### Flag keys

* `fee_v2`
* `kill_switch_transfers`

### Flag evaluation context

```ts
export type FlagContext = {
  tenantId: string;
  userId?: string;
  requestId?: string;
  route?: string;
};
```

### Rule types

* **Boolean flag**: true/false
* **Targeting**: allowlist tenants/users
* **Percentage rollout**: 0..100 by stable hash of userId (or requestId fallback)
* **Kill switch**: global override OFF for a feature (or disable endpoint)

---

## Starter skeleton (điền TODO)

### `src/application/flags/types.ts`

```ts
export type FlagKey = "fee_v2" | "kill_switch_transfers";

export type FlagContext = {
  tenantId: string;
  userId?: string;
  requestId?: string;
  route?: string;
};

export type FlagDecision = {
  key: FlagKey;
  enabled: boolean;
  reason:
    | "KILL_SWITCH"
    | "TENANT_ALLOWLIST"
    | "USER_ALLOWLIST"
    | "PERCENT_ROLLOUT"
    | "DEFAULT"
    | "PROVIDER_ERROR";
  variant?: string;
};

export type FlagRule =
  | { type: "boolean"; enabled: boolean }
  | { type: "tenant_allowlist"; tenants: string[]; enabled: boolean }
  | { type: "user_allowlist"; users: string[]; enabled: boolean }
  | { type: "percent_rollout"; percent: number; seed: "userId"; enabledWhenInBucket: boolean };

export type FlagConfig = {
  key: FlagKey;
  // rules evaluated in order
  rules: FlagRule[];
  defaultEnabled: boolean;
};

export interface FlagProvider {
  /** May throw (simulate remote config failure) */
  getAll(): Promise<FlagConfig[]>;
}
```

### `src/application/flags/evaluator.ts`

```ts
import type { FlagConfig, FlagContext, FlagDecision, FlagKey, FlagRule } from "./types";

/** TODO: stable hash to [0, 99] based on userId */
export function stableBucket(seed: string): number {
  // constraint: deterministic, fast, no crypto libs needed
  // hint: FNV-1a 32-bit or simple rolling hash
  return 0;
}

function evalRule(rule: FlagRule, ctx: FlagContext): { matched: boolean; enabled?: boolean; reason?: FlagDecision["reason"] } {
  switch (rule.type) {
    case "boolean":
      return { matched: true, enabled: rule.enabled, reason: "DEFAULT" };

    case "tenant_allowlist":
      if (rule.tenants.includes(ctx.tenantId)) return { matched: true, enabled: rule.enabled, reason: "TENANT_ALLOWLIST" };
      return { matched: false };

    case "user_allowlist":
      if (ctx.userId && rule.users.includes(ctx.userId)) return { matched: true, enabled: rule.enabled, reason: "USER_ALLOWLIST" };
      return { matched: false };

    case "percent_rollout": {
      // TODO: if no userId -> matched false (or fallback to requestId, your call; but be explicit)
      if (!ctx.userId) return { matched: false };
      const b = stableBucket(ctx.userId); // 0..99
      const inBucket = b < rule.percent;
      const enabled = inBucket ? rule.enabledWhenInBucket : !rule.enabledWhenInBucket;
      return { matched: true, enabled, reason: "PERCENT_ROLLOUT" };
    }
  }
}

export function evaluateFlag(config: FlagConfig, ctx: FlagContext): FlagDecision {
  for (const rule of config.rules) {
    const r = evalRule(rule, ctx);
    if (r.matched) {
      return { key: config.key, enabled: !!r.enabled, reason: r.reason ?? "DEFAULT" };
    }
  }
  return { key: config.key, enabled: config.defaultEnabled, reason: "DEFAULT" };
}
```

### `src/application/flags/flagService.ts`

FlagService = cache + TTL + kill switch precedence.

```ts
import type { FlagConfig, FlagContext, FlagDecision, FlagKey, FlagProvider } from "./types";
import { evaluateFlag } from "./evaluator";

export type FlagServiceDeps = {
  provider: FlagProvider;
  nowMs: () => number;
  ttlMs: number;                 // e.g. 5_000
  killSwitchTtlMs: number;       // e.g. 250 (faster refresh)
  log: (d: FlagDecision, ctx: FlagContext) => void;
};

export class FlagService {
  private cache: { at: number; configs: FlagConfig[] } | null = null;
  private killCache: { at: number; killEnabled: boolean } | null = null;

  constructor(private deps: FlagServiceDeps) {}

  private async loadAll(): Promise<FlagConfig[]> {
    const now = this.deps.nowMs();
    if (this.cache && now - this.cache.at < this.deps.ttlMs) return this.cache.configs;

    const configs = await this.deps.provider.getAll();
    this.cache = { at: now, configs };
    return configs;
  }

  private async loadKillSwitch(): Promise<boolean> {
    const now = this.deps.nowMs();
    if (this.killCache && now - this.killCache.at < this.deps.killSwitchTtlMs) return this.killCache.killEnabled;

    const all = await this.loadAll();
    const kill = all.find(c => c.key === "kill_switch_transfers");
    const enabled = kill ? evaluateFlag(kill, { tenantId: "GLOBAL" }).enabled : false; // ctx irrelevant for global
    this.killCache = { at: now, killEnabled: enabled };
    return enabled;
  }

  /** Evaluate a flag; enforces kill-switch precedence for transfer-related flags */
  async isEnabled(key: FlagKey, ctx: FlagContext): Promise<FlagDecision> {
    try {
      // TODO: precedence:
      // if key === "fee_v2" AND kill_switch_transfers is ON -> return disabled reason KILL_SWITCH
      if (key === "fee_v2") {
        const kill = await this.loadKillSwitch();
        if (kill) {
          const d: FlagDecision = { key, enabled: false, reason: "KILL_SWITCH" };
          this.deps.log(d, ctx);
          return d;
        }
      }

      const all = await this.loadAll();
      const cfg = all.find(c => c.key === key);
      const d = cfg ? evaluateFlag(cfg, ctx) : ({ key, enabled: false, reason: "DEFAULT" } as const);
      this.deps.log(d, ctx);
      return d;
    } catch (e) {
      const d: FlagDecision = { key, enabled: false, reason: "PROVIDER_ERROR" };
      this.deps.log(d, ctx);
      return d;
    }
  }
}
```

### `src/application/usecases/transferMoneyWithFees.ts`

Use-case dùng FlagService; kill switch disable request.

```ts
import type { FlagService } from "../flags/flagService";

export type TransferInput = {
  tenantId: string;
  userId: string;
  fromWalletId: string;
  toWalletId: string;
  amountCents: number;
  idempotencyKey: string;
};

export type TransferOutput =
  | { ok: true; transferId: string; feeCents: number; debitedCents: number }
  | { ok: false; code: "FEATURE_DISABLED" | "INVALID" | "FAILED"; message: string };

export function makeTransferMoneyWithFees(deps: {
  flags: FlagService;
  // TODO: inject existing transferMoney use-case from kata-72 app kernel
  transferMoney: (args: any, ctx: any) => Promise<any>;
}) {
  return async function transferMoneyWithFees(input: TransferInput) {
    // 1) kill switch hard-stop
    const kill = await deps.flags.isEnabled("kill_switch_transfers", { tenantId: input.tenantId, userId: input.userId });
    if (kill.enabled) {
      return { ok: false, code: "FEATURE_DISABLED", message: "Transfers temporarily disabled" } as const;
    }

    // 2) fee flag
    const feeFlag = await deps.flags.isEnabled("fee_v2", { tenantId: input.tenantId, userId: input.userId });

    const feeCents = feeFlag.enabled ? Math.max(100, Math.floor(input.amountCents * 0.01)) : 0;
    const debited = input.amountCents + feeCents;

    // TODO: call underlying transferMoney for `amountCents: debited`
    // Provide ctx with logger/requestId in real app. For kata, you can pass dummy ctx.

    // return ok + fee info
    return { ok: true, transferId: "TODO", feeCents, debitedCents: debited } as const;
  };
}
```

---

## Infra providers

### `src/infra/flags/inMemoryProvider.ts`

```ts
import type { FlagConfig, FlagProvider } from "../../application/flags/types";

export class InMemoryFlagProvider implements FlagProvider {
  constructor(private configs: FlagConfig[]) {}
  set(configs: FlagConfig[]) { this.configs = configs; }
  async getAll(): Promise<FlagConfig[]> { return this.configs; }
}
```

### `src/infra/flags/filePollingProvider.ts`

Mô phỏng “remote config”: đọc JSON file định kỳ.

```ts
import type { FlagConfig, FlagProvider } from "../../application/flags/types";
import { readFile } from "node:fs/promises";

export class FilePollingProvider implements FlagProvider {
  constructor(private path: string) {}
  async getAll(): Promise<FlagConfig[]> {
    const raw = await readFile(this.path, "utf-8");
    const parsed = JSON.parse(raw);
    // TODO: minimal validation
    return parsed as FlagConfig[];
  }
}
```

---

## Flag config mẫu (để test)

```json
[
  {
    "key": "kill_switch_transfers",
    "rules": [{ "type": "boolean", "enabled": false }],
    "defaultEnabled": false
  },
  {
    "key": "fee_v2",
    "rules": [
      { "type": "tenant_allowlist", "tenants": ["tenantA"], "enabled": true },
      { "type": "percent_rollout", "percent": 10, "seed": "userId", "enabledWhenInBucket": true }
    ],
    "defaultEnabled": false
  }
]
```

---

## Tests (bắt buộc)

### 1) Rollout hashing stable: `test/flags.rollout.test.ts`

* `stableBucket("u1")` luôn ra cùng số
* 1000 userIds phân bố tương đối đều (không cần perfect)
* percent=10 → ~10% enabled (± vài %)

### 2) Kill switch precedence: `test/flags.killSwitch.test.ts`

* bật kill switch → `fee_v2` decision trả `enabled:false` reason `KILL_SWITCH`
* `transferMoneyWithFees` trả `FEATURE_DISABLED` ngay cả khi fee flag ON

### 3) Failure mode: `test/flags.failureMode.test.ts`

* provider throw → `isEnabled` trả `enabled:false` reason `PROVIDER_ERROR`
* kill switch refresh TTL nhỏ hơn TTL thường (test qua fake clock)

---

## Checks (Definition of Done)

* ✅ Flag evaluation deterministic, context-aware.
* ✅ Kill switch hard-stop, ưu tiên cao nhất.
* ✅ Runtime update không restart: thay config trong provider → effect trong TTL.
* ✅ Provider failure → safe default OFF, có reason + log.
* ✅ Có tests cover rollout + precedence + failure.

---

## Stretch (Senior+ → rất cao)

1. **Sticky variants (A/B)**
   Thay `enabled:boolean` bằng `variant: "control" | "treatment"`; stable hash map variant.
2. **Rule language**
   Thêm rule `match_route` / `match_header` (để canary theo route).
3. **Audit log + metric**
   Counter: `flag_decision_total{key,reason,enabled}` + sample log cho kill switch.
4. **Safety: “flag debt”**
   Add `expiresAt` trong FlagConfig; nếu quá hạn → log error + auto disable (policy).
5. **Dual-layer kill switch**
   local env kill switch (ENV var) override remote config (khi remote config compromised).