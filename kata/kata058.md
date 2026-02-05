# Kata 58 — Correlation IDs qua Message Queue

*Propagate `correlation_id` / `trace_id` qua MQ (producer → broker → consumer), giữ context end-to-end, không leak/không cardinality bomb.*

---

## 1) Goal (năng lực cần đạt)

Sau kata này bạn làm được:

* Khi request vào API tạo `request_id` + `trace_id`, publish message vào queue và **consumer nhận lại đúng context**.
* Log ở cả producer & consumer đều có cùng `correlation_id` (và `trace_id` nếu có).
* Có **idempotency key** (tối thiểu) để at-least-once không phá hệ.
* Có **DLQ / poison message** simulation (nhẹ) và correlation vẫn giữ.
* Có **tests** (integration) chứng minh propagation + chain logging.

---

## 2) Context (bài toán)

Bạn build 2 process:

### Service A: `api`

* `POST /orders`

  * validate body
  * tạo `order_id`
  * publish message `OrderCreated` vào queue
  * trả `202 Accepted` + `correlation_id`

### Service B: `worker`

* consume `OrderCreated`

  * process (fake payment / reserve inventory)
  * publish `OrderProcessed` (optional) hoặc log result
  * nếu fail: retry N lần rồi DLQ

Queue: bạn chọn 1 (tuỳ tooling bạn thích):

* **RabbitMQ** (amqplib) — dễ làm correlation headers
* **Redis Streams** — cũng ok
* **SQS** — cần AWS; không phù hợp offline
  => Gợi ý: RabbitMQ + docker-compose.

---

## 3) Constraints (Senior+ bắt buộc)

1. Correlation ID **không được** là random mỗi log line:

   * must be **stable per flow**: request → message → consumer
2. Propagation phải qua **message metadata/headers**, không nhét vào body bừa bãi (body có thể, nhưng headers mới chuẩn).
3. Phải có **schema message envelope** chuẩn:

   * `id` (message id)
   * `type`
   * `occurred_at`
   * `payload`
   * `meta` (correlation/trace)
4. Correlation phải gồm tối thiểu:

   * `correlation_id` (stable)
   * `trace_id` (nếu có tracing)
   * `causation_id` (id của message trước đó) — optional nhưng Senior+ nên có
5. Không nổ cardinality:

   * Không metric label theo correlation_id
6. Có idempotency:

   * consumer phải xử lý at-least-once: ignore duplicates theo `message_id` hoặc `idempotency_key`
7. Có failure simulation:

   * poison message → retry → DLQ
   * correlation phải giữ nguyên xuyên retry & DLQ
8. Tests:

   * publish 1 message từ api, worker consume, assert correlation id xuất hiện ở cả hai log streams (hoặc stored).

---

## 4) Deliverables (repo)

```
kata-58/
├─ docker-compose.yml         (rabbitmq)
├─ packages/
│  ├─ api/
│  │  ├─ src/
│  │  │  ├─ server.ts
│  │  │  ├─ publish.ts
│  │  │  ├─ envelope.ts
│  │  │  ├─ correlation.ts
│  │  │  └─ logger.ts
│  │  └─ test/integration.spec.ts
│  ├─ worker/
│  │  ├─ src/
│  │  │  ├─ consume.ts
│  │  │  ├─ handlers.ts
│  │  │  ├─ envelope.ts
│  │  │  ├─ correlation.ts
│  │  │  ├─ idempotency.ts
│  │  │  └─ logger.ts
│  │  └─ test/integration.spec.ts
│  └─ shared/
│     └─ src/envelope.ts      (shared types)
└─ README.md                  (contract + scenarios)
```

---

## 5) Message Envelope Spec (bắt buộc)

Bạn define TypeScript type:

```ts
export type Envelope<TPayload> = {
  id: string;            // message_id (uuid)
  type: string;          // "OrderCreated"
  occurred_at: string;   // ISO
  payload: TPayload;
  meta: {
    correlation_id: string; // stable for a request-flow
    causation_id?: string;  // previous message id
    trace_id?: string;      // from tracing (W3C trace)
    request_id?: string;    // optional
    attempt?: number;       // retry attempt
  };
};
```

**Rule**:

* `correlation_id` do API tạo ra từ request context (hoặc derive từ trace_id).
* `id` là unique per message publish attempt (nhưng retries nên giữ `id` hay tạo mới? → chọn 1 và justify)

  * Gợi ý: giữ `id` **stable** cho logical message, attempt dùng `meta.attempt`.

---

## 6) MQ headers mapping (RabbitMQ)

Bạn phải set headers (minimum):

* `correlation_id`: same as envelope.meta.correlation_id
* `message_id`: envelope.id
* `type`: envelope.type
* `traceparent`: (nếu có) W3C trace context
* `causation_id`: envelope.meta.causation_id (nếu có)

Consumer phải:

* extract headers → rebuild `meta` context → set into AsyncLocalStorage / logger mixin.

---

## 7) Correlation context plumbing (must-have)

Tạo `correlation.ts` (cả 2 service):

* AsyncLocalStorage store:

  * `correlation_id`
  * `trace_id`
  * `request_id`
  * `message_id`

API flow:

* middleware tạo context cho mỗi request:

  * request_id = uuid
  * correlation_id = request_id (hoặc trace_id)
  * trace_id = from OTel active span (kata 52)
* logger mixin inject fields.

Worker flow:

* khi nhận msg:

  * extract headers
  * run handler inside ALS context
  * logger auto include.

---

## 8) Idempotency (consumer) — tối thiểu

`idempotency.ts`:

* In-memory set/map (cho kata) hoặc sqlite/redis (stretch)
* Key: `message_id`
* If seen → ack + skip (log `duplicate=true`)

**Rule**:

* Always do idempotency check **before** side effects.

---

## 9) Retry + DLQ simulation (nhẹ nhưng thật)

Bạn implement:

* max attempts = 3
* Nếu handler throw:

  * republish same message with `meta.attempt + 1`
  * keep same correlation_id
* After 3:

  * publish to DLQ queue `orders.dlq`
  * keep same correlation_id + include error summary (redacted)

---

## 10) Tests (bắt buộc)

### Test A — Propagation end-to-end

1. Start rabbitmq (compose)
2. Start worker consumer
3. Call API `POST /orders` with header `traceparent` (optional)
4. Assert:

   * API response includes `correlation_id`
   * Worker logs/records processed message with same `correlation_id`
   * If traceparent sent, trace_id preserved

Implementation tip for test:

* Worker viết result vào một in-memory “processed store” (exported) hoặc temporary file để test assert thay vì parse logs.

### Test B — Poison → Retry → DLQ

1. Send order with `payload.forceFail=true`
2. Expect:

   * attempts 1..3 happened
   * message ends in DLQ
   * correlation_id remains same across attempts & DLQ envelope

### Test C — Duplicate delivery

1. Publish same `message_id` twice (or replay)
2. Worker should process once, second time `duplicate=true` and no side effects.

---

## 11) Definition of Done (pass/fail)

PASS khi:

* Envelope schema chuẩn + headers mapping đúng.
* Correlation/trace propagate producer→consumer.
* Logs của cả 2 side đều contain same correlation_id.
* Retry + DLQ giữ correlation_id.
* Consumer idempotent theo message_id.
* Integration tests xanh (A + B + C).

---

## 12) Stretch (Senior++)

1. **Trace context injection/extraction** đúng OpenTelemetry propagators:

   * inject `traceparent` vào MQ headers, consumer extract và continue trace (span kind: CONSUMER).
2. **Outbox pattern** (kết hợp kata 33): API ghi DB + outbox rồi publisher.
3. **Schema versioning**: `meta.schema_version`, backward compatible.
4. **Security**: sign envelope meta (HMAC) chống spoof correlation_id.
5. **Observability**: metrics

   * `mq_consume_total{type,outcome}`
   * `mq_retry_total{type}`
   * nhưng tuyệt đối không label theo correlation_id.