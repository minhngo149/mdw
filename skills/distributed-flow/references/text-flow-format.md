# Text Flow Format

MDW's conventions for evidence references, the flow model, plain-text diagrams, and the two
artifacts. Every MDW flow uses the same conventions, so people and other MDW skills can read
any of them without a translation step.

All diagrams are plain text (ASCII) in Markdown and HTML/CSS in HTML. There is no Mermaid, no
SVG, no images, and no diagram files.

## Contents

1. Evidence references
2. Flow model
3. Text diagrams
4. Markdown artifact: section contents
5. HTML artifact: rules and skeleton

## 1. Evidence references

```text
path/to/file.ext:START-END Symbol     when you read those exact lines
path/to/file.ext Symbol               otherwise
```

- Paths are relative to the repository root and use forward slashes.
- Symbol forms: Go `Type.Method` or `Func`; Java, Kotlin, and C# `Class#method`; JS/TS
  `Class.method`, `functionName`, or `export POST`; Python `module.function` or
  `Class.method`; Ruby `Class#method`; SQL `migrations/003_bookings.sql bookings`; config
  `config/app.yaml payments.timeout`.
- Take line numbers only from what you read (the Read tool's line prefixes, or `grep -n`).
  Never estimate them.
- Labels are always spelled out in upper case: `CONFIRMED`, `INFERRED`, `UNKNOWN`. An
  `INFERRED` claim carries its reasoning ("because ..."). An `UNKNOWN` carries how to resolve it
  ("resolve by ...").

## 2. Flow model

The flow model is the compact, line-oriented record of the analysis. It goes at the end of the
Markdown artifact under `## Appendix: Flow Model`, in a `text` code block. Other MDW skills
parse it, so keep to the grammar.

```text
FLOW: <flow name>
SOURCE: <repository> @ <short commit sha | "uncommitted"> (<YYYY-MM-DD>)
ENTRY
  <PROTOCOL> <trigger>  =>  <file> <Symbol>                                  <LABEL>
ACTOR
  <who or what triggers the flow>
PARTICIPANTS
  <name>  <kind>  <runtime>  <file or manifest>                              <LABEL>
SYNC
  <caller> -> <receiver>  <PROTOCOL>  <purpose>                              <LABEL>
ASYNC
  <producer> ~> <broker/target> ~> <consumer>  <PROTOCOL>  <event/topic/queue>  <LABEL>
PERSISTENCE
  <component> -> <store>.<table>  <OPERATION>  <tx id | autocommit>          <LABEL>
CACHE
  <component> -> <store>  <OPERATION> <key pattern>  ttl=<value | none>      <LABEL>
TRANSACTIONS
  <tx id>  <file> <Symbol>: <operations inside, in order>                    <LABEL>
STATE
  <FROM> -> <TO>  by <Symbol or event>                                       <LABEL>
FAILURE
  <trigger> -> <behavior> -> <state after> -> <response>                     <LABEL>
DRIFT
  <documentation claim> (<doc file>)  vs  <what the code does>               <LABEL>
UNKNOWN
  <item>  (resolve by <how>)
```

Rules:

- One fact per line. `->` is a synchronous call or transition; `~>` is an asynchronous handoff;
  `=>` points from a trigger to its code.
- Participant `kind` is one of: process, service, worker, module, package, library, database,
  cache, broker, external system.
- Omit a block that has nothing in it, except `UNKNOWN`: always include it, with `none` if
  nothing is unknown.
- ASCII only, so the model reads the same in every terminal, font, and diff.

## 3. Text diagrams

| Notation | Meaning |
|---|---|
| `[Name]` | runtime participant: process, service, worker, database, cache, broker, or external system |
| indented `Symbol()` under a participant | in-process step inside that participant |
| `\|` then `v` | synchronous call, reading downward |
| `:` then `v` | asynchronous handoff, reading downward |
| `+-->` | synchronous branch |
| `+~~>` | asynchronous branch |
| `x-->` | failure exit: `x--> 409 Conflict` |
| `(?)` | unknown participant or hop, explained in the Unknown section |

- Put the edge label to the right of the connector, as `PROTOCOL MODE: purpose`.
- Keep a diagram to 80 columns and about 30 lines. Draw one diagram per question (the overall
  path, the async chain, a failure path) instead of one diagram that shows everything.
- Mark a hop that is not `CONFIRMED` inline: `(INFERRED)`, `(?)`.

Main flow:

```text
[Client]
   |  REST SYNC: POST /bookings
   v
[Booking API]  (cmd/api)
   RequireToken()          x--> 401 Unauthorized
   BookingHandler.Create() x--> 400 Bad Request
   BookingService.Reserve()
   |
   +--> [PostgreSQL]  DATABASE SYNC: T1 lock seats, insert booking HELD
   |
   +--> [PSP]  REST SYNC: charge card (8 s timeout, no retry)
   |
   +--> [PostgreSQL]  DATABASE SYNC: booking -> CONFIRMED (autocommit)
   |
   +~~> [Kafka]  KAFKA ASYNC: topic booking.confirmed
           :
           v
        [Ticket Worker]  (cmd/ticketer, group ticketer)
           |
           +--> [Object Storage]  OTHER_EXTERNAL_API SYNC: store ticket PDF
           |
           +~~> (?)  booking.confirmed.DLT consumer not in repository
```

State transitions:

```text
HELD --Confirm(): payment ok--> CONFIRMED
  |
  +--Confirm(): payment error--> FAILED
  |
  +--ExpireHolds cron: held > 15 min--> EXPIRED
```

Failure chains use the format defined in `failure-flow.md`.

## 4. Markdown artifact: section contents

Keep every heading, in order. When a section does not apply, write one line saying so and why.
Use tables wherever the data has repeated fields. Every table that holds claims has a `Label`
column.

| Section | Contents |
|---|---|
| Title | `# <Feature> Flow` |
| 1. Overview | A metadata block (repository, commit, analysis date, flow, `Generated by MDW distributed-flow`). Then scope: what is included and what is deliberately excluded. Then a summary of one short paragraph. Then **Key findings**: 3-7 bullets, most important first (risks, drift, stuck states). Then the counts of `CONFIRMED`, `INFERRED`, and `UNKNOWN` items. |
| 2. Entry Point | Trigger, actor, protocol, where it is registered (evidence), and the middleware, interceptor, or guard chain in execution order, with what each link can reject. |
| 3. Main Flow | The overall text diagram, then numbered steps: `# \| Where \| What happens \| Protocol / Mode \| Evidence \| Label`. |
| 4. Components | `Component \| Kind \| Runtime \| Responsibility in this flow \| Evidence \| Label`. Runtime participants first, then in-process components. |
| 5. Synchronous Flow | The boundary table for sync hops that leave the process: `# \| Caller \| Receiver \| Protocol \| Mode \| Purpose \| Timeout \| Retry \| Evidence \| Label`. Then fan-out or ordering notes, and the hazards from `sync-flow.md` that apply. |
| 6. Asynchronous Flow | One row per async hop, with the per-hop record fields from `async-flow.md`. Then delivery semantics, consumer idempotency, and eventual consistency. |
| 7. Persistence | `Component \| Store \| Table / Collection \| Operation \| Transaction \| Purpose \| Evidence \| Label`. Subsection **Cache**: `Component \| Store \| Operation \| Key pattern \| TTL \| Role \| Evidence \| Label`. Subsection **Concurrency controls**: locks, version checks, unique constraints, idempotency records, or the confirmed absence of each. |
| 8. State Changes | The state diagram, then `From \| To \| Trigger \| Guard \| Tx \| Evidence \| Label`. |
| 9. Failure Flow | The summary table `Failure \| Trigger \| Detection \| Behavior \| State after \| Response / recovery \| Label`, then detailed chains for the important failures. |
| 10. Transaction Boundary | Each database transaction as a `BEGIN ... COMMIT` block, with what runs outside it. Then the distributed business transaction: which pattern is present (with its evidence), or that none is, and the dual-write gaps. |
| 11. Evidence | Grouped by file: file, then symbols (with line ranges if read), then what each proves. |
| 12. Confirmed / Inferred / Unknown | Three lists, one item per line, each with evidence or reasoning. Then a subsection **Documentation drift**: `Claim \| Source \| What the code does \| Evidence`. |
| 13. Open Questions | Only the `UNKNOWN`s that matter: `Question \| Why it matters \| Where the answer likely lives` (a team, another repository, deployment config, the broker's admin console). |
| Appendix: Flow Model | The flow model from section 2 of this reference, in a `text` code block. |

## 5. HTML artifact: rules and skeleton

Rules:

- The HTML has the same sections, in the same order, with the same claims as the Markdown.
  Use these section ids: `overview`, `entry-point`, `main-flow`, `components`, `sync-flow`,
  `async-flow`, `persistence`, `state-changes`, `failure-flow`, `transaction-boundary`,
  `evidence`, `confidence`, `open-questions`, `flow-model`.
- Every text diagram from the Markdown appears in a `<pre class="diagram">`. The main flow may
  also be drawn with HTML/CSS flow blocks (`ol.flow`, shown below), with a solid connector for
  `SYNC` and a dashed connector for `ASYNC`.
- Labels are badges: `<span class="label confirmed">CONFIRMED</span>`, and likewise `inferred`
  and `unknown`. Modes are `<span class="mode">SYNC</span>` and `<span class="mode">ASYNC</span>`.
- The file is self-contained: one inline `<style>`. No `<link>` or `<script src>`, no web fonts,
  no CDN, no `<img>`, `<svg>`, or `<canvas>`.
- JavaScript is not needed. Use `<details>` for collapsible content. If you add any script, it
  must be inline and small, and every piece of content must be visible without it.
- Escape `<`, `>`, and `&` in code, diagrams, and table cells.

Skeleton (fill in every section; repeat the table and flow patterns as needed):

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Create Booking Flow - MDW distributed-flow</title>
<style>
  :root {
    --bg: #fafaf9; --fg: #1d1f21; --muted: #5f6368; --line: #d9dcdf; --card: #fff; --code: #f2f3f4;
    --accent: #2457a6; --ok: #1e6b37; --ok-bg: #e3f2e7; --inf: #7a5200; --inf-bg: #fbf0d0;
    --unk: #9b1c22; --unk-bg: #fbe4e4;
    --mono: ui-monospace, SFMono-Regular, Menlo, Consolas, "Liberation Mono", monospace;
    --sans: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", sans-serif;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --bg: #151719; --fg: #e3e5e8; --muted: #9aa0a6; --line: #33373c; --card: #1c1f23; --code: #22262a;
      --accent: #8ab4f8; --ok: #81c995; --ok-bg: #1d3124; --inf: #f5cf66; --inf-bg: #362d17;
      --unk: #f28b82; --unk-bg: #3a1d1d;
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
  code { background: var(--code); padding: 1px 4px; border-radius: 3px; }
  pre { background: var(--code); border: 1px solid var(--line); border-radius: 6px; padding: 12px;
        overflow-x: auto; line-height: 1.35; }
  pre code { background: none; padding: 0; }
  .table-wrap { overflow-x: auto; margin: 8px 0 16px; }
  table { border-collapse: collapse; width: 100%; font-size: .87rem; }
  th, td { border: 1px solid var(--line); padding: 6px 8px; text-align: left; vertical-align: top; }
  th { background: var(--code); font-weight: 600; white-space: nowrap; }
  .cards { display: grid; grid-template-columns: repeat(auto-fill, minmax(240px, 1fr)); gap: 12px; }
  .card { background: var(--card); border: 1px solid var(--line); border-radius: 6px; padding: 12px; }
  .card h3 { margin: 0 0 4px; }
  .kind { color: var(--muted); font-size: .8rem; }
  .label, .mode { display: inline-block; font: 600 .7rem/1.7 var(--mono); padding: 0 6px;
                  border-radius: 3px; white-space: nowrap; }
  .label.confirmed { color: var(--ok); background: var(--ok-bg); }
  .label.inferred { color: var(--inf); background: var(--inf-bg); }
  .label.unknown { color: var(--unk); background: var(--unk-bg); }
  .mode { border: 1px solid var(--line); color: var(--muted); }
  .findings li { margin-bottom: 4px; }
  /* Flow blocks: nodes joined by edges. Solid edge = SYNC, dashed edge = ASYNC. */
  ol.flow { list-style: none; margin: 8px 0; padding: 0; }
  ol.flow > li.node { background: var(--card); border: 1px solid var(--line);
                      border-left: 4px solid var(--accent); border-radius: 6px; padding: 8px 12px; }
  ol.flow > li.node ul { margin: 4px 0 0; padding-left: 18px; font: .82rem/1.5 var(--mono); }
  ol.flow > li.edge { position: relative; margin-left: 28px; padding: 8px 0 12px 16px;
                      border-left: 2px solid var(--muted); font: .8rem/1.4 var(--mono); color: var(--muted); }
  ol.flow > li.edge.async { border-left-style: dashed; }
  ol.flow > li.edge::after { content: ""; position: absolute; left: -7px; bottom: -1px;
                             border: 6px solid transparent; border-top-color: var(--muted); border-bottom: 0; }
  ol.flow ol.flow { margin-left: 28px; }
  .chain { display: grid; grid-template-columns: max-content 1fr max-content; gap: 4px 12px;
           background: var(--card); border: 1px solid var(--line); border-radius: 6px; padding: 10px 12px; }
  .chain dt { font-weight: 600; color: var(--muted); }
  .chain dd { margin: 0; }
  @media print { body { background: #fff; } pre, .card, .chain { break-inside: avoid; } }
</style>
</head>
<body>
<main>
  <header>
    <h1>Create Booking Flow</h1>
    <div class="meta">
      <span>Repository: <code>booking-platform</code></span>
      <span>Commit: <code>3f2c1ab</code></span>
      <span>Analyzed: 2026-01-15</span>
      <span>Generated by MDW distributed-flow</span>
    </div>
    <nav class="toc"><ol>
      <li><a href="#overview">Overview</a></li>
      <li><a href="#entry-point">Entry Point</a></li>
      <!-- one item per section, in Markdown order -->
    </ol></nav>
  </header>

  <section id="overview">
    <h2>1. Overview</h2>
    <p>Summary paragraph.</p>
    <h3>Key findings</h3>
    <ul class="findings">
      <li><span class="label inferred">INFERRED</span> A PSP timeout marks the booking FAILED
        although the charge may have succeeded.</li>
    </ul>
  </section>

  <section id="main-flow">
    <h2>3. Main Flow</h2>
    <pre class="diagram">[Client]
   |  REST SYNC: POST /bookings
   v
[Booking API]  (cmd/api)</pre>
    <ol class="flow">
      <li class="node"><strong>Client</strong> <span class="kind">external system</span></li>
      <li class="edge">REST <span class="mode">SYNC</span> POST /bookings</li>
      <li class="node"><strong>Booking API</strong> <span class="kind">service - cmd/api</span>
        <ul><li>BookingHandler.Create()</li><li>BookingService.Reserve()</li></ul>
      </li>
      <li class="edge async">KAFKA <span class="mode">ASYNC</span> booking.confirmed</li>
      <li class="node"><strong>Ticket Worker</strong> <span class="kind">worker - cmd/ticketer</span></li>
    </ol>
  </section>

  <section id="components">
    <h2>4. Components</h2>
    <div class="cards">
      <div class="card"><h3>Booking API</h3><div class="kind">service - cmd/api</div>
        <p>Accepts bookings, reserves seats, charges the PSP.</p>
        <code>cmd/api/main.go main</code> <span class="label confirmed">CONFIRMED</span></div>
    </div>
  </section>

  <section id="sync-flow">
    <h2>5. Synchronous Flow</h2>
    <div class="table-wrap"><table>
      <thead><tr><th>#</th><th>Caller</th><th>Receiver</th><th>Protocol</th><th>Mode</th><th>Purpose</th>
        <th>Timeout</th><th>Retry</th><th>Evidence</th><th>Label</th></tr></thead>
      <tbody><tr><td>2</td><td>Booking API</td><td>PSP</td><td>REST</td><td><span class="mode">SYNC</span></td>
        <td>charge card</td><td>8 s</td><td>none</td><td><code>internal/booking/psp.go Pay()</code></td>
        <td><span class="label confirmed">CONFIRMED</span></td></tr></tbody>
    </table></div>
  </section>

  <section id="failure-flow">
    <h2>9. Failure Flow</h2>
    <h3>PSP timeout during payment</h3>
    <dl class="chain">
      <dt>Trigger</dt><dd>POST /payments exceeds the 8 s timeout</dd><dd><span class="label confirmed">CONFIRMED</span></dd>
      <dt>State</dt><dd>status = FAILED; the charge may still have succeeded</dd><dd><span class="label inferred">INFERRED</span></dd>
      <dt>Recovery</dt><dd>no reconciliation found in repository</dd><dd><span class="label unknown">UNKNOWN</span></dd>
    </dl>
  </section>

  <section id="flow-model">
    <h2>Appendix: Flow Model</h2>
    <pre>FLOW: Create Booking
SOURCE: booking-platform @ 3f2c1ab (2026-01-15)</pre>
  </section>
</main>
</body>
</html>
```
