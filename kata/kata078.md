# Kata 78 — Library Design: Publish Internal Package với SemVer

## Goal

1. Thiết kế một **internal package** dùng chung trong monorepo:

   * public API rõ, nhỏ, ổn định
   * không leak internals
   * docs + examples + typed contracts
2. Thiết lập **Semantic Versioning** + release workflow:

   * patch/minor/major đúng nghĩa
   * changelog + deprecation policy
   * compatibility tests (consumer contract)
3. Có **CI guardrails**:

   * check exports boundary
   * check breaking change (API extractor / dts diff đơn giản)
   * check version bump đúng loại

---

## Context (bài toán)

Bạn sẽ publish internal package: `@acme/typed-events`

Mục đích:

* cung cấp **TypedEventBus** (từ Kata 76) ở dạng library
* consumer packages: `transfer`, `notification`, `analytics`

Nhu cầu production:

* API phải ổn định, vì nhiều module phụ thuộc
* internal implementation có thể đổi (perf, queue, dlq) mà không bắt consumer sửa
* khi có breaking change phải buộc **major bump**

---

## Constraints

* ✅ Package phải có:

  * `exports` field (package.json) chỉ export public surface
  * `src/` + `dist/` build output
  * `README.md` usage + gotchas
  * `CHANGELOG.md` + versioning rules
* ✅ Không được export deep paths:

  * consumer không thể `import ... from "@acme/typed-events/internal/*"`
* ✅ Public API gồm đúng 3 entrypoints:

  1. `createEventBus<Events, Meta>(options)`
  2. `EventBus` interface/type
  3. `EventMap` helper types
* ✅ Phải có **deprecation policy**:

  * mark deprecated in types + runtime warn optional
  * maintain backward compatible in one minor cycle
* ✅ Phải có tests:

  * unit tests inside package
  * **consumer contract tests** (type-level) ở một package giả lập consumer
* ✅ Có “release simulation”:

  * v1.0.0 baseline
  * làm 3 changes: patch/minor/major, và bạn phải bump version đúng

---

## Deliverables (monorepo structure)

```
kata-78/
  packages/
    typed-events/                  # publishable lib
      package.json
      tsconfig.json
      src/
        index.ts
        types.ts
        bus.ts
        errors.ts
      test/
        bus.runtime.test.ts
        bus.types.test-d.ts        # or ts-expect-error in .ts
      README.md
      CHANGELOG.md
    consumer-a/                    # fake consumer to lock contract
      package.json
      tsconfig.json
      src/
        index.ts
      test/
        contract.types.test-d.ts
    consumer-b/                    # optional second consumer
  scripts/
    api-diff.ts                    # d.ts diff or public surface snapshot
    require-version-bump.ts        # validate semver bump vs change type
  pnpm-workspace.yaml (or npm workspaces)
```

> Bạn không cần publish thật lên npm. “Publish” ở đây là **đóng gói đúng chuẩn** và mô phỏng release.

---

## Public API spec (v1.0.0)

### `packages/typed-events/src/types.ts`

```ts
export type EventMap = Record<string, any>;

export type Handler<Payload, Meta> = (payload: Payload, meta: Meta) => void | Promise<void>;

export type EventBusOptions = {
  /** max parallelism across different keys (when using emitAsync) */
  concurrency?: number;
  /** called when handler throws/rejects */
  onError?: (e: { event: string; err: unknown }) => void;
};

export interface EventBus<Events extends EventMap, Meta extends Record<string, any>> {
  on<K extends keyof Events>(event: K, handler: Handler<Events[K], Meta>): () => void;

  /** fire-and-forget */
  emit<K extends keyof Events>(event: K, payload: Events[K], meta: Meta): void;

  /** awaits handlers; optional ordering if meta.key present */
  emitAndWait<K extends keyof Events>(event: K, payload: Events[K], meta: Meta): Promise<void>;
}
```

### `packages/typed-events/src/index.ts`

```ts
export type { EventBus, EventBusOptions, EventMap, Handler } from "./types";
export { createEventBus } from "./bus";
```

### `packages/typed-events/src/bus.ts`

```ts
import type { EventBus, EventBusOptions, EventMap, Handler } from "./types";

export function createEventBus<Events extends EventMap, Meta extends Record<string, any>>(
  opts: EventBusOptions = {}
): EventBus<Events, Meta> {
  // TODO: implement with private internals
  // Requirements:
  // - on() returns unsubscribe
  // - emit(): does not throw to caller (capture errors)
  // - emitAndWait(): await all handlers, but still isolate errors (calls onError)
  throw new Error("TODO");
}
```

---

## Packaging rules (must)

### `packages/typed-events/package.json`

Bạn phải có:

* `"name": "@acme/typed-events"`
* `"version": "1.0.0"`
* `"main"`, `"types"`, `"exports"`
* `"exports"` chỉ cho phép `.`

Ví dụ:

```json
{
  "name": "@acme/typed-events",
  "version": "1.0.0",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    }
  }
}
```

> Check: consumer không import được `@acme/typed-events/bus` hay `.../internal/...`.

---

## Tests (bắt buộc)

### 1) Runtime tests (package): `packages/typed-events/test/bus.runtime.test.ts`

* `on` + `emit` calls handler
* unsubscribe works
* handler throw không làm `emit` throw
* `emitAndWait` awaits async handler completion

### 2) Type tests (package): `packages/typed-events/test/bus.types.test-d.ts`

* Define Events map:

  * `"a.created": { id: string }`
* Ensure:

  * handler payload typed
  * emit wrong payload fails compile (`@ts-expect-error`)

### 3) Consumer contract tests: `packages/consumer-a/test/contract.types.test-d.ts`

* consumer uses library exactly như docs
* if you break public API, consumer types test must fail

### 4) Export boundary test (script)

* script tries to import deep path and expects failure at build time

  * simplest: grep in consumer code disallow `@acme/typed-events/`
  * better: node resolution check (optional)

---

## SemVer simulation tasks (phần “luyện release”)

Bạn phải làm 3 commits giả lập (không cần git thật, nhưng checklist phải đúng):

### Change A (PATCH): `1.0.0 -> 1.0.1`

* Fix bug: `unsubscribe` previously didn’t remove handler
* ✅ no type changes, no behavior change except bug fix
* update CHANGELOG patch entry

### Change B (MINOR): `1.0.1 -> 1.1.0`

* Add optional API (backward compatible):

  * `once(event, handler)` convenience
* ✅ existing code compiles unchanged
* update CHANGELOG minor entry

### Change C (MAJOR): `1.1.0 -> 2.0.0`

* Breaking change:

  * rename `emitAndWait` -> `publish`
  * OR change `onError` signature
* ✅ consumer contract tests should fail until you update consumers
* update CHANGELOG major entry + migration notes

---

## Guardrails (Senior+ “khóa luật”)

### Script 1: `scripts/api-diff.ts` (public surface snapshot)

* build d.ts (tsc)
* read `packages/typed-events/dist/index.d.ts`
* compare with previous snapshot file `api-snapshots/typed-events@<version>.d.ts`
* if changed and version not bumped appropriately → fail

> Bạn có thể đơn giản hóa: store snapshot string in repo.

### Script 2: `scripts/require-version-bump.ts`

Input:

* old version, new version
* change type: patch/minor/major (manual for kata)
  Rules:
* patch: public d.ts unchanged (or only doc comments)
* minor: only additive changes (no removals/renames)
* major: anything else

Bạn không cần viết perfect AST diff; dùng heuristics:

* patch: file text equals (after removing whitespace/comments)
* minor: old surface is subset of new surface (string containment blocks)
* major: otherwise

---

## README (must-have)

Trong `packages/typed-events/README.md` phải có:

* Quickstart code snippet
* Error semantics (“emit never throws; use onError”)
* Ordering semantics (if meta.key exists, emitAndWait orders per key — nếu bạn implement)
* Compatibility promise: SemVer policy

---

## Checks (Definition of Done)

* ✅ `@acme/typed-events` có exports boundary đúng (only `.`)
* ✅ Consumer packages compile & tests pass
* ✅ Runtime + type tests pass
* ✅ CHANGELOG + version bumps đúng cho patch/minor/major
* ✅ Có guardrail script phát hiện mismatch (ở mức tối thiểu)

---

## Stretch (Senior+ → rất cao)

1. **Dual build (CJS + ESM)** + exports conditions
2. **Tree-shaking safe** + sideEffects false
3. **Performance budget**: microbench for emit throughput
4. **API Extractor style**: generate report of public API, fail on breaking changes
5. **Deprecation runtime warnings** behind env flag