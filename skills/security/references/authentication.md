# Authentication

How identity is established, and whether the mechanism is validated against its own rules. Extends
steps 5 and 9 of `SKILL.md`. Token lifecycle and entropy also appear in `cryptography.md`; cookie
and CSRF behavior in `session-security.md`.

## Contents

1. Find the mechanism
2. JWT
3. Sessions
4. OAuth / OIDC
5. API keys and service tokens
6. Passwords and reset flows
7. What to report

## 1. Find the mechanism

| Mechanism | Evidence |
|---|---|
| JWT | `golang-jwt`, `jsonwebtoken`, `jose`, `PyJWT`, `nimbus-jose-jwt`, `System.IdentityModel.Tokens.Jwt` |
| Session cookie | `express-session`, `gorilla/sessions`, Django sessions, Spring Session, `Set-Cookie` writes |
| OAuth / OIDC | `golang.org/x/oauth2`, `passport-*`, `authlib`, `spring-security-oauth2`, `coreos/go-oidc` |
| API key | a header/query read (`X-API-Key`) compared to a stored value |
| Basic Auth | `BasicAuth()`, `Authorization: Basic` parsing |
| mTLS | `tls.Config{ClientAuth: RequireAndVerifyClientCert}`, client-cert checks |
| Signed webhook | HMAC over the body compared to a header |
| Custom | anything else — read it in full |

Record **where identity is established** (the middleware or guard), **what it trusts** (the token,
a header, a cookie, a client certificate), and **which routes it covers**. A trusted header such as
`X-User-Id` set by a gateway is only safe if the service is not reachable except through that
gateway — usually `UNKNOWN` from the repo alone.

## 2. JWT

Read the parse/verify call and its options. For each property, record what the code does:

| Property | Check | Notes |
|---|---|---|
| Signature | is the token verified, or only decoded? | `jwt.decode` without verify, `ParseUnverified`, `verify=False`, `algorithms` omitted in some libraries |
| Algorithm | is the accepted algorithm pinned? | Pinning (`WithValidMethods`, `algorithms=[...]`) is hardening. Whether a missing pin is exploitable depends on the library and key type: an HMAC-only `[]byte` key in `golang-jwt` v5 rejects `none` and fails RS/ES verification, so a missing pin there is an `OBSERVATION`, not an alg-confusion finding. Algorithm confusion requires a public key usable as an HMAC secret. |
| Expiration | is `exp` set when issuing, and enforced when parsing? | Many libraries enforce `exp` *if present* but do not require it. A token issued without `exp` never expires. |
| Issuer / audience | are `iss`/`aud` checked? | Matters when several services or tenants share a key or IdP. |
| Subject | is identity taken from `sub`/claims after verification? | Never from an unverified part. |
| Key source | where does the signing key come from? | Hardcoded or defaulted secrets → `secrets.md`. |
| Claims trusted for authorization | role/permission in the token | Stale after role changes unless tokens expire or are revoked. |

Do not claim a missing validation unless the source shows it. "Uses JWT" is not a finding.

## 3. Sessions

Session identity lives in a cookie mapped to server state. Check: session id entropy (framework
default is usually fine — `INFERRED`), regeneration on login (fixation), invalidation on logout and
password change, idle and absolute expiration, and storage. Cookie flags and CSRF:
`session-security.md`.

## 4. OAuth / OIDC

| Check | What goes wrong |
|---|---|
| `state` parameter generated, stored, and compared | login CSRF / account linking to the attacker's account |
| `nonce` (OIDC) validated in the ID token | token replay |
| Redirect URI exact-match allowlist | authorization code sent to an attacker-controlled URL |
| ID token signature, `iss`, `aud`, `exp` verified | forged identity |
| Account linking by email only when the IdP asserts the email is verified | account takeover across providers |
| PKCE for public clients | code interception |

## 5. API keys and service tokens

Check how keys are generated (entropy: `cryptography.md`), stored (hashed vs plaintext), compared
(constant-time), scoped (per tenant, per permission), rotated, and revoked. A service token shared by
every internal caller means any compromised internal service is every service.

## 6. Passwords and reset flows

| Area | Check |
|---|---|
| Storage | bcrypt / argon2 / scrypt / PBKDF2 with adequate cost; never plain, reversible, or fast hashes (MD5, SHA-1, unsalted SHA-256) |
| Comparison | the library's compare function; constant-time |
| Brute force | a login attempt limit, lockout, or rate limit — or its confirmed absence, reported per `api-security.md` |
| Enumeration | do login and reset responses differ for existing vs non-existing accounts? |
| Reset token | cryptographically random (`cryptography.md`), expires, single-use, invalidated after use and on password change, stored hashed, not logged, not in URLs that reach logs or `Referer` |
| Reset confirmation | changes only the token owner's password; invalidates sessions |
| MFA | if present: enforced on the sensitive paths, not bypassable via another login route, codes rate-limited and single-use |

Never output a real password or reset token in the artifacts.

## 7. What to report

| Finding | Typical severity |
|---|---|
| Token decoded but not verified | `CRITICAL` |
| Signing secret hardcoded, or defaulted when env is unset | `HIGH`/`CRITICAL` — static risk; exposure `INFERRED` from deploy config |
| Tokens issued without expiration | `MEDIUM` |
| Role claims trusted for authorization with no expiry/revocation | `MEDIUM` (raises chains) |
| Predictable reset or session tokens | `HIGH` |
| Reset token not single-use / not invalidated | `LOW`–`MEDIUM` |
| No brute-force control on login | `OBSERVATION` unless the threat model makes it material |
| Algorithm not pinned with an HMAC-only key | `OBSERVATION` (hardening) |

State the verification safely: issue a token with a test key in a unit test, parse a token lacking
`exp`, or check a reset token's properties in a test — never against real accounts.
