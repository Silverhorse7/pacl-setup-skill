---
name: pacl-setup
version: 4.0.0
description: |
  Use when asked to "set up PACL", "add authorization", "wire up pacl",
  "implement these permissions", or "what permissions do I need" for a
  partner-facing FastAPI service.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - AskUserQuestion
---

# /pacl-setup — PACL Setup Coordinator

This skill does **not** own the ACL contract. The registry repo
(`mp-partner-acl-registry`) owns the YAML shape, role/permission forms, policy,
label, condition format, hierarchy semantics, validation, and PR rules — through
its own `AGENTS.md`. Read that there; do not copy it here. If registry behavior
changes, the registry teaches the agent, not this skill.

**Usage:** `/pacl-setup [path]` — defaults to the current working directory.

---

## The model — two flows, registry is the pivot

```text
Flow A — author:    service scan ──► ACL model ──► registry acl.yaml ──► PR
Flow B — enforce:   registry acl.yaml ──► pacl.check() in service
```

The registry is the source of truth in both directions: **A writes it, B reads
it.** They are independent entry points — run A alone (author the namespace, stop
at the PR), B alone (namespace already in the registry, just wire enforcement),
or A→B for a full setup. When doing both, run B against the **merged** registry
state, not A's open PR — the check must match what is actually defined.

Pick the flow from the request:

- **Flow A** — the user wants the model created/changed from the service, or asks
  "what permissions do I need". (Old "design mode".)
- **Flow B** — the namespace already exists in the registry and the user wants
  `pacl.check()` wired into the service. (Old "implementation mode".)

Most "set up PACL from scratch" requests are A then B.

---

## Flow A — author the registry

### A1. Scan the service

```bash
CODEBASE="${1:-$(pwd)}"
echo "Scanning service repo: $CODEBASE"
```

Assume FastAPI unless the repo proves otherwise. Find partner-facing route files
only — skip team/admin/internal apps:

```bash
find "$CODEBASE/src" -name "*.py" \
  -not -path "*/appteam/*" \
  -not -path "*/appadmin/*" \
  -not -path "*/app*team*/*" \
  -not -path "*/team/*" \
  | xargs grep -l "@router\.\|@app\." 2>/dev/null
```

Confirm partner-facing status by reading code, not filename alone. Signals:
partner route prefixes, partner middleware, `Context.partners()`, or context
fields like `user_code`, `project_code`, `id_partner`, `partner_code`. Skip
`/hc`, `/health`, `/swagger`, `/openapi`, `/docs`.

For every partner-facing endpoint capture: HTTP method, path, route function, the
domain/service operation called, summary/tags, and sensitive behavior (create,
update, delete, approve, reject, import, export, upload, permission management,
finance/legal/user access, sudoer behavior).

```bash
grep -rh "@router\.\(get\|post\|put\|delete\|patch\)" <route_files> \
  | grep -v "hc\|health\|swagger\|openapi\|docs" | sort -u
```

Grep starts the list; then **read the full route and domain files**. The
permission protects the business action, not the HTTP path.

### A2. Build the ACL model (handoff)

Working context for the registry write, not the final artifact. Resolve:

- **Namespace** — in order: existing registry namespace file → service constant
  (`NAMESPACE`, `namespace_code`) → user-provided → repo-name inference
  (`mp-invoice-api` → `invoice`). Ask only if these are absent or conflict.
- **Resource type** — the core entity in the path (segment before the action).
  `/invoice/item/create` → `item`.
- **Action** — verb from path/method: `list`, `get`, `create`, `update`,
  `delete`, `approve`, etc. Otherwise the last path segment.
- **Roles** — propose `viewer`/`admin`/`owner` and confirm; adjust to the actors
  the codebase suggests. Roles are full strings `<ns>.<resource_type>.<actor>`.
- **Hierarchy intent** — including platform inheritance (root roles normally
  inherit the matching `platform.project.*` role).

**Conditions and policies — detect, do not invent.** Flow A surfaces a *signal*
and confirms it; it does not silently add conditions or policies:

- **Condition signal** — the endpoint filters/branches on a stable resource
  attribute so the same permission should be *scoped* by that field (e.g.
  `merch_type`, region, business). Surface it as an open question; only define it
  if the user confirms.
- **Policy/label signal** — access is gated on a role/resource property rather
  than granted uniformly — internal-only (`is_internal`) is the live example
  (`internalize`). Same rule: surface, confirm, then define.
- **`is_sudoer`** — a runtime check argument, **not** a registry condition (see
  B3). The sudoer context is populated by default; you do not add it to *enable*
  impersonation. Flag the inverse: **critical/sensitive actions** the real owner
  or org/project member must perform and an internal employee must **not** do on
  their behalf (create org/project, ownership or permission changes,
  finance/legal). Those are the endpoints that need a sudoer restriction.

### A3. Write the registry + open the PR

Locate the registry, cloning if absent, then read its instructions:

```bash
REGISTRY="${ACL_REGISTRY_DIR:-$PWD/.acl-registry}"
[ -d "$REGISTRY/.git" ] || \
  git clone https://github.com/fastfishio/mp-partner-acl-registry "$REGISTRY"
cd "$REGISTRY"
sed -n '1,200p' AGENTS.md
```

From here, **follow the registry's `AGENTS.md` as authoritative** for YAML
layout, role/permission/condition/policy form, formatting, validation, and PR
expectations. Create or patch `namespaces/<namespace_code>/acl.yaml`; start from
`namespaces/_template/acl.yaml` for a new namespace. Preserve unrelated existing
state — do not silently replace owners, metadata, or unrelated
roles/permissions/conditions/policies/hierarchy.

Run the registry's documented format/validate/plan commands exactly. If a command
fails because a valid ACL shape is unsupported, fix the registry tooling in the
registry repo — do not weaken the model. Open the PR per the registry's PR-notes
section. Never commit production user assignments, auth files, cookies, tokens,
service-account JSON, or copied request headers.

---

## Flow B — enforce from the registry

Flow B reads two things: the **registry acl.yaml** (what the check asserts) and
the **service scan** (where the check goes).

### B1. Read the registry as source of truth

```bash
cd "$REGISTRY"
sed -n '1,400p' "namespaces/<namespace_code>/acl.yaml"
```

Take the permissions and the `conditions:` defined per resource_type straight
from this file. Do not re-derive or rename them.

### B2. Place the check

Prefer the domain/service operation that owns the business action; if there is no
such layer, the top of the route handler. Read the service `Context` object for
real field names — do not guess (`ctx.user_code`, `ctx.project_code`,
`ctx.partner_code`, `ctx.id_partner`).

### B3. Emit `pacl.check()`

Real signature (`noonhelpers/v1/pacl/pacl.py`):
`check(permission, conditions=None, principal=None, resource=None, ...)`.

**`conditions={}` is the same wire for everything.** Populate it with **every
condition the registry defines on that endpoint's resource_type**:

```python
from noonhelpers.v1 import pacl

pacl.check(
    permission="namshi_bw.merch.list_requests",
    principal=ctx.user_code,        # ← verify field from Context
    resource=ctx.project_code,      # ← verify field from Context
    conditions={
        "merch_type": ctx.merch_type,   # ← registry defines this on resource_type `merch`
    },
)
```

Omit `conditions` when the resource_type defines none — the common case.

**Sudoer is the exception, not the default add.** `ctx.sudoer` is populated by
default from the `x-forwarded-sudoer` header, so a normal check needs nothing.
Reach for `is_sudoer` only to **restrict a critical/sensitive action** from an
impersonating internal employee. The codebase does this two ways — match the
existing pattern in the target service, do not invent a stance:

```python
# Block the impersonator outright (create org/project, ownership changes):
assert not ctx.sudoer, "The impersonated user is unable to create a project."

# Or evaluate the permission as the real user, excluding sudo grants:
pacl.check(permission="platform.project.update", principal=ctx.user_code,
           resource=ctx.project_code, conditions={"is_sudoer": False})
```

(`conditions={"is_sudoer": True}` is the other direction — it checks whether the
caller hitting the endpoint is a sudoer at all. Don't reach for it unless that's
what the endpoint is doing.)

Where a defined condition's value comes from (`ctx` field vs request body vs path
param) is a per-endpoint read; if it is not obvious, surface it as a blocker
rather than guessing.

If `ACLPermissionError` is not registered in the partner app, output the handler
addition for that codebase:

```python
# src/apppartners/web.py — after app is created:
from noonhelpers.v1.pacl import ACLPermissionError
errhandler_pacl = fastapiutil.generate_exception_handler(
    403, client_error_message=lambda exc: exc.message
)
app.add_exception_handler(ACLPermissionError, errhandler_pacl)
```

---

## Questions to ask

Ask only what blocks a correct registry patch or enforcement plan:

- namespace cannot be resolved
- role names are product-specific and cannot be inferred safely
- a condition/policy signal needs confirmation before defining it (Flow A)
- a defined condition's value source is unclear at the call site (Flow B)
- a critical action's sudoer stance is unclear — should an impersonating internal
  employee be blocked from it, or not
- provided permissions conflict with endpoint behavior
- sensitive partner endpoints appear intentionally unprotected

Do not ask about anything the service code or the registry already answers.

---

## Final response shape

1. Flow run (A, B, or A→B) and namespace confirmed, with evidence.
2. Endpoint coverage summary — N partner-facing endpoints across M files.
3. **Flow A:** registry file created/patched, and the validation/plan command
   output. Permission review findings: must-fix / recommended / informational.
4. **Flow B:** `pacl.check()` insertion points — one block per endpoint, with
   conditions pulled from the registry.
5. Open questions or blockers — keep short; resolve as much as possible.

The registry PR is the registration path. Do not tell the user to contact
Partplat to register a namespace.
