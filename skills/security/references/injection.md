# Injection

Analyze **actual data flows**, not string proximity. A finding requires attacker-controlled data
reaching a dangerous sink where no safe API or parameterization neutralizes the dangerous part.
Extends step 8 and the SSRF part of step 13. Path traversal and uploads: `file-security.md`.

## Contents

1. The method
2. SQL injection
3. Command injection
4. SSRF
5. Template / expression / other
6. XSS
7. Path traversal (pointer)

## 1. The method

For each candidate:

```text
source (untrusted input)  ->  ...transformations...  ->  sink (dangerous operation)
```

1. Identify the **sink** (query execution, command, outbound request, template render, HTML output).
2. Identify the **source** and confirm the attacker controls it.
3. Follow the path: is the dangerous part parameterized, escaped, allowlisted, or type-constrained
   before the sink?
4. Only if a controllable dangerous part reaches the sink unneutralized is it a finding.

A string reaching a function is not injection. Parameterization, prepared statements, safe builders,
and strict type conversion neutralize most candidates — say so and move on.

## 2. SQL injection

Inspect: raw SQL, string concatenation, `fmt.Sprintf`/f-strings/template literals into queries,
query builders used unsafely, ORM raw expressions, dynamic `WHERE`, and identifier interpolation.

**Values** are safe when passed as parameters:

```text
Unsafe:  "SELECT * FROM users WHERE id = " + id
         fmt.Sprintf("... WHERE email = '%s'", email)
Safe:    db.Query("SELECT * FROM users WHERE id = $1", id)
```

**Identifiers are the trap.** Parameters bind *values*, never table names, column names, `ORDER BY`,
`ASC/DESC`, or `LIMIT`. Code that parameterizes values but interpolates an identifier from input is
still injectable:

```text
q := fmt.Sprintf("SELECT ... FROM orders WHERE user_id = $1 ORDER BY %s", sort)  // sort from ?sort=
db.Query(q, userID)   // user_id is safe; ORDER BY is not
```

Analyze dynamic **column / table / ORDER BY / LIMIT / filter / sort** construction separately from
values. The fix for identifiers is an allowlist (map the input to a fixed set of permitted
columns/directions), not escaping.

This kind of injection is often **error-** or **boolean-based** even without a UNION, especially when
error strings are echoed to the client (`data-protection.md`): an invalid sort expression that
changes the response or leaks a DB error confirms control.

NoSQL: operator injection (a JSON body that becomes a Mongo query with `$where`/`$gt`), and
string-built queries in Cypher, N1QL, etc. Same rule: is the query structure attacker-controlled?

## 3. Command injection

Inspect `os/exec`, `exec.Command`, `child_process` (`exec` vs `execFile`/`spawn`), `subprocess`
(`shell=True` vs a list), `Runtime.exec`, backticks, `system()`.

Distinguish **argument passing** from **shell construction**:

```text
Safe:    exec.Command("convert", src, "-resize", "128x128", dst)   // argv, no shell
Unsafe:  exec.Command("sh", "-c", "convert "+src+" ...")           // shell parses src
         subprocess.run(f"convert {src} ...", shell=True)
```

Passing a user value as a **separate argument** to a program is not command injection, even if the
value is attacker-controlled — the shell never parses it. Report command injection only when input
reaches a shell string or the command/program name itself. (A user-controlled value passed as an
argument may still matter to *that program* — e.g. an image parser's own vulnerabilities — but that
is a dependency/observation, not command injection, and needs no fabricated CVE.)

## 4. SSRF

Server-Side Request Forgery: the server makes an outbound request to a destination the attacker
influences. Find the outbound sinks first (`http.Get`, `http.NewRequest`, `axios`, `fetch`,
`requests`, URL fetchers, webhook callers, image/URL preview, importers, proxies), then check who
controls the destination.

```text
Not SSRF:  request to a fixed host, only a path segment escaped/validated from input
SSRF:      request to a full URL / host / scheme taken from the request with no allowlist
```

For a real SSRF finding, confirm the attacker controls the **URL, host, scheme, or port**, and check
for controls: destination allowlist, scheme restriction (block non-`http(s)`), private/link-local IP
blocking (`127.0.0.0/8`, `10/8`, `172.16/12`, `192.168/16`, `169.254.169.254`, `::1`, `fc00::/7`),
redirect validation (a public URL can 302 to a private one), and DNS-rebinding protection.

Impact depends on where the server sits (cloud metadata, internal services), which is usually
`INFERRED`/`UNKNOWN` from the repo. Note that a handler which **returns the response body** to the
caller is a stronger SSRF (data exfiltration) than one that only fires the request. Do not claim SSRF
merely because the app makes outbound requests.

## 5. Template / expression / other

- **Template injection (SSTI)**: user input concatenated into a template *source* (not passed as
  data) in Jinja, Freemarker, Velocity, Thymeleaf, ERB, Go `text/template` used for HTML, Handlebars.
  Can reach RCE in some engines.
- **Expression injection**: SpEL, OGNL, MVEL, EL evaluated from input.
- **Header / CRLF injection**: input written into a response header or a constructed request line
  without stripping `\r\n` → header splitting, response splitting, log forging.
- **LDAP / XPath / GraphQL**: query structure built from input; and GraphQL depth/complexity abuse
  (unbounded nested queries) — see `api-security.md`.

## 6. XSS

Output-side injection: attacker input rendered into HTML/JS without contextual encoding.

- **Reflected/stored**: input echoed into a page. Modern template engines auto-escape (`INFERRED`
  safe) — the findings are where escaping is bypassed: `dangerouslySetInnerHTML`, `v-html`,
  `|safe`/`|raw`, `innerHTML`, `render_template_string`, `template.HTML`, building HTML by
  concatenation.
- **DOM XSS**: client code writes `location`/`document` data into a sink (`innerHTML`, `eval`,
  `document.write`).
- For **JSON APIs** with no HTML rendering, reflected XSS usually does not apply — say so rather than
  flagging every echoed field. Check the `Content-Type`.

## 7. Path traversal (pointer)

User-controlled file paths reaching file operations are covered in `file-security.md`. The same
data-flow method applies: is the path component attacker-controlled, and is it canonicalized and
confined to a base directory before the file operation?
