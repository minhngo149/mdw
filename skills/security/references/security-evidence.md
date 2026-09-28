# Security Evidence

MDW's conventions for security findings: the finding model, severity and confidence, false-positive
rules, secret redaction, attack-path text format, and the two artifacts. Every MDW security review
uses the same conventions, so people and other MDW skills can read any of them.

All attack paths are plain text (ASCII) in Markdown and HTML/CSS in HTML. There is no Mermaid, no
SVG, no images, and no diagram files.

## Contents

1. Evidence references
2. The finding model
3. Severity
4. Confidence
5. False-positive rules
6. Redaction
7. Attack-path format
8. Markdown artifact: section contents
9. HTML artifact: rules and skeleton

## 1. Evidence references

```text
path/to/file.ext:START-END Symbol     when you read those exact lines
path/to/file.ext Symbol               otherwise
```

- Paths are relative to the repository root, with forward slashes.
- Symbol forms: Go `Type.Method` or `Func`; Java/Kotlin/C# `Class#method`; JS/TS `Class.method` or
  `functionName`; Python `module.function`; Ruby `Class#method`; SQL `migrations/001.sql users`;
  config `config/app.yaml auth.secret`.
- Take line numbers only from what you read. Never estimate them.
- Labels are spelled out in upper case: `CONFIRMED`, `INFERRED`, `UNKNOWN`. An `INFERRED` claim
  carries its reasoning ("because ..."). An `UNKNOWN` carries how to resolve it ("resolve by ...").

## 2. The finding model

Every finding uses exactly these fields, in this order:

```text
Finding          short title of the risk
Category         Authentication | Authorization | Injection | Secrets | Session | Cryptography |
                 Data Protection | File Security | Configuration | Dependency | Business Logic | ...
Severity         CRITICAL | HIGH | MEDIUM | LOW | OBSERVATION, qualified for static risk
Location         file(s) + symbol(s)
Attack Surface   the entry point an attacker reaches (route, resolver, consumer, CLI arg, ...)
Evidence         what the code does and what control is missing, with references
Attack Path      plain-text path from actor to impact (section 7)
Security Impact  the concrete consequence, for the actors this repo grants
Confidence       CONFIRMED | INFERRED | UNKNOWN, with the reason
How to Verify    a safe, non-destructive test (see "Safe verification" below)
Recommended Fix  the change that closes the gap, at the right boundary
Trade-offs       cost of the fix: extra query, latency, compatibility, migration
```

Give each finding a stable id: `SEC-01`, `SEC-02`, ... ordered by severity, then confidence.

Worked example:

```text
Finding          Potential Broken Object-Level Authorization on order lookup
Category         Authorization
Severity         HIGH — static risk
Location         internal/order/handler.go Handler.Get; internal/order/repo.go Repo.FindByID
Attack Surface   GET /orders/{id} (requires a valid bearer token)
Evidence         Handler.Get parses {id} and calls Repo.FindByID(id). FindByID runs
                 SELECT ... FROM orders WHERE id = $1 with no user_id filter. No ownership or
                 permission check exists between lookup and response. (List filters by user_id,
                 which shows ownership is the intended rule.)
Attack Path      see section 7
Security Impact  Any authenticated user may read another user's order, including the shipping
                 address.
Confidence       CONFIRMED — missing ownership check in the traced path
How to Verify    In a local environment, create two test users A and B. As A, request B's order id.
                 Expect 404/403; a 200 with B's order confirms the issue. Add this as a test.
Recommended Fix  Enforce ownership at the service/repository boundary:
                 WHERE id = $1 AND user_id = $2, returning 404 when not owned.
Trade-offs       One extra predicate; returning 404 (not 403) avoids confirming the id exists.
```

## 3. Severity

| Level | Typical shape |
|---|---|
| `CRITICAL` | Unauthenticated or trivially reachable path to full compromise: auth bypass, RCE, admin takeover, mass data exposure, forgeable credentials with no preconditions |
| `HIGH` | Authenticated path to another user's data or privileges (IDOR, privilege escalation), injection reaching a datastore, SSRF with response read-back, predictable reset tokens, a live credential committed |
| `MEDIUM` | Real risk needing a precondition or giving limited impact: non-expiring tokens, secrets written to logs, race conditions with bounded loss, missing replay protection on non-idempotent actions |
| `LOW` | Hardening gap with small or conditional impact: verbose errors, non-constant-time compare, container as root with no other exposure |
| `OBSERVATION` | Not a vulnerability as analyzed, but worth knowing: defense-in-depth hardening, dev-only config, a control that would matter if the auth model changed |

Rate from exploitability, impact, exposure, required privileges, attack complexity, and affected
assets. Raise severity for chains (section 7): two `MEDIUM`s that combine into admin takeover make
the chain `HIGH` or `CRITICAL`; record the chain as its own finding or in each member's Attack Path.

**Static-risk qualifier.** This skill analyzes code, not a running system. When a severity rests on
static reading and runtime exposure is not proven, write `HIGH — static risk`. Use
`verified locally` only when you actually ran a safe local test that demonstrated the behavior.
Never write "exploited", "actively exploitable in production", or a measured impact you did not
measure.

## 4. Confidence

`CONFIRMED` / `INFERRED` / `UNKNOWN`, with the definitions in `SKILL.md`. Keep two questions apart:

- **Does the flaw exist?** A missing check, an unparameterized identifier, a hardcoded value. Often
  `CONFIRMED`.
- **Is it reachable and impactful at runtime?** Depends on deployment, network, and data. Often
  `INFERRED` or `UNKNOWN`.

Write both when they differ: "Confidence: CONFIRMED that `JWT_SECRET` falls back to a hardcoded
default; INFERRED exposure, because it only applies when the environment does not set the variable,
which the repo does not show for production." Never hide uncertainty.

## 5. False-positive rules

Do **not** report any of these as vulnerabilities by themselves:

```text
"uses HTTP"               "has environment variables"     "uses JWT"
"has database queries"    "has an admin endpoint"         "has POST/PUT/DELETE"
"makes outbound requests" "no rate limiter found"         "runs as root in a container"
```

Every finding must explain **what is wrong, why it matters, where it occurs, and what evidence
proves it**. Before keeping a candidate, check:

1. Does attacker-controlled data actually reach the dangerous sink on a reachable path?
2. Is there a safe API, parameterization, allowlist, or control that neutralizes it?
3. Does the impact exist for an actor this repository actually grants?
4. Is the code wired in (registered, called, not behind a disabled flag, not test-only)?

If a candidate fails any check, drop it or demote it to `OBSERVATION` with the reason. Demoted
items that a reader might expect to see listed (for example CORS reflection under bearer auth) go in
a short **Reviewed, not a vulnerability** list so the reasoning is visible.

## 6. Redaction

The artifacts must never contain a real password, API key, private key, access or refresh token,
session cookie, database password, cloud credential, webhook secret, or encryption key.

- Replace the value with `[REDACTED]`. When the type prefix is needed to classify a credential, show
  only the vendor's documented fixed prefix and mask the rest: `sk_live_****[REDACTED]`,
  `AKIA****[REDACTED]`. Never show characters from the random part, and never show the last four.
- In code excerpts, redact the literal: `getenv("STRIPE_SECRET_KEY", "[REDACTED]")`.
- Redaction also applies to the reply to the user, verification commands, and HTML. A verification
  step greps for the variable name or prefix, never the value.
- Record only what identifies location and type:

```text
Potential hardcoded API credential
  Location   internal/config/config.go Load (STRIPE_SECRET_KEY fallback)
  Type       payment provider secret key (live-mode prefix)
  Value      [REDACTED]
```

- A committed secret is compromised whether or not it is removed later: the fix is **rotate**, then
  remove, then prevent recurrence (secret manager, scanning in CI). See `secrets.md`.

## Safe verification

Every "How to Verify" is safe and non-destructive:

- Prefer unit tests, integration tests, an isolated local environment, test accounts, static
  inspection, and controlled requests with harmless test values.
- Use two test users you create to check authorization; never another real user's data.
- Never instruct the reader to attack production, use real credentials, destroy data, bypass real
  authorization on live systems, exfiltrate data, or run destructive payloads.
- For injection, verify with a benign marker (an unexpected column name that causes a harmless
  error, or an invalid sort value) against a local database, not a data-extracting payload.
- Where a finding cannot be verified safely without infrastructure the repo does not have, say so
  and mark the runtime impact `UNKNOWN`.

## 7. Attack-path format

| Notation | Meaning |
|---|---|
| `[Name]` | actor or runtime participant: attacker, API, service, database, external system |
| indented `Symbol()` under a participant | in-process step inside that participant |
| `\|` then `v` | the path continues, reading downward |
| `!!` | the gap: where a control is missing or bypassable |
| `ok` | a control that exists and holds on this path |
| `x-->` | a control that rejects: `x--> 401 Unauthorized` |
| `(?)` | an unknown step, explained in the Unknown section |

Mark each step's confidence at the end of the line when it is not `CONFIRMED`. Keep a path to 80
columns and about 25 lines. Draw one path per finding or chain.

```text
[Authenticated user A]  (any valid bearer token)
   |  REST: GET /orders/{id of user B's order}
   v
[API]  cmd/api
   RequireAuth()          ok  token signature valid
   Handler.Get()              parses {id} from the path
   Repo.FindByID(id)      !!  no ownership check; WHERE id = $1 only
   |
   +--> [PostgreSQL]  SELECT ... FROM orders WHERE id = $1
   |
   v
200 OK: user B's order returned to user A
```

Chain across findings:

```text
[Customer]
   |  REST: PUT /users/me  {"role": "admin"}
   v
[API]  UpdateMe()        !!  role bound from JSON into UPDATE        (SEC-02)
   |
   |  REST: POST /login  (fresh token now carries role=admin)
   v
[API]  RequireRole()     ok  checks claims.Role == "admin"  (trusts the token)
   |
   v
GET /admin/users: full user list returned to a former customer
```

## 8. Markdown artifact: section contents

Keep every heading, in order. When a section does not apply, write one line saying so and why.
Every table that holds claims has a `Label` column.

| Section | Contents |
|---|---|
| Title | `# <Scope> Security Analysis` |
| 1. Scope | A metadata block (repository, commit, analysis date, scope, `Generated by MDW security`). What is included and excluded, and the technologies present. Then **Key findings**: 3-7 bullets, most severe first. Then counts of findings by severity, and of `CONFIRMED` / `INFERRED` / `UNKNOWN`. |
| 2. Application Attack Surface | `Entry \| Authentication \| Authorization \| Input \| Sensitive operation \| Data access \| Evidence \| Label`. |
| 3. Trust Boundaries | A plain-text boundary diagram, then `Boundary \| Data crossing \| Validation \| Authorization \| Evidence \| Label`. Assets and actors (from the threat model) as short tables. |
| 4. Authentication | Mechanism, where identity is established, validation checks performed and missing. |
| 5. Authorization | Enforcement points, the model (RBAC/ABAC/ownership), per-operation checks, IDOR/BOLA, escalation, mass assignment, tenant isolation. |
| 6. Input / Output Security | Where input is validated and output is encoded; validation vs authorization. |
| 7. Injection Analysis | Each sink: `Sink \| Input source \| Safe API? \| Controllable part \| Verdict \| Evidence \| Label`. |
| 8. Session / Token Security | Cookies, CSRF, CORS, token lifecycle. |
| 9. Secrets | `Location \| Type \| Real or placeholder \| Value \| Label`, with `Value` always `[REDACTED]`. |
| 10. Data Protection | Sensitive data and its exposure paths; logging; errors; cryptography. |
| 11. File / Resource Security | Path handling, uploads, archives. |
| 12. External Integrations | Webhooks, brokers, third-party APIs, SSRF surface. |
| 13. Dependency / Configuration Security | Dependency table (`Package \| Version \| Status`), config and container findings. |
| 14. Business Logic Security | Logic flaws and race conditions with their concrete interleaving. |
| 15. Security Findings | Every finding in the model from section 2, ordered by severity. Then **Reviewed, not a vulnerability**. |
| 16. Safe Verification Plan | `Finding \| Test \| Environment \| Expected safe result`. |
| 17. Confirmed / Inferred / Unknown | Three lists, each item with evidence or reasoning. Then **Documentation drift**: `Claim \| Source \| What the code does \| Evidence`. |
| 18. Open Questions | Only the `UNKNOWN`s that matter: `Question \| Why it matters \| Where the answer likely lives`. |
| 19. Evidence | Grouped by file: file, symbols (with line ranges if read), what each proves. Secret values redacted. |

## 9. HTML artifact: rules and skeleton

Rules:

- Same sections, same order, same findings as the Markdown. Section ids: `scope`, `attack-surface`,
  `trust-boundaries`, `authentication`, `authorization`, `input-output`, `injection`, `session-token`,
  `secrets`, `data-protection`, `file-security`, `integrations`, `dependency-config`,
  `business-logic`, `findings`, `verification`, `confidence`, `open-questions`, `evidence`.
- Findings are cards (`div.finding`) with the severity as a colored left border and badge.
- Every attack path from the Markdown appears as HTML/CSS path blocks (`ol.path`) and may also appear
  in a `<pre class="diagram">`. The gap step uses `li.node.gap`.
- Severity badges: `<span class="sev critical">CRITICAL</span>`, and `high`, `medium`, `low`, `obs`.
  Confidence badges: `<span class="label confirmed">CONFIRMED</span>`, `inferred`, `unknown`.
- Self-contained: one inline `<style>`. No `<link>`, no `<script src>`, no web fonts, no CDN, no
  `<img>`, `<svg>`, or `<canvas>`. No JavaScript needed; use `<details>` for collapsible content.
- Escape `<`, `>`, and `&` in code, attack paths, and table cells. Every secret stays `[REDACTED]`.

Skeleton (fill in every section; repeat the patterns as needed):

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>shopapi Security Analysis - MDW security</title>
<style>
  :root {
    --bg: #fafaf9; --fg: #1d1f21; --muted: #5f6368; --line: #d9dcdf; --card: #fff; --code: #f2f3f4;
    --accent: #2457a6; --ok: #1e6b37; --ok-bg: #e3f2e7; --inf: #7a5200; --inf-bg: #fbf0d0;
    --unk: #9b1c22; --unk-bg: #fbe4e4;
    --crit: #7f1d1d; --high: #b42318; --med: #b54708; --low: #1d4ed8; --obs: #5f6368;
    --mono: ui-monospace, SFMono-Regular, Menlo, Consolas, "Liberation Mono", monospace;
    --sans: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", sans-serif;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --bg: #151719; --fg: #e3e5e8; --muted: #9aa0a6; --line: #33373c; --card: #1c1f23; --code: #22262a;
      --accent: #8ab4f8; --ok: #81c995; --ok-bg: #1d3124; --inf: #f5cf66; --inf-bg: #362d17;
      --unk: #f28b82; --unk-bg: #3a1d1d;
      --crit: #fca5a5; --high: #f97066; --med: #fdb022; --low: #93c5fd; --obs: #9aa0a6;
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
  .counts { display: flex; flex-wrap: wrap; gap: 8px; margin: 8px 0; }
  .label, .sev { display: inline-block; font: 600 .7rem/1.7 var(--mono); padding: 0 6px;
                 border-radius: 3px; white-space: nowrap; }
  .label.confirmed { color: var(--ok); background: var(--ok-bg); }
  .label.inferred { color: var(--inf); background: var(--inf-bg); }
  .label.unknown { color: var(--unk); background: var(--unk-bg); }
  .sev { color: #fff; }
  .sev.critical { background: var(--crit); } .sev.high { background: var(--high); }
  .sev.medium { background: var(--med); } .sev.low { background: var(--low); }
  .sev.obs { background: var(--obs); }
  .finding { background: var(--card); border: 1px solid var(--line); border-left: 5px solid var(--obs);
             border-radius: 6px; padding: 12px 16px; margin: 12px 0; }
  .finding.critical { border-left-color: var(--crit); } .finding.high { border-left-color: var(--high); }
  .finding.medium { border-left-color: var(--med); } .finding.low { border-left-color: var(--low); }
  .finding h3 { margin: 0 0 8px; }
  .finding dl { display: grid; grid-template-columns: max-content 1fr; gap: 4px 14px; margin: 0; }
  .finding dt { font-weight: 600; color: var(--muted); }
  .finding dd { margin: 0; }
  .evidence { border-left: 3px solid var(--accent); padding-left: 10px; }
  /* Attack path: nodes joined by edges. The gap node is highlighted. */
  ol.path { list-style: none; margin: 8px 0; padding: 0; }
  ol.path > li.node { background: var(--card); border: 1px solid var(--line);
                      border-left: 4px solid var(--accent); border-radius: 6px; padding: 6px 12px;
                      font: .85rem/1.5 var(--mono); }
  ol.path > li.node.gap { border-left-color: var(--high); background: var(--unk-bg); }
  ol.path > li.node.impact { border-left-color: var(--crit); font-weight: 600; }
  ol.path > li.edge { position: relative; margin-left: 24px; padding: 6px 0 10px 14px;
                      border-left: 2px solid var(--muted); font: .78rem/1.4 var(--mono); color: var(--muted); }
  ol.path > li.edge::after { content: ""; position: absolute; left: -7px; bottom: -1px;
                             border: 6px solid transparent; border-top-color: var(--muted); border-bottom: 0; }
  @media print { body { background: #fff; } pre, .finding { break-inside: avoid; } }
</style>
</head>
<body>
<main>
  <header>
    <h1>shopapi Security Analysis</h1>
    <div class="meta">
      <span>Repository: <code>shopapi</code></span>
      <span>Commit: <code>3028d31</code></span>
      <span>Analyzed: 2026-01-15</span>
      <span>Generated by MDW security</span>
    </div>
    <nav class="toc"><ol>
      <li><a href="#scope">Scope</a></li>
      <li><a href="#findings">Security Findings</a></li>
      <!-- one item per section, in Markdown order -->
    </ol></nav>
  </header>

  <section id="scope">
    <h2>1. Scope</h2>
    <p>Summary paragraph.</p>
    <div class="counts">
      <span class="sev high">HIGH 4</span> <span class="sev medium">MEDIUM 3</span>
      <span class="label confirmed">CONFIRMED 9</span> <span class="label unknown">UNKNOWN 3</span>
    </div>
  </section>

  <section id="findings">
    <h2>15. Security Findings</h2>
    <div class="finding high">
      <h3>SEC-01 Potential Broken Object-Level Authorization on order lookup
        <span class="sev high">HIGH — static risk</span>
        <span class="label confirmed">CONFIRMED</span></h3>
      <dl>
        <dt>Category</dt><dd>Authorization</dd>
        <dt>Location</dt><dd><code>internal/order/handler.go Handler.Get</code></dd>
        <dt>Attack Surface</dt><dd><code>GET /orders/{id}</code></dd>
        <dt>Evidence</dt><dd class="evidence">FindByID runs <code>WHERE id = $1</code> with no
          user_id filter; no ownership check before the response.</dd>
        <dt>Attack Path</dt><dd>
          <ol class="path">
            <li class="node">[Authenticated user A]</li>
            <li class="edge">GET /orders/{id of user B's order}</li>
            <li class="node">RequireAuth() — ok, token valid</li>
            <li class="edge">Handler.Get()</li>
            <li class="node gap">Repo.FindByID(id) — !! no ownership check</li>
            <li class="edge">SELECT ... WHERE id = $1</li>
            <li class="node impact">200 OK: user B's order returned to user A</li>
          </ol></dd>
        <dt>Security Impact</dt><dd>Any authenticated user may read another user's order.</dd>
        <dt>How to Verify</dt><dd>Locally, as test user A, request test user B's order id.</dd>
        <dt>Recommended Fix</dt><dd><code>WHERE id = $1 AND user_id = $2</code>, 404 when not owned.</dd>
        <dt>Trade-offs</dt><dd>One extra predicate.</dd>
      </dl>
    </div>
  </section>

  <section id="secrets">
    <h2>9. Secrets</h2>
    <div class="table-wrap"><table>
      <thead><tr><th>Location</th><th>Type</th><th>Real or placeholder</th><th>Value</th><th>Label</th></tr></thead>
      <tbody><tr><td><code>internal/config/config.go Load</code></td><td>payment secret key</td>
        <td>live-mode prefix</td><td><code>sk_live_****[REDACTED]</code></td>
        <td><span class="label confirmed">CONFIRMED</span></td></tr></tbody>
    </table></div>
  </section>
</main>
</body>
</html>
```
