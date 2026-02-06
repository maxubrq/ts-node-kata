# Kata 74 — Config System: Layered + Schema + Reload

## Goal

1. Build **Config System production-grade** cho Node/TS:

   * **Layered**: defaults → file → env → runtime overrides
   * **Schema validation**: parse unknown → typed config (Parse Don’t Validate)
   * **Reload**: file watcher / polling, cập nhật runtime không restart
2. Có **policy** rõ:

   * secrets không log
   * fail-fast hay degrade tùy field
   * change notification (pub/sub) cho components (FlagService, logger, rate limit…)
3. Có tests cho:

   * precedence
   * schema validation + error messages
   * reload behavior + debounce
   * partial failure (file config invalid) với strategy

---

## Context

Bạn đang có:

* `FlagService` (Kata 73) cần đọc `flags.json` và reload nhanh
* `Server` cần:

  * `PORT`
  * `LOG_LEVEL`
  * `DB_URL`
  * `REQUEST_TIMEOUT_MS`
  * `FEATURE_FLAGS_TTL_MS`

Bạn muốn:

* local dev: defaults + `.env`
* staging/prod: env overrides
* runtime: `flags.json` đổi là áp dụng (không restart)
* config invalid: **không được silently accept**

---

## Constraints

* ❌ Không được “process.env as any rồi dùng thẳng”
* ✅ Phải có `ConfigSchema` parse từ `unknown` → `AppConfig` typed
* ✅ Layer precedence MUST be deterministic:

  1. defaults
  2. config file(s)
  3. env
  4. runtime overrides (admin / tests)
* ✅ Reload:

  * file reload phải **debounce** (VD 200ms)
  * publish event `config.changed`
* ✅ Secrets:

  * field marked secret phải **redact** khi log / toJSON
* ✅ Failure policy:

  * “critical config” invalid → **fail fast**
  * “reload config invalid” → **keep last-known-good**, emit alert event
* ✅ Không dùng framework config có sẵn (để học bản chất). (Zod cho schema OK nếu bạn muốn; nhưng bạn cũng có thể tự viết parser nhỏ.)

---

## Deliverables (repo shape)

```
kata-74/
  src/
    config/
      schema.ts
      loader.ts
      layers.ts
      store.ts
      redact.ts
      errors.ts
    infra/
      fsWatcher.ts
    main/
      compositionRoot.ts
  test/
    config.precedence.test.ts
    config.schema.test.ts
    config.reload.test.ts
    config.failurePolicy.test.ts
  README.md
```

---

## Spec: AppConfig (bạn phải implement)

### Type (mục tiêu)

```ts
export type AppConfig = {
  env: "dev" | "staging" | "prod";

  server: {
    port: number;                 // 1..65535
    requestTimeoutMs: number;     // 100..120000
  };

  log: {
    level: "debug" | "info" | "warn" | "error";
    pretty: boolean;
  };

  db: {
    url: string;                  // required in staging/prod
    poolSize: number;             // 1..50
  };

  flags: {
    filePath: string;             // e.g. "./flags.json"
    ttlMs: number;                // 100..60000
    killSwitchTtlMs: number;      // 50..5000 (must be <= ttlMs)
  };

  secrets: {
    jwtSecret: string;            // secret (must redact)
  };
};
```

### Policy rules

* `db.url` required if `env !== "dev"`
* `flags.killSwitchTtlMs <= flags.ttlMs`
* `server.port` default 3000
* When reload file invalid → keep last config, emit `config.invalid` with reason

---

## Core Architecture (đúng “Senior+”)

Bạn sẽ có 4 lớp chính:

1. **Schema parser**: `parseConfig(input: unknown): Result<AppConfig, ConfigError[]>`
2. **Layer merge**: merge defaults/file/env/overrides (deep merge)
3. **ConfigStore**: giữ current config, last-known-good, subscribe changes
4. **Reload engine**: watch file(s) + debounce + validate + commit/rollback

---

## Starter skeleton (điền TODO)

### `src/config/errors.ts`

```ts
export type ConfigIssue = {
  path: string;         // e.g. "db.url"
  message: string;      // human readable
  severity: "error" | "warn";
};

export type ParseResult<T> =
  | { ok: true; value: T }
  | { ok: false; issues: ConfigIssue[] };
```

### `src/config/schema.ts`

Không dùng Zod cũng được. Ở đây mình đưa skeleton tự-parse để bạn luyện.

```ts
import type { AppConfig } from "./types";
import type { ParseResult, ConfigIssue } from "./errors";

export type UnknownRecord = Record<string, unknown>;

function isRecord(x: unknown): x is UnknownRecord {
  return typeof x === "object" && x !== null && !Array.isArray(x);
}

function getStr(obj: UnknownRecord, key: string, path: string, issues: ConfigIssue[], opts?: { required?: boolean }) {
  const v = obj[key];
  if (v === undefined || v === null) {
    if (opts?.required) issues.push({ path, message: "is required", severity: "error" });
    return undefined;
  }
  if (typeof v !== "string") {
    issues.push({ path, message: "must be a string", severity: "error" });
    return undefined;
  }
  return v;
}

function getNum(obj: UnknownRecord, key: string, path: string, issues: ConfigIssue[], opts?: { min?: number; max?: number; required?: boolean }) {
  const v = obj[key];
  if (v === undefined || v === null) {
    if (opts?.required) issues.push({ path, message: "is required", severity: "error" });
    return undefined;
  }
  if (typeof v !== "number" || !Number.isFinite(v)) {
    issues.push({ path, message: "must be a number", severity: "error" });
    return undefined;
  }
  if (opts?.min !== undefined && v < opts.min) issues.push({ path, message: `must be >= ${opts.min}`, severity: "error" });
  if (opts?.max !== undefined && v > opts.max) issues.push({ path, message: `must be <= ${opts.max}`, severity: "error" });
  return v;
}

function getBool(obj: UnknownRecord, key: string, path: string, issues: ConfigIssue[], opts?: { required?: boolean }) {
  const v = obj[key];
  if (v === undefined || v === null) {
    if (opts?.required) issues.push({ path, message: "is required", severity: "error" });
    return undefined;
  }
  if (typeof v !== "boolean") {
    issues.push({ path, message: "must be a boolean", severity: "error" });
    return undefined;
  }
  return v;
}

export function parseAppConfig(input: unknown): ParseResult<AppConfig> {
  const issues: ConfigIssue[] = [];
  if (!isRecord(input)) return { ok: false, issues: [{ path: "", message: "config must be an object", severity: "error" }] };

  // TODO: read env, server, log, db, flags, secrets using helpers above
  // TODO: apply defaults when fields missing
  // TODO: cross-field validation:
  // - db.url required if env !== "dev"
  // - killSwitchTtlMs <= ttlMs
  // - port int 1..65535, poolSize int 1..50 etc (you can enforce int via Number.isInteger)

  return { ok: false, issues: [{ path: "", message: "TODO", severity: "error" }] };
}
```

### `src/config/types.ts`

```ts
export type AppConfig = {
  env: "dev" | "staging" | "prod";
  server: { port: number; requestTimeoutMs: number };
  log: { level: "debug" | "info" | "warn" | "error"; pretty: boolean };
  db: { url: string; poolSize: number };
  flags: { filePath: string; ttlMs: number; killSwitchTtlMs: number };
  secrets: { jwtSecret: string };
};
```

### `src/config/layers.ts`

Layer representation + deep merge + env parsing.

```ts
import type { UnknownRecord } from "./schema";

export type ConfigLayer = {
  name: "defaults" | "file" | "env" | "overrides";
  data: UnknownRecord;
};

export function deepMerge(base: UnknownRecord, override: UnknownRecord): UnknownRecord {
  // TODO: deep merge objects; override wins; arrays replaced
  return {};
}

export function mergeLayers(layers: ConfigLayer[]): UnknownRecord {
  return layers.reduce((acc, layer) => deepMerge(acc, layer.data), {} as UnknownRecord);
}

/** Parse env vars -> UnknownRecord in same shape as AppConfig */
export function envToLayer(env: Record<string, string | undefined>): UnknownRecord {
  // TODO: support:
  // APP_ENV
  // PORT
  // REQUEST_TIMEOUT_MS
  // LOG_LEVEL
  // LOG_PRETTY
  // DB_URL
  // DB_POOL_SIZE
  // FLAGS_FILE
  // FLAGS_TTL_MS
  // FLAGS_KILL_TTL_MS
  // JWT_SECRET
  return {};
}
```

### `src/config/redact.ts`

```ts
import type { AppConfig } from "./types";

export function redactConfigForLog(cfg: AppConfig): Record<string, unknown> {
  // TODO: clone & replace cfg.secrets.jwtSecret with "***"
  return {};
}
```

### `src/config/store.ts`

ConfigStore with subscribe + last-known-good.

```ts
import type { AppConfig } from "./types";

type Listener = (next: AppConfig, prev: AppConfig) => void;
type InvalidListener = (issues: { path: string; message: string }[]) => void;

export class ConfigStore {
  private current: AppConfig;
  private listeners: Listener[] = [];
  private invalidListeners: InvalidListener[] = [];

  constructor(initial: AppConfig) {
    this.current = initial;
  }

  get(): AppConfig {
    return this.current;
  }

  onChange(fn: Listener): () => void {
    this.listeners.push(fn);
    return () => { this.listeners = this.listeners.filter(x => x !== fn); };
  }

  onInvalid(fn: InvalidListener): () => void {
    this.invalidListeners.push(fn);
    return () => { this.invalidListeners = this.invalidListeners.filter(x => x !== fn); };
  }

  /** Commit new config (assume already validated) */
  commit(next: AppConfig): void {
    const prev = this.current;
    this.current = next;
    for (const l of this.listeners) l(next, prev);
  }

  emitInvalid(issues: { path: string; message: string }[]): void {
    for (const l of this.invalidListeners) l(issues);
  }
}
```

### `src/config/loader.ts`

Loader loads layers, validates, returns store + reload control.

```ts
import type { AppConfig } from "./types";
import type { UnknownRecord } from "./schema";
import { parseAppConfig } from "./schema";
import { mergeLayers, type ConfigLayer, envToLayer } from "./layers";
import { readFile } from "node:fs/promises";

export type LoaderDeps = {
  readTextFile: (path: string) => Promise<string>;
  env: Record<string, string | undefined>;
  nowMs: () => number;
};

export async function loadFileLayer(path: string, deps: LoaderDeps): Promise<UnknownRecord> {
  const raw = await deps.readTextFile(path);
  return JSON.parse(raw) as UnknownRecord;
}

export async function buildConfig(deps: LoaderDeps, args: { defaults: UnknownRecord; filePath?: string; overrides?: UnknownRecord }) {
  const layers: ConfigLayer[] = [{ name: "defaults", data: args.defaults }];

  if (args.filePath) {
    const file = await loadFileLayer(args.filePath, deps);
    layers.push({ name: "file", data: file });
  }

  layers.push({ name: "env", data: envToLayer(deps.env) });

  if (args.overrides) layers.push({ name: "overrides", data: args.overrides });

  const merged = mergeLayers(layers);
  const parsed = parseAppConfig(merged);

  return { merged, parsed };
}
```

### `src/infra/fsWatcher.ts`

Debounced reload trigger (polling cũng được).

```ts
import { watch } from "node:fs";

export function watchFileDebounced(path: string, onChange: () => void, debounceMs: number) {
  let t: NodeJS.Timeout | null = null;
  const w = watch(path, { persistent: true }, () => {
    if (t) clearTimeout(t);
    t = setTimeout(() => onChange(), debounceMs);
  });
  return () => w.close();
}
```

### `src/main/compositionRoot.ts`

Wire config store + reload loop.

```ts
import { ConfigStore } from "../config/store";
import { buildConfig } from "../config/loader";
import { watchFileDebounced } from "../infra/fsWatcher";
import type { AppConfig } from "../config/types";
import { redactConfigForLog } from "../config/redact";

export async function bootstrap() {
  const defaults = {
    env: "dev",
    server: { port: 3000, requestTimeoutMs: 30_000 },
    log: { level: "info", pretty: true },
    db: { url: "", poolSize: 10 },
    flags: { filePath: "./flags.json", ttlMs: 5000, killSwitchTtlMs: 250 },
    secrets: { jwtSecret: "" }
  };

  const deps = {
    readTextFile: (p: string) => import("node:fs/promises").then(m => m.readFile(p, "utf-8")),
    env: process.env,
    nowMs: () => Date.now(),
  };

  const filePath = process.env.APP_CONFIG_FILE ?? "./app.config.json";

  const initial = await buildConfig(deps, { defaults, filePath });
  if (!initial.parsed.ok) {
    // FAIL FAST on boot
    console.error(initial.parsed.issues);
    process.exit(1);
  }

  const store = new ConfigStore(initial.parsed.value);
  console.log({ config: redactConfigForLog(store.get()) }, "config.loaded");

  // RELOAD (keep last-known-good if invalid)
  const stop = watchFileDebounced(filePath, async () => {
    const next = await buildConfig(deps, { defaults, filePath });
    if (!next.parsed.ok) {
      store.emitInvalid(next.parsed.issues.map(i => ({ path: i.path, message: i.message })));
      return;
    }
    store.commit(next.parsed.value);
    console.log({ config: redactConfigForLog(store.get()) }, "config.reloaded");
  }, 200);

  return { store, stop };
}
```

---

## Tests (bắt buộc)

### 1) Precedence: `test/config.precedence.test.ts`

* defaults port 3000
* file sets port 4000
* env sets PORT=5000
* overrides sets port 6000
  ✅ expected: 6000

### 2) Schema: `test/config.schema.test.ts`

* env=prod, db.url empty → error `db.url is required`
* killSwitchTtlMs > ttlMs → error
* port out of range → error
* jwtSecret empty in prod → error (bạn nên thêm rule)

### 3) Reload: `test/config.reload.test.ts`

* start store with valid config
* simulate file change to new valid config → store.onChange fired once (debounce)
* config updated

### 4) Failure policy: `test/config.failurePolicy.test.ts`

* reload with invalid config → store.get() giữ nguyên (last-known-good)
* store.onInvalid fired with issues

---

## Checks (Definition of Done)

* ✅ Typed `AppConfig` chỉ xuất hiện sau parse schema.
* ✅ Layer precedence đúng 1→4.
* ✅ Boot invalid → fail fast.
* ✅ Reload invalid → keep last-good + emit invalid event.
* ✅ Secrets redact khi log.
* ✅ Tests cover precedence/schema/reload/failure policy.

---

## Stretch (Senior+ → rất cao)

1. **Hot reload consumers**
   FlagService subscribe store, tự update TTL/filePath ngay lập tức.
2. **Config diff**
   Khi reload, compute diff paths changed → log ngắn, không spam.
3. **Partial reload policy**
   Cho phép “non-critical sections” reload even if other sections invalid (rất khó, rất real-world).
4. **Immutable config snapshot**
   Freeze deep để tránh code mutate config.
5. **Multi-file layering**
   `base.json` + `env.json` + `local.json` + env vars.