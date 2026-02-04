# Kata 05 — Parse Don’t Validate (UserProfile)

## Goal

1. Nhận input `unknown` (giả lập HTTP body / message queue)
2. **Parse** thành domain type **đã đúng** (typed + normalized)
3. Domain functions chỉ nhận domain type, không validate lại
4. Trả lỗi theo kiểu accumulate (dùng ValidationResult của Kata 04.2)

---

## Context

Bạn làm endpoint: `POST /user-profile`

Input payload (unknown) có thể bẩn:

* `email` có thể uppercase, có spaces
* `birthYear` có thể string `"2001"` hoặc number `2001`
* `tags` có thể `["a","b"]` hoặc `"a,b"`
* `marketingOptIn` có thể `"true" | true | 1 | "1"`

Bạn phải parse thành domain chuẩn, normalize & canonicalize.

---

## Constraints

* ❌ Không throw
* ❌ Không `any`
* ✅ Parse trả `ValidationResult<T>` (accumulate errors)
* ✅ Domain types dùng **branded types** khi hợp lý (Email, UserName, Tag, BirthYear)
* ✅ Không để `unknown` đi sâu vào domain

---

## Deliverables

File `kata05.ts` gồm:

1. `ValidationResult`, `combineV`, `mapV` (có thể copy từ 04.2)
2. Domain types + constructors/parsers:

   * `Email`, `UserName`, `BirthYear`, `Tags`, `MarketingOptIn`
3. `parseUserProfile(body: unknown): ValidationResult<UserProfile>`
4. 6–10 demo cases (console log hoặc tests)

---

## Domain spec

### Domain types

Bạn parse ra:

```ts
type UserProfile = {
  email: Email;
  name: UserName;
  birthYear?: BirthYear;
  tags: Tag[];               // default []
  marketingOptIn: boolean;   // default false
};
```

### Parsing rules

* `email` (required)

  * trim, lowercase
  * phải match regex email đơn giản (đủ dùng): `^[^\s@]+@[^\s@]+\.[^\s@]+$`
* `name` (required)

  * trim, collapse multiple spaces thành 1
  * length 2..50
* `birthYear` (optional)

  * chấp nhận number hoặc string number
  * integer, 1900..(currentYear-13) (giả sử user >= 13)
* `tags` (optional)

  * nếu array: phải là string[]
  * nếu string: split `,` và trim
  * normalize: lowercase, unique, remove empty
  * max 10 tags, mỗi tag length 1..20, regex: `^[a-z0-9-]+$`
  * default []
* `marketingOptIn` (optional)

  * accept boolean OR `"true"/"false"` OR `1/0` OR `"1"/"0"`
  * default false

---

## Starter skeleton (điền TODO)

```ts
/* kata05.ts */

// ---------- Validation core (copy from kata 04.2) ----------
export type ValidationError = { type: "validation"; field: string; message: string };

export type ValidationResult<T> =
  | { ok: true; value: T }
  | { ok: false; errors: ValidationError[] };

export const valid = <T>(value: T): ValidationResult<T> => ({ ok: true, value });
export const invalid = (...errors: ValidationError[]): ValidationResult<never> => ({
  ok: false,
  errors,
});

export function mapV<T, U>(vr: ValidationResult<T>, fn: (v: T) => U): ValidationResult<U> {
  if (!vr.ok) return vr;
  return valid(fn(vr.value));
}

export function combineV<T extends readonly ValidationResult<any>[]>(
  ...results: T
): ValidationResult<{ [K in keyof T]: T[K] extends ValidationResult<infer V> ? V : never }> {
  const errors: ValidationError[] = [];
  const values: unknown[] = [];

  for (const r of results) {
    if (r.ok) values.push(r.value);
    else errors.push(...r.errors);
  }

  return errors.length ? { ok: false, errors } : (valid(values as any) as any); // TODO: remove any
}

// ---------- Branding ----------
declare const __brand: unique symbol;
type Brand<T, Name extends string> = T & { readonly [__brand]: Name };

export type Email = Brand<string, "Email">;
export type UserName = Brand<string, "UserName">;
export type BirthYear = Brand<number, "BirthYear">;
export type Tag = Brand<string, "Tag">;

const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
const TAG_RE = /^[a-z0-9-]+$/;

// ---------- Parsing helpers ----------
function isRecord(x: unknown): x is Record<string, unknown> {
  return typeof x === "object" && x !== null;
}

// TODO: normalize spaces in name
function normalizeName(raw: string): string {
  // trim + collapse whitespace to single space
  return raw;
}

// ---------- Parsers for primitives -> domain ----------
export function parseEmail(x: unknown, field = "email"): ValidationResult<Email> {
  // TODO: required string, trim+lowercase, regex
  return invalid({ type: "validation", field, message: "TODO" });
}

export function parseUserName(x: unknown, field = "name"): ValidationResult<UserName> {
  // TODO: required string, normalizeName, length 2..50
  return invalid({ type: "validation", field, message: "TODO" });
}

export function parseBirthYear(x: unknown, field = "birthYear"): ValidationResult<BirthYear | undefined> {
  // TODO: optional; accept number or numeric string; integer; range 1900..currentYear-13
  return invalid({ type: "validation", field, message: "TODO" });
}

export function parseMarketingOptIn(x: unknown, field = "marketingOptIn"): ValidationResult<boolean> {
  // TODO: optional; accept boolean/"true"/"false"/1/0/"1"/"0"; default false
  return valid(false);
}

export function parseTags(x: unknown, field = "tags"): ValidationResult<Tag[]> {
  // TODO: optional; string[] or "a,b"; normalize/unique/max 10; validate each tag
  return valid([]);
}

// ---------- Domain ----------
export type UserProfile = {
  email: Email;
  name: UserName;
  birthYear?: BirthYear;
  tags: Tag[];
  marketingOptIn: boolean;
};

// ---------- Main parse ----------
export function parseUserProfile(body: unknown): ValidationResult<UserProfile> {
  if (!isRecord(body)) {
    return invalid({ type: "validation", field: "body", message: "must be an object" });
  }

  // TODO: parse independent fields with combineV, then mapV to build UserProfile
  // email + name required, birthYear optional, tags optional default [], marketingOptIn optional default false
  return invalid({ type: "validation", field: "TODO", message: "TODO" });
}

// ---------- Demo cases ----------
const cases: unknown[] = [
  { email: "  MAX@Example.com ", name: "  Max   Nguyen ", birthYear: "2001", tags: "Dev, TypeScript,dev", marketingOptIn: "1" },
  { email: "not-an-email", name: "M", birthYear: 1888, tags: ["ok", "bad tag"], marketingOptIn: "maybe" },
  "not-an-object",
  { name: "Max", tags: "a,b,c" }, // missing email
];

for (const c of cases) {
  console.log(JSON.stringify(parseUserProfile(c), null, 2));
}
```

> Trong `combineV` skeleton đang có `any` để “cắm cờ”. Nhiệm vụ của bạn là **xóa sạch any** bằng cách viết `combineV` typed đúng (y như kata 04.2) hoặc viết riêng `combineVObj` (stretch).

---

## Required behaviors (phải đạt)

Case 1 phải:

* email → lowercase, trimmed
* name → “Max Nguyen” (1 space)
* birthYear `"2001"` → number branded
* tags → `["dev","typescript"]` (unique, lowercase)
* marketingOptIn `"1"` → `true`

Case 2 phải trả nhiều lỗi (ít nhất):

* email invalid
* name too short
* birthYear out of range
* tags có tag invalid (`"bad tag"` fails regex)
* marketingOptIn invalid

---

## Checks (Definition of Done)

* `parseUserProfile` không throw
* Domain type `UserProfile` không chứa `unknown | string` bẩn cho email/name/year/tag
* Validation trả list lỗi đầy đủ và field path đúng
* Không `any`

---

## Stretch (đáng làm)

1. `combineVObj({ email:..., name:..., ... })` để build object không cần tuple
2. Field path nested: `"tags[1]"` khi tag thứ 2 sai
3. Trả về normalized output + warnings (vd: “tags truncated to 10”)