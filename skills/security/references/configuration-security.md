# Configuration Security

Deployment, infrastructure, and container configuration. Extends step 14 of `SKILL.md`. Judge
configuration in context: a compose file for local development is not production, and dev conveniences
in it are observations, not critical findings.

## Contents

1. What is this config for
2. Application configuration
3. Containers
4. Orchestration and IaC
5. TLS and reverse proxy
6. What to report

## 1. What is this config for

Before flagging anything, decide what a config file represents. `docker-compose.yml` at the repo root
is usually a **local dev** setup; a `k8s/production/` manifest is production; a `.env.example` is a
template. Do not assume deployment files represent production unless the path, comments, or values say
so. State the assumption in the finding, and downgrade dev-only issues to `OBSERVATION`.

## 2. Application configuration

| Check | Risk |
|---|---|
| Debug/development mode | stack traces and internals exposed (`data-protection.md`); Flask `debug=True` exposes a console; Django `DEBUG=True`; Rails dev; verbose error pages |
| Secrets in config | hardcoded credentials (`secrets.md`) |
| Insecure defaults | permissive CORS, disabled auth in a "dev" branch that ships, feature flags that widen access |
| Verbose logging in prod | sensitive data in logs (`logging-security.md`) |
| Disabled security features | TLS verification off (`InsecureSkipVerify: true`, `verify=False`, `rejectUnauthorized: false`), CSRF disabled, host checks off |

`InsecureSkipVerify`/`verify=False` on an outbound TLS client is a concrete finding: it defeats
certificate validation and enables man-in-the-middle. Check whether it is guarded to non-prod.

## 3. Containers

| Check | Concern |
|---|---|
| `USER` | running as root (no `USER` directive, or `USER root`) — a `LOW`/`MEDIUM` hardening issue on its own; do not make every `USER root` a critical. It raises the impact of another RCE/escape. |
| Capabilities / `privileged` | `privileged: true` or added capabilities widen container escape → higher |
| Read-only filesystem | writable root fs where not needed |
| Base image | large/unpinned base (`:latest`), or one carrying its own package vulns (scan, don't assume) |
| Secret handling | secrets baked into image layers or `ENV` (persist in the image) vs mounted at runtime |
| Build contents | copying `.git`, source, or `.env` into the final image; multi-stage build absent |
| Exposed ports | ports exposed beyond what is needed |
| Healthcheck | present (availability, minor) |

## 4. Orchestration and IaC

Kubernetes, Helm, Terraform, CloudFormation, serverless:

```text
securityContext: runAsNonRoot, readOnlyRootFilesystem, allowPrivilegeEscalation, dropped capabilities
secrets as plain env vs Secret objects / external secret managers
overly broad IAM roles / policies (wildcard actions or resources)
public exposure: LoadBalancer/Ingress on internal services, security groups open to 0.0.0.0/0
databases/caches reachable from the internet (published ports, public subnets)
network policies present or absent
```

Wildcard IAM (`Action: "*"`, `Resource: "*"`) and internet-open datastores are concrete findings.
Rate by exposure and blast radius.

## 5. TLS and reverse proxy

```text
TLS terminated and enforced (HTTP redirected to HTTPS)   modern TLS versions/ciphers
HSTS   security headers (CSP, X-Content-Type-Options, X-Frame-Options/frame-ancestors)
proxy passing/overriding X-Forwarded-* (trust boundary for client IP and host)
request size and timeout limits at the proxy
```

Missing security headers are `LOW`/`OBSERVATION` unless a specific attack (clickjacking on a
sensitive action, MIME sniffing of user content) makes one material.

## 6. What to report

| Finding | Typical severity |
|---|---|
| Debug mode / verbose errors enabled in production | `MEDIUM`/`HIGH` |
| TLS verification disabled on an outbound client | `HIGH` (MITM) |
| `privileged` container / broad added capabilities | `MEDIUM`/`HIGH` |
| Wildcard IAM policy | `HIGH` by blast radius |
| Datastore exposed to the internet | `HIGH` |
| Container runs as root (no other exposure) | `LOW`/`OBSERVATION` |
| Secrets baked into image / `ENV` | `MEDIUM` (with `secrets.md`) |
| Missing security headers | `LOW`/`OBSERVATION` |
| Dev-only convenience in a local compose file | `OBSERVATION`, stated as dev-only |

Verify safely by reading the config; do not deploy or modify live infrastructure to test.
