# Kata 09 — Typed Event Map (Strongly-typed emitter)

## Goal

1. Định nghĩa `EventMap` (eventName → payload type)
2. Tạo `TypedEmitter<EventMap>` sao cho:

* `emit("user.created", payload)` payload đúng type
* `on("user.created", handler)` handler nhận đúng payload
* `off` / unsubscribe typed

3. Hỗ trợ “wildcard-ish” logging: `onAny((name, payload) => ...)` với payload typed theo name (không any)
4. Bonus: metadata (requestId, occurredAt) immutable + attach tự động (nối Kata 08.1)

---

## Constraints

* ❌ Không `any`
* ❌ Không dùng `EventEmitter` kiểu loose (nếu dùng, phải wrap lại typed)
* ✅ `emit` trả `void` (hoặc `Result` nếu bạn thích), nhưng type safety là trọng tâm
* ✅ Compile-time tests bằng `@ts-expect-error`

---

## Deliverables

File `kata09.ts` gồm:

1. `EventMap` sample (User/Order)
2. `TypedEmitter<M>`
3. `createEmitter<M>()`
4. Demo: emit/on/off + compile-time checks

---

## Event spec (fixed)

Bạn tạo event map như sau:

```ts
type Events = {
  "user.created": { userId: string; email: string };
  "user.renamed": { userId: string; oldName: string; newName: string };
  "order.created": { orderId: string; userId: string; amount: number };
  "order.cancelled": { orderId: string; reason: "user" | "fraud" };
};
```

Bonus metadata (optional nhưng recommended):

```ts
type EventMeta = {
  requestId?: string;
  occurredAt: number;
};
```

Event envelope:

```ts
type Envelope<Name extends keyof Events> = {
  name: Name;
  payload: Events[Name];
  meta: Readonly<EventMeta>;
};
```

---

## Tasks

### Task A — Typed `on` + `emit`

* `on<K extends keyof M>(name: K, handler: (payload: M[K]) => void): Unsub`
* `emit<K extends keyof M>(name: K, payload: M[K]): void`

### Task B — `once`

* `once(name, handler)` auto-unsubscribe after first call

### Task C — `onAny`

* `onAny(handler: <K extends keyof M>(name: K, payload: M[K]) => void): Unsub`

  * Handler phải được gọi cho mọi event
  * Không dùng `any`

### Task D — Envelope + meta attach (bonus)

* `emit` internally creates envelope `{name,payload,meta}`
* meta includes `occurredAt: Date.now()`
* if you have `getContext()` (Kata 08.1), attach `requestId` when available
* meta must be frozen/readonly

---

## Starter skeleton (điền TODO)

```ts
/* kata09.ts */

// ---------- Types ----------
export type Unsubscribe = () => void;

export type EventMap = Record<string, unknown>;

// optional: meta + envelope
export type EventMeta = {
  requestId?: string;
  occurredAt: number;
};

export type Envelope<M extends EventMap, K extends keyof M> = {
  name: K;
  payload: M[K];
  meta: Readonly<EventMeta>;
};

// ---------- TODO: TypedEmitter ----------
export interface TypedEmitter<M extends EventMap> {
  on<K extends keyof M>(name: K, handler: (payload: M[K]) => void): Unsubscribe;
  once<K extends keyof M>(name: K, handler: (payload: M[K]) => void): Unsubscribe;
  onAny(handler: <K extends keyof M>(name: K, payload: M[K]) => void): Unsubscribe;

  emit<K extends keyof M>(name: K, payload: M[K]): void;

  // bonus: observe envelope (useful for adapters / logging)
  onEnvelope(handler: <K extends keyof M>(env: Envelope<M, K>) => void): Unsubscribe;
}

// ---------- TODO: createEmitter ----------
export function createEmitter<M extends EventMap>(): TypedEmitter<M> {
  // You can implement with Maps:
  // - handlersByName: Map<keyof M, Set<(payload) => void>>
  // - anyHandlers: Set<(name,payload)=>void>
  // - envelopeHandlers: Set<(env)=>void>
  throw new Error("TODO");
}

// ---------- Demo event map ----------
type Events = {
  "user.created": { userId: string; email: string };
  "user.renamed": { userId: string; oldName: string; newName: string };
  "order.created": { orderId: string; userId: string; amount: number };
  "order.cancelled": { orderId: string; reason: "user" | "fraud" };
};

const bus = createEmitter<Events>();

// ---------- Compile-time checks ----------

bus.on("user.created", (p) => {
  p.userId;
  p.email;
  // @ts-expect-error - no amount on user.created
  p.amount;
});

// @ts-expect-error - wrong payload shape
bus.emit("user.created", { userId: "u1" });

// @ts-expect-error - wrong event name
bus.emit("user.deleted", { userId: "u1" });

bus.emit("order.cancelled", { orderId: "o1", reason: "fraud" });
// @ts-expect-error - invalid reason
bus.emit("order.cancelled", { orderId: "o1", reason: "ops" });

bus.onAny((name, payload) => {
  // name should be typed as keyof Events
  name;

  // payload type depends on name (generic)
  // example: narrow by if
  if (name === "order.created") {
    payload.amount;
    // @ts-expect-error - payload is order.created here, no email
    payload.email;
  }
});

bus.onEnvelope((env) => {
  env.meta.occurredAt;
  // meta is readonly
  // @ts-expect-error
  env.meta.occurredAt = 0;
});

// ---------- Runtime demo ----------
const off = bus.on("user.renamed", (p) => console.log("renamed:", p));
bus.emit("user.renamed", { userId: "u1", oldName: "A", newName: "B" });
off();
bus.emit("user.renamed", { userId: "u1", oldName: "A", newName: "C" }); // should not log

bus.once("user.created", (p) => console.log("once user.created:", p));
bus.emit("user.created", { userId: "u1", email: "x@y.com" });
bus.emit("user.created", { userId: "u2", email: "z@y.com" }); // should not log
```

---

## Required behaviors

* Type errors đúng như `@ts-expect-error`
* `once` chỉ gọi đúng 1 lần
* `off()` unsubscribe hoạt động
* `onAny` nhận event name typed và payload phụ thuộc theo name (bạn có thể narrow bằng `if`)

---

## Definition of Done

* Không `any`
* `createEmitter` không leak handler type (không ép kiểu bậy)
* Envelope meta readonly

---

## Stretch (rất production)

1. **Async handlers**: `handler` có thể return Promise, emitter có `emitAsync` await all + capture errors
2. **Backpressure**: limit in-flight handlers (nối Kata 41)
3. **Event versioning**: `"user.created.v1"` + compat utilities
4. **Context attach**: nếu bạn làm Kata 08.1, `emit` auto gắn `requestId`