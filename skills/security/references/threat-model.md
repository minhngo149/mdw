# Threat Model

This reference covers scope and technology discovery, attack-surface enumeration, trust boundaries,
and the lightweight threat model built from the actual repository. It extends steps 1-4 of
`SKILL.md`.

## Contents

1. Discover the technologies present
2. Enumerate the attack surface
3. Map trust boundaries
4. Assets, actors, and sensitive operations
5. Attack paths
6. Repository shapes

## 1. Discover the technologies present

Stop once you can name the processes that expose entry points, the identity mechanism, and the
infrastructure they touch. Everything else comes from tracing.

| Question | Look at |
|---|---|
| Language and framework | `go.mod`, `package.json`, `pom.xml`, `build.gradle(.kts)`, `pyproject.toml`, `requirements*.txt`, `Gemfile`, `Cargo.toml`, `composer.json`, `*.csproj`. The dependency list names the web framework, ORM, auth library, HTTP client, and crypto libraries. |
| What runs | `cmd/*/main.go`, `main.*`, `apps/*`, `services/*`, `Dockerfile*`, `docker-compose*.yml`, Kubernetes/Helm, `Procfile`, `serverless.yml` |
| Identity and access | auth middleware, guards, interceptors, JWT/session/OAuth libraries, `@PreAuthorize`, policy files, role enums |
| Data stores | DB drivers and ORMs, migrations, cache clients, object-storage SDKs |
| External calls | HTTP clients, SDKs (payment, email, cloud), webhook callers |
| Deployment | Dockerfiles, compose, k8s, Terraform, Helm, CI workflows, reverse-proxy and TLS config |

Useful first searches (use the Grep and Glob tools, or `rg`):

```bash
git ls-files | grep -Ei '(^|/)(main\.[a-z]+|Dockerfile[^/]*|docker-compose[^/]*|\.env[^/]*)$'
rg -l --no-messages -i 'jwt|jsonwebtoken|passport|oauth|oidc|session|bcrypt|argon|scrypt'
rg -l --no-messages -i 'exec\.Command|child_process|subprocess|os\.system|Runtime\.getRuntime'
rg -l --no-messages -i 'http\.Get|http\.NewRequest|axios|fetch\(|requests\.(get|post)|HttpClient'
```

Do not assume the repository is an HTTP API. A CLI's attack surface is its arguments, environment,
and the files it reads. A worker's is its message payloads. A cron job's is the data it reads.

## 2. Enumerate the attack surface

Find where each entry point is **registered**, not a function with a matching name.

| Entry type | Registration patterns |
|---|---|
| HTTP / REST | Go `mux.Handle("GET /x"`, `.GET(`, `HandleFunc(`; Node `router.get(`, `app.post(`, `@Get(`; JVM `@GetMapping`, `@RequestMapping`; Python `@app.get`, `urls.py`, `@api_view`; Ruby `routes.rb`; .NET `[HttpGet]`, `MapGet(` |
| GraphQL | SDL `type Query`/`type Mutation`, resolver maps, `@Resolver`, `@Mutation` |
| gRPC | `.proto` `service`/`rpc`, `RegisterXServer(`, `@GrpcService` |
| Webhook | an HTTP route plus signature verification plus an event-type switch |
| Message consumer | `Consume(`, `@RabbitListener`, `@KafkaListener`, `Subscribe(`, `ReceiveMessage` |
| CLI | cobra `&cobra.Command{`, click, argparse, commander |
| Scheduled job | `cron.AddFunc(`, `@Scheduled(`, Celery beat, k8s `CronJob` |
| File upload | multipart parsing (`FormFile`, `multer`, `MultipartFile`, `request.files`) |
| Admin / internal | routes under `/admin`, `/internal`, debug handlers (`/debug/pprof`, actuator) |

For each meaningful entry point record, in the attack-surface table:

```text
Entry | Authentication | Authorization | Input | Sensitive operation | Data access | Evidence | Label
```

Read the middleware chain in execution order. A route inherits the controls of every middleware
wrapping it; a route registered outside the wrapped group inherits none. Mixed groups are a common
source of missing-auth findings: note exactly which wrapper each route has.

Pay attention to routes that are **public by accident**: a handler added to the public group, a
debug endpoint left mounted, a health check that returns config, an actuator exposed.

## 3. Map trust boundaries

A trust boundary is where data moves between parties that trust each other differently. Draw it as
plain text:

```text
[Internet]
   |  untrusted: any request
   v
[API]  cmd/api       authn: bearer JWT    authz: per-route
   |
   +--> [PostgreSQL]      trusted store; receives queries built from request data
   +--> [Payment API]     third party; its responses are trusted for order status
   +--> [Any URL]         (?) outbound to caller-supplied URL
```

For each boundary ask: **does the application validate and authorize data crossing it?**

| Boundary | What to check |
|---|---|
| Internet → API | authentication, input validation, size limits, rate limits |
| API → database | parameterization, tenant/ownership filters, least-privileged DB user |
| API → internal service | does the callee re-authorize, or trust the caller blindly? |
| Inbound webhook | signature, timestamp, replay protection, secret presence |
| Outbound request | who controls the destination (SSRF) |
| Broker producer/consumer | is the message trusted because it is "internal"? |
| File storage | who controls the path; is the stored file served back or executed? |
| OAuth / identity provider | state, nonce, redirect URI validation, token audience |
| Admin interface | separate authentication and strong authorization |

## 4. Assets, actors, and sensitive operations

**Assets** — what an attacker would want: credentials, tokens, personal data, payment data, other
users' resources, admin capability, money/credit, the host itself (RCE), internal network reach.

**Actors** — only those the repository grants:

| Actor | Granted by |
|---|---|
| anonymous user | any public route |
| authenticated user | any route behind authentication, with that user's identity |
| admin | routes behind a role/permission check that this repo grants to some users |
| service account / internal service | service tokens, mTLS, network-only trust |
| third-party service | webhook senders, OAuth providers, payment providers |
| malicious external system | a URL the app fetches, a message a broker delivers |

Never assume an actor has privileges the repository does not grant. If a role check exists, the
default actor is the least-privileged one that reaches the route. If no admin role exists in code,
there is no admin actor.

**Sensitive operations** — authentication, password reset, role/permission change, payment,
refund, credit, data export, file access, outbound request, admin actions, deletion.

## 5. Attack paths

An attack path joins an actor to an asset through the entry points and missing controls you found.
Build them after the per-category analysis, in step 16 of `SKILL.md`:

```text
Actor -> Entry point -> Missing/bypassable control -> Sink -> Asset / impact
```

Look deliberately for **chains**, where one finding enables another:

- mass assignment of a role field + authorization that trusts that role → privilege escalation
- a hardcoded or default signing secret + role claims in tokens → forged admin tokens
- an injectable sink + errors echoed to the client → data extraction through error messages
- SSRF + a cloud metadata endpoint reachable from the host → credential exposure (`INFERRED`/`UNKNOWN`: depends on deployment)
- predictable reset tokens + no rate limit on the confirm endpoint → account takeover

Draw each path with the format in `security-evidence.md` section 7.

## 6. Repository shapes

| Shape | What changes |
|---|---|
| Monolith | One process. The value is in per-route authorization and in-process data flow. |
| Microservices in one repo | Trace both ends. Check whether internal services re-authorize or trust headers like `X-User-Id` from the gateway. |
| One service of many | Upstream gateways and auth may live elsewhere. Mark their controls `UNKNOWN`; do not assume they exist. |
| Serverless | Each function is an entry point; triggers and IAM live in `serverless.yml`, SAM, CDK, or Terraform. |
| Library / SDK | There is no attack surface of its own; analyze the API it exposes to callers and what it trusts. |
| CLI | Inputs are args, env, stdin, and files. The attacker is whoever controls those, often the same user — scope accordingly. |
