# Kata 61 — Secrets Handling: load, rotate, never log

## Goal

Bạn sẽ xây một **secrets subsystem** có các tính chất production:

1. **Config load có schema** (không “đọc process.env rải rác”).
2. **Secrets không bao giờ bị log** (kể cả khi dev lỡ tay log config).
3. **Rotate secrets** (DB password / API key) **không restart service**.
4. **Fail-safe**: thiếu/invalid secrets → **không boot** (hoặc degrade rõ ràng theo policy).
5. **Auditability**: metrics/logs có thể cho biết “đã rotate”, “đang dùng phiên bản nào” **mà không lộ secret**.

---

## Context (Production pain)

Bạn có service gọi downstream:

* Database (password rotate theo lịch)
* External API (API key rotate / revoked)

Các lỗi thường gặp:

* Log vô tình in `process.env` hoặc dump config => **leak**
* Rotate xong mà service vẫn giữ secret cũ trong memory => **auth fail**
* Reload secret giữa request => inconsistent, race
* “Config file” bị commit nhầm

Bạn sẽ làm kata này như một **module độc lập** có thể drop vào bất kỳ service nào.

---

## Constraints (ép trade-off đúng)

* ❌ Không được đọc `process.env.MY_SECRET` rải rác ngoài module config/secrets.
* ❌ Không được log raw secret dưới bất kỳ hình thức nào.
* ❌ Không stringify toàn bộ config object ra log.
* ✅ Có **schema validation** cho config (zod/valibot…).
* ✅ Secrets phải được **redact** nếu ai đó log nhầm object.
* ✅ Rotation phải **atomic**: tại mọi thời điểm, consumer thấy **either old hoặc new**, không “half-half”.
* ✅ Có unit test + ít nhất 1 integration-ish test (fake secret provider + rotate).
* ✅ Có “break glass” mode cho local dev nhưng **không bao giờ default**.

---

## Deliverables (artifact bạn phải nộp)

Tạo folder `kata-61/` với:

* `README.md` (Goal / Policy / Run)
* `src/config.ts` — load config + schema + safe logging view
* `src/secrets/`

  * `provider.ts` — SecretProvider interface
  * `envProvider.ts` — đọc từ env (local/dev)
  * `fileProvider.ts` — đọc từ file (docker/k8s secret mount)
  * `mockProvider.ts` — dùng cho test + rotate simulation
  * `store.ts` — SecretsStore (atomic, rotate, watch)
* `src/httpClient.ts` — ví dụ consumer dùng secret (API key) với snapshot
* `src/logger.ts` — logger + redaction
* `test/kata61.test.ts`
* `docker-compose.yml` (optional) — nếu bạn muốn mô phỏng secret file mount

Tooling gợi ý: `vitest`, `tsx`, `zod`, `pino` (hoặc tương đương).

---

## Spec: Secret model & policies

### Secrets cần quản lý

* `DB_PASSWORD`
* `EXTERNAL_API_KEY`

### Rotation requirements

* Provider có thể trả về `(value, version)`:

  * `version` là string/number (vd: timestamp hoặc “v3”)
* Store refresh mỗi `N` giây (poll) hoặc watch file (nếu bạn thích).
* Khi rotate:

  * update phải **atomic** (consumer lấy snapshot một lần rồi dùng).
  * log event “secret rotated” chỉ ghi **key name + version**, không ghi value.

### Logging policy

* Logger phải hỗ trợ **redaction keys**: `DB_PASSWORD`, `EXTERNAL_API_KEY`, `Authorization`, `cookie`, …
* Nếu ai đó log `config` object, secret fields phải hiện `"[REDACTED]"`.
* Nếu error có message chứa secret (edge case), bạn phải có **sanitize** cho error string (stretch).

---

## Starter skeleton (điền TODO)

### `src/secrets/provider.ts`

```ts
export type SecretKey = "DB_PASSWORD" | "EXTERNAL_API_KEY";

export type SecretValue = {
  value: string;
  version: string; // e.g. "2026-02-05T00:00:00Z" or "v3"
};

export interface SecretProvider {
  /** Load one secret. Must throw on hard failure (e.g., IO error). */
  load(key: SecretKey): Promise<SecretValue>;

  /** Optional: load many to reduce IO round trips */
  loadMany?(keys: SecretKey[]): Promise<Record<SecretKey, SecretValue>>;
}
```

### `src/secrets/store.ts`

```ts
import type { SecretKey, SecretProvider, SecretValue } from "./provider";

export type SecretsSnapshot = Readonly<Record<SecretKey, SecretValue>>;

export type RotationEvent = {
  key: SecretKey;
  oldVersion: string;
  newVersion: string;
  at: number;
};

export type SecretsStoreOptions = {
  keys: readonly SecretKey[];
  refreshMs: number; // e.g. 10_000
  /** failFast: if true, missing secret prevents boot */
  failFast: boolean;
  onRotate?: (evt: RotationEvent) => void;
};

export class SecretsStore {
  private current: SecretsSnapshot | null = null;
  private timer: NodeJS.Timeout | null = null;

  constructor(
    private readonly provider: SecretProvider,
    private readonly opts: SecretsStoreOptions,
  ) {}

  /** Boot-time load. Must be called before getSnapshot(). */
  async init(): Promise<void> {
    // TODO: load all keys
    // TODO: set this.current atomically
    // TODO: start refresh loop (setInterval) if refreshMs > 0
  }

  /** Stop refresh loop. */
  shutdown(): void {
    // TODO
  }

  /** Consumer calls this once per operation to avoid mid-flight rotation issues. */
  getSnapshot(): SecretsSnapshot {
    // TODO: throw if not initialized
    // TODO: return immutable snapshot
    throw new Error("TODO");
  }

  /** Force refresh now (used by tests / admin endpoint). */
  async refreshNow(): Promise<void> {
    // TODO: load and rotate atomically
    // TODO: call onRotate for each changed key (compare versions)
  }
}
```

### `src/config.ts`

```ts
import { z } from "zod";

const ConfigSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),

  // secrets policy
  SECRETS_REFRESH_MS: z.coerce.number().int().min(0).default(10_000),
  SECRETS_SOURCE: z.enum(["env", "file"]).default("env"),
  SECRETS_DIR: z.string().optional(), // required if source=file

  // example downstream
  EXTERNAL_API_BASE_URL: z.string().url(),
});

export type AppConfig = z.infer<typeof ConfigSchema>;

/** IMPORTANT: must not include secrets. Only non-secret config. */
export function loadConfig(env: NodeJS.ProcessEnv): AppConfig {
  const parsed = ConfigSchema.safeParse(env);
  if (!parsed.success) {
    // TODO: throw error with safe formatting (no env dump)
    throw new Error("Invalid config");
  }
  return parsed.data;
}

/** Safe view for logging (no secrets present anyway) */
export function configForLog(cfg: AppConfig) {
  return cfg; // still keep this helper as a convention boundary
}
```

### `src/logger.ts`

```ts
import pino from "pino";

const REDACT_PATHS = [
  "DB_PASSWORD",
  "EXTERNAL_API_KEY",
  "headers.authorization",
  "headers.cookie",
  "req.headers.authorization",
  "req.headers.cookie",
];

export const logger = pino({
  // TODO: redact config - pino has redact option
  redact: {
    paths: REDACT_PATHS,
    censor: "[REDACTED]",
    remove: false,
  },
});
```

### `src/secrets/envProvider.ts`

```ts
import type { SecretKey, SecretProvider, SecretValue } from "./provider";

export class EnvSecretProvider implements SecretProvider {
  constructor(private readonly env: NodeJS.ProcessEnv) {}

  async load(key: SecretKey): Promise<SecretValue> {
    // TODO: read env[key]
    // TODO: validate non-empty
    // TODO: version - you can use env[`${key}_VERSION`] or "env"
    throw new Error("TODO");
  }
}
```

### `src/secrets/fileProvider.ts`

```ts
import { promises as fs } from "node:fs";
import path from "node:path";
import type { SecretKey, SecretProvider, SecretValue } from "./provider";

/**
 * Convention:
 *  - value at <dir>/<KEY>
 *  - version at <dir>/<KEY>.version (optional)
 */
export class FileSecretProvider implements SecretProvider {
  constructor(private readonly dir: string) {}

  async load(key: SecretKey): Promise<SecretValue> {
    // TODO: read file content trim()
    // TODO: read version file if exists else use mtimeMs or "file"
    throw new Error("TODO");
  }
}
```

### `src/secrets/mockProvider.ts` (for tests + rotate simulation)

```ts
import type { SecretKey, SecretProvider, SecretValue } from "./provider";

export class MockSecretProvider implements SecretProvider {
  private data: Record<SecretKey, SecretValue>;

  constructor(initial: Record<SecretKey, SecretValue>) {
    this.data = { ...initial };
  }

  async load(key: SecretKey): Promise<SecretValue> {
    const v = this.data[key];
    if (!v) throw new Error(`missing secret: ${key}`);
    return v;
  }

  rotate(key: SecretKey, next: SecretValue) {
    this.data = { ...this.data, [key]: next };
  }
}
```

### Example consumer: `src/httpClient.ts`

```ts
import type { SecretsStore } from "./secrets/store";

export async function callExternalApi(store: SecretsStore) {
  const snap = store.getSnapshot(); // snapshot once
  const apiKey = snap.EXTERNAL_API_KEY.value;

  // TODO: build request with Authorization header
  // NOTE: do not log apiKey
  return apiKey.length; // placeholder
}
```

---

## Tests (vitest) — tối thiểu phải có

### `test/kata61.test.ts` (gợi ý checklist)

* **Boot fail-fast**

  * thiếu `DB_PASSWORD` => `init()` throws (khi `failFast=true`)
* **No logging leak**

  * log object có field `EXTERNAL_API_KEY` => output phải có `"[REDACTED]"` (bạn có thể capture stream)
* **Rotation atomic + observable**

  * init với version v1
  * rotate provider sang v2
  * `refreshNow()` cập nhật snapshot
  * `onRotate` được gọi với `{oldVersion:"v1", newVersion:"v2"}`
  * snapshot trước/giữa/sau không bị null/partial
* **Snapshot consistency**

  * trong lúc refresh, consumer gọi `getSnapshot()` nhiều lần => luôn nhận object đầy đủ keys

---

## Checks (Definition of Done)

Bạn pass kata nếu:

1. **Không có** chỗ nào ngoài `secrets/` đọc `process.env.DB_PASSWORD` / `EXTERNAL_API_KEY`.
2. Logger redaction hoạt động (test chứng minh).
3. `SecretsStore`:

   * `init()` load đủ keys theo policy
   * `getSnapshot()` luôn trả snapshot immutable, đầy đủ keys
   * `refreshNow()` rotate atomically, gọi `onRotate` đúng
4. Có unit tests cover boot + rotation + redaction.
5. README mô tả rõ:

   * secrets source (env/file)
   * rotation model
   * những thứ “tuyệt đối không làm” (dump env/config)

---

## Stretch (Senior+ hơn nữa, bắt đầu “Ops-grade”)

1. **Hot reload với Abort semantics**
   Khi rotate API key, những request mới dùng key mới; request cũ vẫn xài snapshot cũ (đã làm).
   *Nâng cấp:* nếu key bị revoke, bạn có policy **invalidate immediately**: request mới bị chặn cho đến khi refresh thành công.

2. **Two-phase rotation for DB password**
   Mô phỏng “DB accepts both old+new” trong một window.

   * store giữ `active` + `previous` trong snapshot
   * client retry with previous nếu auth fail (giới hạn retry budget)

3. **Secret access audit counter (no values)**
   Mỗi lần consumer lấy secret key, tăng counter in-memory (per key) → expose metrics.
   *Mục tiêu:* detect code path “spam secret access”.

4. **Sanitize error strings**
   Nếu upstream trả error có dính secret (hiếm nhưng có), bạn viết `sanitizeErrorMessage(msg)` mask pattern.

5. **CI guardrail**
   Thêm test/grep rule fail build nếu có `process.env.DB_PASSWORD` outside `src/secrets/**`.

6. **K8s-style file watch**
   Thay poll bằng `fs.watch` + debounce, vẫn phải atomic.