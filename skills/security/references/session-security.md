# Session Security

Cookies, CSRF, CORS, and the session lifecycle. Extends step 9 of `SKILL.md`. Token entropy and
lifecycle for bearer tokens are in `authentication.md` and `cryptography.md`.

**First, establish the authentication model**, because it decides which of these apply:

- **Cookie-based session in a browser** → CSRF and cookie flags matter.
- **Bearer token (`Authorization` header) API** → CSRF generally does not apply (the browser does not
  attach the header automatically); CORS with credentials matters less. Say so instead of flagging
  every state-changing route.

## Contents

1. Cookie flags
2. Session lifecycle
3. CSRF
4. CORS

## 1. Cookie flags

For each cookie that carries authentication or session state, check the attributes actually set in
code or config (not the framework default unless the default is confirmed):

| Attribute | Why | Missing means |
|---|---|---|
| `Secure` | only sent over HTTPS | can leak over plaintext |
| `HttpOnly` | not readable by JS | stealable via XSS |
| `SameSite` | `Lax`/`Strict` limits cross-site sending | `None` (or unset on some stacks) enables CSRF |
| `Domain`/`Path` scope | limits where it is sent | over-broad scope shares it with subdomains |
| `Expires`/`Max-Age` | lifetime | overly long-lived sessions |

## 2. Session lifecycle

- **Fixation**: the session id must be regenerated on privilege change (login). Reusing the
  pre-login id lets an attacker who planted it ride the authenticated session.
- **Invalidation**: logout must destroy server-side state, not just clear the cookie. Password
  change and role change should invalidate other sessions.
- **Expiration**: idle and absolute timeouts. A session that never expires is a standing risk.
- **Storage**: server-side store vs signed client-side session; if client-side, the signing key is a
  secret (`secrets.md`) and tampering must be rejected.

## 3. CSRF

Applies to **state-changing requests authenticated by an ambient credential the browser sends
automatically** (a session cookie, HTTP Basic). For those, check for at least one defense:

```text
anti-CSRF token (synchronizer or double-submit)   SameSite cookie (Lax/Strict)
Origin header validation   Referer validation   a required custom header (for XHR)
framework CSRF middleware (and that it is enabled for these routes)
```

Do **not** report CSRF merely because `POST`/`PUT`/`DELETE` routes exist. If the auth is a bearer
token in a header, CSRF typically does not apply — record that reasoning under
"Reviewed, not a vulnerability". If cookie auth and no defense, report it with the affected routes.

## 4. CORS

CORS controls which web origins may read cross-origin responses; it does not protect the server, it
protects other sites' users. Inspect the response headers the app sets:

```text
Access-Control-Allow-Origin        Access-Control-Allow-Credentials
Access-Control-Allow-Methods       Access-Control-Allow-Headers
```

Dangerous combinations:

| Configuration | Risk |
|---|---|
| `Allow-Origin: *` **with** `Allow-Credentials: true` | browsers forbid this combo; if a framework emits it, it is broken — but check it is not achieved via reflection instead |
| Reflecting the request `Origin` **with** `Allow-Credentials: true` | any site can make credentialed cross-origin reads → account data theft, **when the credential is a cookie the browser sends automatically** |
| Reflecting `Origin` with credentials, but auth is a **bearer header** | limited: the attacker's page cannot obtain the victim's token to set the header, so it cannot forge authenticated reads. Usually `LOW`/`OBSERVATION` — state the reasoning. |
| Over-broad `Allow-Origin` on public, unauthenticated data | low impact |

Assess CORS against the **actual authentication model**. Reflected origin + credentials is serious
for cookie auth and often minor for header-bearer auth. Explain which case this repo is in rather
than flagging the header pattern alone.
