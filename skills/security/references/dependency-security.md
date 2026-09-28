# Dependency Security

Dependencies and their versions. Extends step 14 of `SKILL.md`. The governing rule: **never fabricate
or assert a CVE you have not verified this session.**

## Contents

1. Read the manifests
2. The no-fabricated-CVE rule
3. What to check without a scanner
4. What to report

## 1. Read the manifests

```text
Go            go.mod, go.sum
Node          package.json, package-lock.json, pnpm-lock.yaml, yarn.lock
Python        requirements*.txt, pyproject.toml, poetry.lock, Pipfile.lock
Java/Kotlin   pom.xml, build.gradle(.kts), gradle.lockfile
Ruby          Gemfile, Gemfile.lock
PHP           composer.json, composer.lock
Rust          Cargo.toml, Cargo.lock
Containers    Dockerfile base images and pinned tags (see configuration-security.md)
```

Record the direct dependencies that matter for security (web framework, auth/JWT, crypto, ORM/DB
driver, serialization, HTTP client, template engine) with their **pinned versions** from the
lockfile where one exists. Note the absence of a lockfile (`go.sum`, `package-lock.json`) — it means
builds are not reproducible and transitive versions are unpinned, which is itself an observation.

## 2. The no-fabricated-CVE rule

CVE and advisory identifiers are precise facts. An invented or misremembered id (wrong number, wrong
package, wrong affected range) is worse than saying nothing: it sends people chasing a phantom and
discredits the report.

```text
Do NOT state "CVE-XXXX-YYYYY" or "GHSA-xxxx" from memory or inference.
```

- State a specific advisory **only** when a tool run **in this session** produced it (`govulncheck`,
  `npm audit`, `pip-audit`, `osv-scanner`, Trivy, Dependabot output the user shared) — and then cite
  the tool and its output.
- If no such evidence is available, record the version and write:

> Dependency version identified. Known-vulnerability status not verified (no advisory data available
> in this session). Recommend running `govulncheck` / `npm audit` / `osv-scanner` against the
> lockfile.

- Recommending that the user run a scanner is the correct output when you cannot verify. Do not
  approximate a scanner from memory.
- General, non-identifier reasoning is fine when clearly labeled `INFERRED`: "this major version is
  several years old; upgrading is advisable" — without inventing a CVE number.

## 3. What to check without a scanner

Even without advisory data, static observations are legitimate:

| Observation | Note as |
|---|---|
| A very old major version of a security-relevant library | `INFERRED` / `OBSERVATION`: outdated, upgrade advisable; status not verified |
| A dependency pulled from an untrusted or non-standard source (git URL, fork, vendored copy) | supply-chain observation |
| A dependency that looks abandoned/unmaintained | observation |
| Unnecessary or duplicated dependencies | observation, low priority |
| Missing lockfile / unpinned versions | observation: non-reproducible builds |
| A dependency whose purpose is security (crypto, auth) at an old version | flag for scanning specifically |

## 4. What to report

Report dependency findings as observations with versions and a scan recommendation, unless a tool in
this session confirmed a specific advisory — then report it with the tool citation and set severity
from the advisory. Keep the dependency table simple:

```text
Package | Version | Role | Status
--------|---------|------|-------------------------------------------------------
golang-jwt/jwt/v5 | v5.0.0 | JWT verification | version identified; run govulncheck to verify advisories
```

Do not turn "a dependency exists" or "a version is not the latest" into a vulnerability on its own.
The finding is either a verified advisory (with evidence) or an observation to act on by scanning.
