# Authorization

Authorization is the primary focus of most repository security reviews: authentication says *who you
are*, authorization says *what you may do*. Extends step 6 and the logic part of step 15 of
`SKILL.md`.

For every sensitive operation, answer: **who** can perform it, on **what** resource, with **which**
operation, and **where** is that enforced?

## Contents

1. Find the enforcement points
2. IDOR / BOLA
3. Privilege escalation and mass assignment
4. Multi-tenant isolation
5. Function- and field-level authorization
6. Business-logic authorization
7. What to report

## 1. Find the enforcement points

| Model | Evidence |
|---|---|
| RBAC | role checks: `RequireRole`, `hasRole`, `@PreAuthorize("hasRole")`, `if user.Role != ...` |
| ABAC / policy | policy engines (OPA, Casbin, Cedar), `can?`, Pundit/CanCan, `@PostAuthorize` |
| Ownership | queries filtered by the caller's id, or a check that `resource.OwnerID == caller` |
| Middleware/guard | authorization applied per route or per group |

Map each sensitive operation to the checks that run before it. The dangerous cases:

- **No check** between loading a resource and returning or mutating it.
- **Authentication mistaken for authorization**: the route requires a valid token but never checks
  that this user may act on this resource.
- **Check on the wrong subject**: verifies a role but not ownership, or ownership but not the role
  needed for a privileged field.
- **Client-supplied identity**: authorization based on a `user_id` from the body/query instead of
  the authenticated identity.

## 2. IDOR / BOLA

Broken Object-Level Authorization: the caller names a resource by id and the code returns or mutates
it without checking they may. The most common high-impact web finding.

Look at every operation that takes a resource identifier:

```text
GET /resource/{id}      PUT /resource/{id}      DELETE /resource/{id}
GraphQL node(id:)        gRPC GetX(id)           a message referencing an id
```

Trace:

```text
request id -> datastore lookup -> [ownership / permission check?] -> operation
```

If the lookup is `SELECT ... WHERE id = $1` (or `findById(id)`) and nothing checks the caller owns
or may access the row before it is returned or changed, report it. Two strong tells:

- A **sibling** endpoint filters by owner (for example `List` uses `WHERE user_id = $1`) while the
  by-id endpoint does not — the intended rule is ownership, and the by-id path breaks it.
- The id is sequential/guessable, so enumeration is trivial (raises severity).

Do not assume the resource is public. If access truly is intended to be public, say why (evidence),
and record it as reviewed-not-a-vulnerability.

Fix at the boundary: scope the query to the caller (`WHERE id = $1 AND user_id = $2`) or check
ownership in the service/policy layer, returning 404 (not 403) so ids are not confirmed.

## 3. Privilege escalation and mass assignment

**Vertical**: a lower-privileged user gains higher privileges. **Horizontal**: a user acts as
another user of the same level (IDOR is often horizontal).

**Mass assignment / over-posting** is the classic escalation source: the handler binds request JSON
directly into a model or an UPDATE that includes privileged columns.

```text
request body (JSON)  ->  struct / ORM model / SQL  ->  privileged field written
```

Look for sensitive fields the user should not control:

```text
role  status  is_admin  is_verified  owner_id  user_id  tenant_id  permissions
balance  credit  price  discount  approved  verified  email_verified
```

Patterns to flag:

- A single struct used for both input and storage, decoded from the body, then persisted whole.
- An UPDATE listing user-controlled columns that include a privileged one (for example
  `SET role = $1` fed from the body).
- ORM `update(req.body)`, `Model.objects.update(**data)`, `attrs.merge`, `ModelMapper` with no
  allowlist.

Confirm the field is reachable from input **and** trusted downstream (for example, a `role` a user
can set that authorization then believes). That combination is a privilege-escalation chain —
document it as one attack path (`security-evidence.md` section 7).

Fix: bind an explicit input DTO with only user-editable fields; set privileged fields server-side.

## 4. Multi-tenant isolation

When the repo is multi-tenant, tenant isolation is authorization. Detect the tenant key:

```text
tenant_id  organization_id  org_id  account_id  workspace_id  merchant_id  company_id
```

A field named `tenant_id` does not prove isolation. Confirm the tenant context is derived from the
**authenticated identity** (not the request body) and is applied to **every**:

```text
SELECT  INSERT  UPDATE  DELETE   cache key   message payload/routing
background job   file path/prefix   outbound API call   search index query
```

The dangerous pattern is a **global-id lookup without tenant scoping**: `WHERE id = $1` on a
tenant-owned table, letting one tenant read another's row. Also check: does a tenant admin's role
apply only within their tenant, or globally? Are cross-tenant references (a shared resource id)
validated against the caller's tenant?

## 5. Function- and field-level authorization

- **Function level**: is each admin/privileged endpoint behind the right check, including
  non-obvious ones (bulk actions, exports, debug routes, GraphQL mutations, gRPC methods)? A route
  registered outside the protected group has none of its checks.
- **Field level**: does a response include fields the caller may not see (another user's email,
  internal flags, password hashes)? Does an input let them set fields they may not (section 3)?
- **GraphQL**: authorization must be on resolvers/fields, not only at the HTTP layer, because one
  query reaches many resolvers. Check nested resolvers and `node(id:)` interfaces.

## 6. Business-logic authorization

Logic flaws are authorization on *state* and *sequence*, not on identity. Analyze only behavior
present in the repo:

| Flaw | What to look for |
|---|---|
| Double spend / duplicate redemption | credit, coupon, refund, or voucher applied without a guard against reuse (see race conditions in `data-protection.md`) |
| Workflow / approval bypass | a state reachable without the step that should precede it (paid without charge, shipped without payment) |
| Status manipulation | client can set a status field that should be server-controlled (mass assignment, section 3) |
| Ownership transfer | changing `owner_id`/`user_id` on a resource without authorization |
| Price / quantity manipulation | totals computed from client-supplied prices or negative quantities |
| Refund / coupon abuse | refund exceeding paid amount; coupon reused across accounts |
| Replay | the same signed request or event accepted twice (see `api-security.md`) |

## 7. What to report

| Finding | Typical severity |
|---|---|
| Missing ownership check on by-id read/mutate (IDOR/BOLA) | `HIGH` — static risk |
| Mass assignment of a privileged field, trusted downstream | `HIGH`/`CRITICAL` (chain) |
| Missing tenant scoping on a global-id lookup | `HIGH`/`CRITICAL` |
| Privileged endpoint outside the auth group | `HIGH`/`CRITICAL` |
| Authorization on client-supplied identity | `HIGH` |
| Business-logic bypass with financial impact | `HIGH`/`MEDIUM` |

Verify safely with two test accounts in a local environment: as user A, attempt B's resource; as a
low-privilege user, attempt a privileged field or endpoint. Expect a denial; a success confirms the
finding. Never test against real users or production.
