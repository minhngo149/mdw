# Contract Evidence

How to collect, weigh, cite, and present the evidence behind every alignment claim. It covers the
contract sheet you build per boundary, the finding record, the plain-text diagrams, and both
artifacts.

## Contents

1. Evidence priority
2. Evidence references
3. The contract sheet
4. Transformations and mappings
5. Shared types
6. Tests
7. Documentation drift
8. Finding record
9. Plain-text contract diagrams
10. Markdown artifact: section contents
11. HTML artifact: rules and skeleton

## 1. Evidence priority

```text
Source code > Tests > Configuration > Generated schemas / contracts > Documentation > Comments > Assumptions
```

| Source | Supports |
|---|---|
| Production code on the path that crosses the boundary | `CONFIRMED` |
| Tests, fixtures, sample payloads | what the author expected; `CONFIRMED` only for what the test asserts, never for runtime behavior |
| In-repo config (routes, topics, bindings, base URLs, naming strategy) | `CONFIRMED` as the in-repo value; deployment can override it |
| Generated code and schemas (`.proto`, OpenAPI, GraphQL SDL, Avro) | `CONFIRMED` for what the generated code does; a spec the code does not compile against is documentation |
| README, ADRs, OpenAPI descriptions, wikis, comments | leads to verify; drift when code disagrees |
| Names of types, fields, packages, services | leads only |

## 2. Evidence references

```text
path/to/file.ext:START-END Symbol     when you read those exact lines
path/to/file.ext Symbol               otherwise
```

- Paths are relative to the repository root, with forward slashes.
- Symbols: Go `Type.Method` or `Func`; Java/Kotlin/C# `Class#method`; JS/TS `Class.method` or
  `functionName`; Python `module.function`; config `config/app.yaml payments.routing_key`; struct
  fields `pkg/events/types.go InvoiceIssued.DueAt`.
- Take line numbers only from what you read (the Read tool's line prefixes, or `grep -n`). Never
  estimate them. When unsure, cite file + symbol.
- Every contract comparison cites **both** sides. A finding that cites only one side is at most
  `INFERRED`.

## 3. The contract sheet

Build one sheet per boundary before comparing. It is your working record and the source for the
per-category sections of the Markdown artifact.

```text
BOUNDARY  <id>  <producer> -> <consumer>  <PROTOCOL> <SYNC|ASYNC>  <route | exchange+key | topic | subject | url>
PAIRING   <how the two sides are linked, with evidence>                                        <LABEL>
PRODUCER  <file> <Symbol> (emit/respond site)   serializer: <type + tags/strategy>
CONSUMER  <file> <Symbol> (receive site)        decoder:    <type + tags/strategy>
MAPPING   <file> <Symbol> | none found (searched: <where>)
ELEMENT   <name>  producer=<actual>  consumer=<expected>  -> <STATUS>  <LABEL>  <evidence both sides>
...
ROLLUP    <STATUS>  (<reason>)
```

Fill one `ELEMENT` line per compared element: routing identifier, method/path, each field the
consumer reads, each state value, each error. The element names come from the category
references (`api-contract.md`, `event-contract.md`, `data-contract.md`, `state-contract.md`,
`error-contract.md`).

## 4. Transformations and mappings

Before reporting a mismatch, search the path between the wire and the consumer's use for a
conversion, and record where you searched:

- mapper, adapter, translator, anti-corruption layer, DTO constructor (`toDomain`, `fromDTO`,
  `mapStatus`)
- custom unmarshalling (`UnmarshalJSON`, `@JsonCreator`, pydantic validators, `@Transform`)
- serializer configuration (global naming strategy, enum serialization, date format)
- gateway, ingress, or proxy rewrites defined in the repository
- outbox relay, enrichment step, or republisher that rebuilds the payload
- upcaster or version adapter

If a mapping exists, compare producer to mapping input, and mapping output to consumer use. A
mapping that covers only some values is `PARTIALLY ALIGNED`; list the uncovered values.

A mapping is `CONFIRMED` only when it is on the consumer's actual path: called from the handler,
not merely present in the package.

## 5. Shared types

Both sides importing one type (a shared DTO package, a generated client, one `.proto`) is
evidence of intended agreement on field names and types. It does not prove:

- that both sides use it: a side may decode into a local struct instead. Check the decode site
- the same serializer settings on both sides
- the same version: two deployables can build against different revisions of a shared module
- routing: a shared payload type says nothing about whether the message is delivered
- meaning: both sides can agree on `int64 amount` and disagree on the unit

Record a shared type as `CONFIRMED` evidence for the elements it covers, and still check the list
above.

## 6. Tests

Tests show what the author expected. Compare every fixture and mock on either side with the
**producer's real output**:

- a consumer test fixture that uses field names, values, or units the producer never sends is
  **test drift**: the test passes and runtime fails. Report it as its own drift item and cite it
  in the related finding
- a producer test that asserts a payload is `CONFIRMED` for that assertion only
- a contract test (Pact, schema compatibility test, shared golden file used by both sides)
  directly supports alignment for what it checks. Say which side runs it
- when a mismatch has no test that would catch it, name the missing test in the finding:
  producer/consumer contract test, event schema test, API compatibility test, enum compatibility
  test, serialization round-trip test, backward-compatibility test, state-transition contract
  test. Do not write the test unless asked

## 7. Documentation drift

When an OpenAPI file, README, ADR, or comment describes a contract the code does not implement:

- the code is the contract; compare producer code with consumer code
- record the documentation claim as **documentation drift**: `Claim | Source | What the code does |
  Evidence`
- if one side's code follows the drifted documentation, say so in the finding: it explains how
  the mismatch arose and which side the team probably believes is right
- a boundary where both sides' code agrees and only documentation differs is `ALIGNED`, with a
  drift item

## 8. Finding record

Every finding uses this record, in this order. `Actual Contract` is what the producer really
emits; `Expected Contract` is what the consumer really expects.

```text
ID:                      CA-<n>
Finding:                 <one line naming the mismatch>
Category:                API | Event | Data | State | Error Contract
Status:                  MISALIGNED | PARTIALLY ALIGNED | UNKNOWN
Severity:                CRITICAL | HIGH | MEDIUM | LOW | OBSERVATION
Producer:                <component> (<deployable evidence>)
Consumer:                <component> (<deployable evidence>)
Boundary:                <PROTOCOL> <identifier>
Actual Contract:         <what the producer emits>                  <file> <Symbol>
Expected Contract:       <what the consumer expects>                <file> <Symbol>
Mismatch:                <structural | semantic | both>: <what differs, and whether a mapping was searched for>
Evidence:                <references for both sides, mapping search, tests, docs>
Impact:                  <what happens at runtime, for which inputs; what fails silently>
Confidence:              CONFIRMED | INFERRED (because ...) | UNKNOWN (resolve by ...)
How to Verify:           <a concrete check: a test to run, a message to replay, a log or row to inspect>
Recommended Resolution:  <options tied to this mismatch: align producer, align consumer, add mapping, accept both, version>
Trade-offs:              <who else is affected by each option; compatibility during rollout>
```

Rules:

- One finding per root cause. A renamed field that breaks three consumers is one finding with
  three consumers, or three findings if their impact or resolution differs.
- `Impact` describes this repository's behavior. Do not claim business consequences the code
  does not show ("customers are double charged") unless a confirmed path produces them.
- `Recommended Resolution` offers options; it does not choose an architecture. Name the
  compatibility step when both sides cannot deploy at once (accept both values, then remove the
  old one).
- An `OBSERVATION` has no mismatch today. Use it for fragile alignment (a mapping with no test, a
  lenient decoder hiding unused fields) and keep the record short.

## 9. Plain-text contract diagrams

ASCII only. No Mermaid, no images. Two forms:

**Path form**, for routing and pairing:

```text
[billing]  (services/billing)
   |  RABBITMQ publish: exchange billing (topic), key invoice.issued
   v
[RabbitMQ]
   :  binding invoice.created  (queue ledger.invoices)
   v
[ledger]  (services/ledger)
   x  never delivered: key invoice.issued does not match binding invoice.created   MISALIGNED
```

**Side-by-side form**, for payloads, states, and errors:

```text
Producer contract (billing)       Consumer expectation (ledger)      Alignment
--------------------------------  ---------------------------------  -------------------
invoice_id   string               invoiceId    string                MISALIGNED (name)
due_at       Unix milliseconds    due_at       Unix seconds          MISALIGNED (unit)
status       ISSUED | VOID        status       ISSUED | CANCELLED    PARTIALLY ALIGNED
total_minor  int64 minor units    total_minor  int64 minor units     ALIGNED
```

Keep each diagram within 100 columns. Draw one diagram per boundary or question, not one
diagram of everything. Mark non-`CONFIRMED` elements inline: `(INFERRED)`, `(?)`.

## 10. Markdown artifact: section contents

Keep every heading, in order. When a section does not apply, write one line saying so and why.
Every table that holds claims has a `Label` column.

| Section | Contents |
|---|---|
| Title | `# Contract Alignment Analysis: <scope>` |
| 1. Scope | Metadata block (repository, commit, analysis date, scope, `Generated by MDW contract-alignment`). What is in scope; boundaries found but excluded, each with a one-line reason. A one-paragraph summary. **Key findings**: 3-7 bullets, most severe first. Counts: boundaries by status, findings by severity, claims by label. **Handed to other skills**: one line per concern that belongs to another MDW skill (`distributed-flow`, `sequence-flow`, `database-performance`, `security`), naming the skill, with no analysis; or "None". |
| 2. Services / Components | `Component \| Kind \| Role in scope (producer / consumer / both) \| Deployable evidence \| Label`. Libraries (shared DTO or event packages) are listed as `library`, never as services. |
| 3. Boundaries | `# \| Boundary \| Protocol \| Mode \| Identifier (route, exchange + key, topic, subject, URL) \| Evidence \| Label` |
| 4. Producer / Consumer Map | Path-form diagram(s), then `Producer \| Boundary \| Consumer \| Pairing evidence \| Label`. Producers without an in-repo consumer and consumers without an in-repo producer are listed with `UNKNOWN`. |
| 5. Contract Matrix | `Producer \| Consumer \| Boundary \| Contract \| Status \| Findings`. One row per boundary and contract category that applies. No scores. |
| 6. API Contract Alignment | Per API boundary: the element table `Element \| Producer (actual) \| Consumer (expected) \| Mapping \| Status \| Label \| Evidence`, then a side-by-side diagram for any misaligned element. |
| 7. Event Contract Alignment | Per event boundary: routing first, then envelope, headers, and payload, in the same table form. |
| 8. Data Contract Alignment | Cross-cutting field semantics (IDs, money, time, enums, nullability, defaults) across all boundaries in scope, with the unit evidence for both sides. |
| 9. State Contract Alignment | Per boundary that carries states: the state mapping table and transition assumptions from `state-contract.md`. |
| 10. Error Contract Alignment | Per boundary: the error matrix from `error-contract.md`. |
| 11. Contract Drift | `Drift \| Earlier contract \| Current producer \| Consumer still expects \| Evidence \| Label`. Subsections **Test drift** (fixtures vs producer output) and **Documentation drift** (`Claim \| Source \| What the code does \| Evidence`). Include backward-compatibility implications only for drift that was found. |
| 12. Alignment Findings | Finding records from section 8, ordered by severity, then a short **Contract test gaps** list (`Finding \| Missing test \| Where it would live`). |
| 13. Evidence | Grouped by file: file, symbols (line ranges only if read), and what each proves. |
| 14. Confirmed / Inferred / Unknown | Three lists: **Confirmed**, **Inferred** (each with its reasoning), **Unknowns** (each with how to resolve it). |
| 15. Open Questions | Only the unknowns that change a status or severity: `Question \| Why it matters \| Where the answer likely lives`. |

## 11. HTML artifact: rules and skeleton

Rules:

- Derived from the Markdown: the same sections in the same order, the same claims, nothing added
  or dropped. Section ids: `scope`, `components`, `boundaries`, `map`, `matrix`, `api`, `event`,
  `data`, `state`, `error`, `drift`, `findings`, `evidence`, `confidence`, `open-questions`.
- One self-contained file: one inline `<style>`, no `<link>`, no `<script src>`, no web fonts or
  CDN, no `<img>`, `<svg>`, or `<canvas>`. Opens from disk and is fully readable with JavaScript
  disabled. Use `<details>` for long evidence lists; add no script unless a feature cannot work
  without it, and then keep it inline and small.
- Text diagrams go in `<pre class="diagram">`. The producer/consumer map may also use the
  `ol.map` rows below.
- Status pills: `<span class="st aligned">ALIGNED</span>`, and `partial` (PARTIALLY ALIGNED),
  `misaligned`, `unknown`. Severity: `<span class="sev critical">CRITICAL</span>`, and `high`,
  `medium`, `low`, `observation`. Evidence labels: `<span class="label confirmed">CONFIRMED</span>`,
  and `inferred`, `unknown`.
- Each finding is an `<article class="finding">` with the actual and expected contracts side by
  side in `div.versus`.
- Escape `<`, `>`, and `&` in code, diagrams, and table cells.

Skeleton (fill every section; repeat patterns as needed):

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Invoicing Contract Alignment - MDW contract-alignment</title>
<style>
  :root {
    --bg: #fafaf9; --fg: #1d1f21; --muted: #5f6368; --line: #d9dcdf; --card: #fff; --code: #f2f3f4;
    --accent: #2457a6;
    --ok: #1e6b37; --ok-bg: #e3f2e7; --part: #7a5200; --part-bg: #fbf0d0;
    --bad: #9b1c22; --bad-bg: #fbe4e4; --unk: #4a5058; --unk-bg: #e8eaed;
    --mono: ui-monospace, SFMono-Regular, Menlo, Consolas, "Liberation Mono", monospace;
    --sans: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", sans-serif;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --bg: #151719; --fg: #e3e5e8; --muted: #9aa0a6; --line: #33373c; --card: #1c1f23; --code: #22262a;
      --accent: #8ab4f8;
      --ok: #81c995; --ok-bg: #1d3124; --part: #f5cf66; --part-bg: #362d17;
      --bad: #f28b82; --bad-bg: #3a1d1d; --unk: #c4c7cc; --unk-bg: #2c3035;
    }
  }
  * { box-sizing: border-box; }
  body { margin: 0; background: var(--bg); color: var(--fg); font: 15px/1.55 var(--sans); }
  main { max-width: 1120px; margin: 0 auto; padding: 24px 16px 64px; }
  h1 { font-size: 1.6rem; margin: 0 0 8px; }
  h2 { font-size: 1.2rem; margin: 40px 0 12px; padding-top: 12px; border-top: 1px solid var(--line); }
  h3 { font-size: 1rem; margin: 20px 0 8px; }
  a { color: var(--accent); }
  .meta { display: flex; flex-wrap: wrap; gap: 4px 20px; color: var(--muted); font-size: .85rem; }
  nav.toc ol { columns: 2; margin: 12px 0 0; padding-left: 20px; font-size: .9rem; }
  code, pre { font-family: var(--mono); font-size: .85em; }
  code { background: var(--code); padding: 1px 4px; border-radius: 3px; overflow-wrap: anywhere; }
  pre { background: var(--code); border: 1px solid var(--line); border-radius: 6px; padding: 12px;
        overflow-x: auto; line-height: 1.35; }
  pre code { background: none; padding: 0; }
  .table-wrap { overflow-x: auto; margin: 8px 0 16px; }
  table { border-collapse: collapse; width: 100%; font-size: .87rem; }
  th, td { border: 1px solid var(--line); padding: 6px 8px; text-align: left; vertical-align: top; }
  th { background: var(--code); font-weight: 600; white-space: nowrap; }
  .st, .sev, .label { display: inline-block; font: 600 .7rem/1.7 var(--mono); padding: 0 6px;
                      border-radius: 3px; white-space: nowrap; }
  .st.aligned { color: var(--ok); background: var(--ok-bg); }
  .st.partial { color: var(--part); background: var(--part-bg); }
  .st.misaligned { color: var(--bad); background: var(--bad-bg); }
  .st.unknown { color: var(--unk); background: var(--unk-bg); }
  .sev { border: 1px solid currentColor; }
  .sev.critical, .sev.high { color: var(--bad); }
  .sev.medium { color: var(--part); }
  .sev.low, .sev.observation { color: var(--muted); }
  .label { border: 1px dashed currentColor; }
  .label.confirmed { color: var(--ok); } .label.inferred { color: var(--part); } .label.unknown { color: var(--unk); }
  /* Producer / consumer map: one row per pairing */
  ol.map { list-style: none; margin: 8px 0; padding: 0; display: grid; gap: 8px; }
  ol.map li { display: grid; grid-template-columns: 1fr minmax(160px, 1.2fr) 1fr max-content; gap: 8px;
              align-items: center; background: var(--card); border: 1px solid var(--line);
              border-radius: 6px; padding: 8px 12px; }
  ol.map .via { font: .8rem/1.4 var(--mono); color: var(--muted); border-top: 2px solid var(--muted);
                padding-top: 4px; text-align: center; }
  ol.map .via.async { border-top-style: dashed; }
  /* Findings */
  article.finding { background: var(--card); border: 1px solid var(--line); border-left: 4px solid var(--bad);
                    border-radius: 6px; padding: 12px 16px; margin: 12px 0; }
  article.finding.partial { border-left-color: var(--part); }
  article.finding.unknown, article.finding.observation { border-left-color: var(--unk); }
  article.finding h3 { margin: 0 0 6px; }
  .versus { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; margin: 8px 0; }
  .versus > div { background: var(--code); border-radius: 6px; padding: 8px 10px; }
  .versus h4 { margin: 0 0 4px; font-size: .75rem; text-transform: uppercase; color: var(--muted); }
  dl.record { display: grid; grid-template-columns: max-content 1fr; gap: 4px 12px; margin: 8px 0 0; }
  dl.record dt { font-weight: 600; color: var(--muted); }
  dl.record dd { margin: 0; }
  .unknowns { border: 1px dashed var(--line); border-radius: 6px; padding: 8px 16px; }
  @media (max-width: 720px) {
    ol.map li, .versus { grid-template-columns: 1fr; }
    nav.toc ol { columns: 1; }
  }
  @media print { body { background: #fff; } pre, article.finding { break-inside: avoid; } }
</style>
</head>
<body>
<main>
  <header>
    <h1>Contract Alignment Analysis: invoicing</h1>
    <div class="meta">
      <span>Repository: <code>billing-platform</code></span>
      <span>Commit: <code>9a41c0e</code></span>
      <span>Analyzed: 2026-01-15</span>
      <span>Generated by MDW contract-alignment</span>
    </div>
    <nav class="toc"><ol>
      <li><a href="#scope">Scope</a></li>
      <!-- one item per section, in Markdown order -->
    </ol></nav>
  </header>

  <section id="map">
    <h2>4. Producer / Consumer Map</h2>
    <ol class="map">
      <li><strong>billing</strong>
        <span class="via async">RABBITMQ invoice.issued</span>
        <strong>ledger</strong>
        <span class="st misaligned">MISALIGNED</span></li>
    </ol>
  </section>

  <section id="matrix">
    <h2>5. Contract Matrix</h2>
    <div class="table-wrap"><table>
      <thead><tr><th>Producer</th><th>Consumer</th><th>Boundary</th><th>Contract</th><th>Status</th><th>Findings</th></tr></thead>
      <tbody><tr><td>billing</td><td>ledger</td><td>RabbitMQ invoice.issued</td><td>Event</td>
        <td><span class="st misaligned">MISALIGNED</span></td><td><a href="#CA-1">CA-1</a></td></tr></tbody>
    </table></div>
  </section>

  <section id="findings">
    <h2>12. Alignment Findings</h2>
    <article class="finding" id="CA-1">
      <h3>CA-1 Routing key renamed; ledger binding still uses the old key</h3>
      <span class="sev critical">CRITICAL</span> <span class="st misaligned">MISALIGNED</span>
      <span class="label confirmed">CONFIRMED</span>
      <div class="versus">
        <div><h4>Actual (producer)</h4><code>key invoice.issued</code><br><code>services/billing/events.go Publish</code></div>
        <div><h4>Expected (consumer)</h4><code>binding invoice.created</code><br><code>services/ledger/consumer.go Bind</code></div>
      </div>
      <dl class="record">
        <dt>Category</dt><dd>Event Contract</dd>
        <dt>Impact</dt><dd>ledger never receives issued invoices; no error on either side.</dd>
        <!-- remaining fields of the finding record, in order -->
      </dl>
    </article>
  </section>

  <section id="confidence">
    <h2>14. Confirmed / Inferred / Unknown</h2>
    <div class="unknowns"><h3>Unknowns</h3><ul><li>...</li></ul></div>
  </section>
</main>
</body>
</html>
```
