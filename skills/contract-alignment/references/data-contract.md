# Data Contract

Compare what each shared field **means** on both sides of a boundary, and what each side's
serializer actually puts on and takes off the wire. This reference applies to every boundary
kind: API bodies, event payloads, webhook payloads, and shared messages.

## Contents

1. Wire shape comes from the serializer, not the type
2. Decoder behavior that hides mismatches
3. Field-by-field comparison
4. Establishing a unit or meaning from code
5. Structural vs semantic mismatch
6. Evidence to collect
7. Common false positives

## 1. Wire shape comes from the serializer, not the type

The contract is the bytes that cross the boundary. For each side, find what turns the in-memory
value into those bytes, or back:

- struct tags and annotations: Go `json:"..."`, Jackson `@JsonProperty`, `@JsonNaming`,
  pydantic `Field(alias=...)`, serde `#[serde(rename_all = ...)]`, `@SerializedName`
- global naming strategy: `spring.jackson.property-naming-strategy`, a custom `ObjectMapper`,
  an axios or fetch interceptor that converts keys to or from camelCase, a Rails serializer
- custom marshalling: `MarshalJSON`/`UnmarshalJSON`, `@JsonSerialize`, `toJSON()`,
  pydantic validators, protobuf `json_name`
- omission rules: `omitempty`, `JsonInclude.Include.NON_NULL`, `exclude_none=True`
- the value actually assigned before serialization, and under which conditions (a pointer set
  only when a field is non-empty, a default applied only in one handler)

Two structs with identical field names can still produce different bytes, and two structs with
different names can produce identical bytes.

## 2. Decoder behavior that hides mismatches

Most mismatches compile and pass unit tests because the consumer's decoder is lenient. Know
what the consumer's decoder does with a missing, extra, or wrongly typed field before you rate
impact. Treat these as `INFERRED` library defaults unless the repository configures them; check
the configured decoder, not the library's defaults, when one is configured.

| Decoder | Missing field | Unknown field | Name matching | Wrong type |
|---|---|---|---|---|
| Go `encoding/json` | zero value, no error | ignored (error only with `DisallowUnknownFields`) | exact, then case-insensitive; `user_id` never matches `userId` | error for that field (e.g. `12.5` into `int64`); `null` into a non-pointer leaves the value unchanged |
| Jackson (plain `ObjectMapper`) | Java default (`null`, `0`, `false`) | error (`FAIL_ON_UNKNOWN_PROPERTIES` on) | case-sensitive | numbers coerce: a float into an `int` truncates by default |
| Jackson (Spring Boot auto-config) | as above | ignored (Spring Boot turns `FAIL_ON_UNKNOWN_PROPERTIES` off) | case-sensitive | as above |
| pydantic v2 | required field: validation error; optional: default | ignored (`extra="ignore"` default) | exact, or alias | lax coercion (`"42"` to `42`); a fractional float into `int` fails; a number into `datetime` is read as Unix seconds or milliseconds by magnitude |
| Python `json` / dict access | `KeyError` at `d["x"]`, `None` with `d.get("x")` | ignored | exact | no check until the value is used |
| JS/TS `JSON.parse` | `undefined` | ignored | exact | no check; TypeScript types do not exist at runtime unless a validator (zod, io-ts, class-validator) runs |
| Protobuf binary | default value (unset and zero are indistinguishable unless `optional` or a wrapper type) | kept as unknown fields | by field **number**, not name | a changed type or reused number decodes as garbage or fails |
| Protobuf JSON mapping | default value | error by default in most parsers (ignore options exist) | lowerCamelCase `json_name`; parsers also accept the proto name | `int64`/`uint64` are JSON **strings** |

Consequences to look for:

- A renamed field is read as a **zero value**, not an error, in Go, Jackson, and JS. The
  consumer "works" and silently uses `""`, `0`, `false`, or `undefined`.
- A missing field and a present zero value look the same to a Go or protobuf consumer.
- A Go `nil` slice marshals as `null`, an empty slice as `[]`. A JS consumer that calls
  `.length` or `.map` on the result fails on `null`.
- `omitempty` drops `false`, `0`, and `""`. If the consumer's default for the missing field is
  `true` or non-zero, a real `false` or `0` is misread.
- Integers above 2^53 lose precision in JavaScript. 64-bit IDs sent as JSON numbers to a JS
  consumer are an `INFERRED` risk once values can exceed that range.

## 3. Field-by-field comparison

Compare only the fields that cross this boundary. For each one, fill both sides:

| Aspect | Producer side | Consumer side |
|---|---|---|
| Wire name | key after serialization | key the decoder reads |
| Wire type | JSON type, proto type, header string | type the decoder expects |
| Presence | always, conditional (when?), never | required, optional, defaulted (to what?) |
| Nullability | can it be `null` or absent? when? | does the code dereference or index it without a check? |
| Unit / scale | minor or major units, seconds or milliseconds, percent or ratio | the unit the consumer computes with |
| Value set | every value the producer can assign | every value the consumer handles; what happens to the others |
| Meaning | what the value represents when emitted | what the consumer does because of it |

Field families that carry most semantic mismatches:

| Family | Compare |
|---|---|
| IDs | internal database ID vs public or external ID; UUID vs integer vs prefixed string (`ord_...`); tenant-scoped vs global uniqueness; which entity the ID names (order vs payment vs cart) |
| Money | minor vs major units; integer vs decimal vs float vs decimal string; currency sent or assumed; currency exponent (USD 2, JPY 0, KWD 3: a fixed `*100` is wrong for some currencies); rounding mode; gross vs net vs tax-inclusive |
| Time | Unix seconds vs milliseconds vs microseconds vs nanoseconds; ISO 8601 with or without offset; naive local time; date-only values and the timezone they are interpreted in; what the timestamp marks (`created_at` of which record? `expires_at` inclusive or exclusive?) |
| Enums | exact spelling and case; which values each side knows; what the consumer does with an unknown value (reject, drop, default, crash) |
| Booleans | `false` omitted by `omitempty`; strings `"true"`/`"1"`; tri-state (null means "unknown", not `false`) |
| Defaults | producer omits a field and the two sides apply different defaults (server defaults `country` to `US`, client assumes the store's country) |
| Collections | `null` vs `[]`; ordering guaranteed or not; server-side size caps the consumer does not know about |
| Pagination | 0- vs 1-based pages; cursor vs offset; `has_more` vs `next_cursor`; server page-size cap vs a client that assumes one page is everything |
| Metadata / maps | free-form maps whose keys one side reads by name (`metadata["tenant"]`): treat each read key as a field |

## 4. Establishing a unit or meaning from code

Units and meanings are rarely typed. Rank the evidence:

| Strength | Examples |
|---|---|
| Strong | the computation that produces or consumes the value: `time.Now().UnixMilli()`, `Date.now()` (ms), `time.time()` (s, float), `Instant.toEpochMilli()`, `time.Unix(x, 0)` (reads seconds), `new Date(x)` (reads ms), `amount * 100`, `decimal.Shift(2)`, `NUMERIC(12,2)` vs `BIGINT` column the value is stored in |
| Medium | a field or variable name with a unit (`amount_minor`, `expires_at_ms`, `timeoutSeconds`), a named constant, a formatter that prints the value (`fmt.Sprintf("%.2f", amount)`) |
| Weak, never enough alone | a comment, a README, an OpenAPI `description` |

A unit established only from strong evidence on both sides is `CONFIRMED`. A unit from a name or
formatter is `INFERRED`, with the name as the reasoning. A unit supported only by comments or
docs is `UNKNOWN` for code, and a documentation claim to check.

## 5. Structural vs semantic mismatch

| Kind | Test | Example |
|---|---|---|
| Structural | The bytes do not fit the consumer's decoder or field lookup | producer sends `invoice_id`, consumer reads `invoiceId`; producer sends `0.25`, consumer decodes into an integer |
| Semantic | The bytes fit, but the consumer computes with a different meaning | both use an integer `discount`; producer means basis points, consumer means percent. Both use `expires_at`; producer writes milliseconds, consumer reads seconds |

Report both kinds. A semantic mismatch is usually more severe: nothing fails, and wrong values
spread into storage and other systems. When one field has both kinds, report one finding that
names both effects.

## 6. Evidence to collect

For each compared field, collect:

- producer: the serialization type and tag, the assignment site, and the condition under which it
  is set or left empty
- consumer: the decode type and tag, **every use site** that depends on the value (comparison,
  arithmetic, lookup, dereference), and the handling of absent or unknown values
- any conversion in between (section 7)
- the unit or meaning evidence from section 4, on both sides

## 7. Common false positives

- **A conversion exists.** A mapper, adapter, DTO constructor, custom unmarshaller, or gateway
  transform between the wire and the use site. Search for it before reporting; if it exists,
  compare the wire to the conversion's input and the conversion's output to the use.
- **The consumer never reads the field.** Extra producer fields are not a mismatch. Mention them
  only when the consumer clearly intends to use the data but reads it from a different field.
- **Case-insensitive decoding.** In Go, `userID` and `userId` match; `user_id` and `userId`
  do not.
- **A shared naming strategy.** A global snake_case setting on one side can make a camelCase
  field name produce `user_id` on the wire. Check it before comparing names.
- **Both sides use the same value space under different names.** `amount_minor` and
  `amountCents` can be aligned. Compare units, not names.
- **Unreachable producer branches.** A value the producer declares but never assigns on any path
  cannot reach the consumer. Say so instead of reporting a missing case.
