# Kata 15 — Backpressure Basics (Stream pipeline đúng, không OOM)

Mục tiêu của kata này: bạn xử lý **luồng dữ liệu lớn** (file/log/events) theo kiểu streaming, **không đọc hết vào RAM**, và pipeline vẫn ổn khi downstream chậm (DB/API).

---

## Goal

1. Xây 1 pipeline Node.js stream: **Source → Transform → Sink**
2. Enforce backpressure:

* Source **không bơm vô hạn**
* Transform **không tạo buffer vô hạn**
* Sink **giả lập chậm** nhưng không làm OOM

3. Có **benchmark mini**: 1–5 triệu lines (tuỳ máy), RAM không phình theo N

---

### Constraints

* ❌ Không `readFileSync`, không `fs.readFile` toàn bộ
* ❌ Không `.on("data")` rồi tự accumulate array (đây là OOM trap)
* ✅ Dùng `stream/promises.pipeline` hoặc `Readable.from(asyncIterator)`
* ✅ Dùng `highWaterMark` hợp lý
* ✅ Có giới hạn concurrency khi gọi IO (DB/API) trong Transform

---

## Context

Bạn có file log rất lớn `input.ndjson` (mỗi dòng 1 JSON). Bạn cần:

* parse JSON
* filter/transform
* “ghi DB” (fake) chậm
* vẫn chạy ổn

---

## Deliverables

File `kata15.ts` (Node 18+), gồm:

1. Generator tạo NDJSON lớn (để tự test)
2. Pipeline streaming:

   * Readable: đọc file theo stream
   * Transform: split lines + parse JSON
   * Transform: map/filter + batch (optional)
   * Sink: fake DB writer chậm + concurrency limit
3. Demo đo:

* thời gian
* RSS memory (process.memoryUsage().rss)

---

## Spec dữ liệu

Mỗi dòng NDJSON:

```json
{"id":123,"type":"click","value":42,"ts":1700000000000}
```

Rule xử lý:

* chỉ giữ event `type === "click"`
* map thành `{ id, score }` với `score = value * 2`
* ghi “DB” theo batch 100 records (khuyến khích)

---

## Starter skeleton (điền TODO)

```ts
/* kata15.ts */
import fs from "node:fs";
import { pipeline } from "node:stream/promises";
import { Transform, Writable } from "node:stream";
import readline from "node:readline";
import path from "node:path";

type Event = { id: number; type: string; value: number; ts: number };
type Row = { id: number; score: number };

const INPUT = path.join(process.cwd(), "input.ndjson");

// ---------- 1) Generate big NDJSON (for testing) ----------
async function generateNdjson(file: string, lines: number) {
  const w = fs.createWriteStream(file, { encoding: "utf8" });

  for (let i = 0; i < lines; i++) {
    const e: Event = {
      id: i,
      type: i % 3 === 0 ? "click" : "view",
      value: i % 100,
      ts: Date.now(),
    };

    const ok = w.write(JSON.stringify(e) + "\n");
    if (!ok) {
      // backpressure on write stream
      await new Promise<void>((res) => w.once("drain", () => res()));
    }
  }

  await new Promise<void>((res, rej) => {
    w.end(() => res());
    w.on("error", rej);
  });
}

// ---------- 2) Transform: NDJSON lines -> objects (NO buffering whole file) ----------
function createNdjsonParser() {
  // TODO: implement a Transform that takes string chunks and emits Event objects
  // Tip: keep a "carry" string for partial lines across chunks.
  return new Transform({
    readableObjectMode: true,
    writableObjectMode: false,
    transform(_chunk, _enc, cb) {
      cb(new Error("TODO"));
    },
  });
}

// ---------- 3) Transform: filter/map to Row ----------
function createMapper() {
  return new Transform({
    readableObjectMode: true,
    writableObjectMode: true,
    transform(obj: unknown, _enc, cb) {
      // TODO: validate shape minimally (no any)
      // if click -> push Row; else skip
      cb(new Error("TODO"));
    },
  });
}

// ---------- 4) Transform: batch rows (size=100) ----------
function createBatcher(batchSize: number) {
  let buf: Row[] = [];
  return new Transform({
    readableObjectMode: true,
    writableObjectMode: true,
    transform(row: unknown, _enc, cb) {
      // TODO: push row into buf, when buf.length==batchSize emit buf and reset
      cb(new Error("TODO"));
    },
    flush(cb) {
      // TODO: emit remaining buf if not empty
      cb();
    },
  });
}

// ---------- 5) Sink: fake DB writer (slow) with concurrency limit ----------
function sleep(ms: number) {
  return new Promise<void>((r) => setTimeout(r, ms));
}

// Simple concurrency limiter without external libs
function createLimiter(max: number) {
  let inFlight = 0;
  const queue: (() => void)[] = [];

  const runNext = () => {
    if (inFlight >= max) return;
    const job = queue.shift();
    if (!job) return;
    inFlight++;
    job();
  };

  const schedule = async <T>(fn: () => Promise<T>): Promise<T> => {
    return await new Promise<T>((resolve, reject) => {
      queue.push(() => {
        fn()
          .then(resolve, reject)
          .finally(() => {
            inFlight--;
            runNext();
          });
      });
      runNext();
    });
  };

  return { schedule };
}

async function fakeDbWrite(batch: Row[]) {
  // simulate slow downstream
  await sleep(5);
  // pretend success
  return batch.length;
}

function createDbSink(concurrency: number) {
  const limiter = createLimiter(concurrency);
  let total = 0;

  return new Writable({
    objectMode: true,
    highWaterMark: 8, // important: keep small
    async write(chunk: unknown, _enc, cb) {
      // TODO:
      // - chunk must be Row[]
      // - schedule fakeDbWrite with limiter (await)
      // - update total
      cb(new Error("TODO"));
    },
    final(cb) {
      console.log("DB total rows written:", total);
      cb();
    },
  });
}

// ---------- 6) Run pipeline ----------
function logMem(label: string) {
  const rssMb = Math.round(process.memoryUsage().rss / 1024 / 1024);
  console.log(`[mem] ${label}: rss=${rssMb}MB`);
}

async function main() {
  if (!fs.existsSync(INPUT)) {
    console.log("Generating input...");
    await generateNdjson(INPUT, 1_000_000);
  }

  logMem("before");

  const read = fs.createReadStream(INPUT, {
    encoding: "utf8",
    highWaterMark: 64 * 1024, // tune
  });

  const parser = createNdjsonParser();
  const mapper = createMapper();
  const batcher = createBatcher(100);
  const sink = createDbSink(8);

  const t0 = Date.now();
  await pipeline(read, parser, mapper, batcher, sink);
  const dt = Date.now() - t0;

  logMem("after");
  console.log("done in ms:", dt);
}

main().catch((e) => {
  console.error("FAILED:", e);
  process.exitCode = 1;
});
```

---

## Tasks bạn phải làm (cụ thể)

### Task A — Parser không OOM

Implement `createNdjsonParser()` đúng:

* giữ `carry` string
* split theo `\n`
* parse JSON từng dòng
* emit object từng event (`this.push(event)`)

**Không được** đọc toàn file vào 1 string.

### Task B — Mapper type-safe

Implement `createMapper()`:

* check `obj` là record có `type/value/id`
* nếu `type !== "click"` → skip
* emit `{ id, score }`

### Task C — Batcher + flush

* emit `Row[]` đúng size
* flush phần còn lại cuối stream

### Task D — Sink có backpressure thật

Implement `createDbSink()`:

* nhận chunk là `Row[]`
* `await limiter.schedule(() => fakeDbWrite(batch))`
* **chỉ gọi `cb()` sau khi write xong** (đây là chỗ tạo backpressure)

---

## Required behaviors (phải đạt)

1. Chạy với 1,000,000 lines mà **RSS không tăng tuyến tính** theo N
2. Không có mảng “giữ hết events” trong RAM
3. Downstream chậm vẫn ổn (db sleep 5ms)
4. Không “MaxListenersExceededWarning” / không leak listeners

---

## Definition of Done

* Pipeline dùng `pipeline(...)` (auto teardown khi error)
* Backpressure đúng: sink chậm ⇒ upstream tự chậm lại
* highWaterMark được set (không để default bừa)
* Concurrency được giới hạn (8) ⇒ không tạo “exploding promises”

---

## Stretch (Senior+ thật sự)

1. **Abort end-to-end**: add `AbortController` để cancel pipeline giữa chừng (nối Kata 14)
2. **Error classification**: JSON parse fail là operational; bug là programmer (nối Kata 11)
3. **Batching theo time**: batch 100 hoặc 50ms, cái nào đến trước
4. **Metrics**: throughput lines/s, lag, inFlight, queue length
5. **Stream ↔ async iterator**: implement version `for await (const line of rl)` và so với Transform