---
name: pacl-setup
version: 2.0.0
description: |
  PACL Roles & Permissions Setup. Crawls a service codebase, finds all
  partner-facing endpoints, infers or validates a permissions model, asks
  targeted clarifying questions, and outputs a ready-to-use roles/permissions
  sheet plus exact pacl.check() insertion points.
  Use when asked to "set up PACL", "add authorization", "wire up pacl",
  "implement these permissions", or "what permissions do I need".
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - AskUserQuestion
---

# /pacl-setup — PACL Roles & Permissions Setup

Crawls a codebase, extracts partner-facing endpoints, infers or validates a
permissions model, asks the minimum questions needed, and hands back a
ready-to-use roles + permissions table and `pacl.check()` insertion points.

**Usage:** `/pacl-setup [path]`  
Defaults to the current working directory.

---

## Step 1 — Scan the codebase

```bash
CODEBASE="${1:-$(pwd)}"
echo "Scanning: $CODEBASE"
```

Language: Assume its always going to be FastAPI.

Find **partner-facing** route files only — skip team/admin/internal apps:
To detect/confirm this endpoint is partner-facing, check if context handles any of user_code/project_code/id_partner/partner_code, etc. 
```bash
# Python / FastAPI
find "$CODEBASE/src" -name "*.py" \
  -not -path "*/appteam/*" \
  -not -path "*/appadmin/*" \
  -not -path "*/app*team*/*" \
  -not -path "*/team/*" \
  | xargs grep -l "@router\.\|@app\." 2>/dev/null
```

---

## Step 2 — Extract endpoints

Read each route file. For every endpoint capture:
- HTTP method
- Path (e.g. `/invoice/list`)
- Summary or tags if present

**Skip:** healthchecks (`/hc`, `/health`), docs (`/swagger`, `/openapi`, `/docs`).

For Python/FastAPI, greping like this helps as a first pass:

```bash
grep -rh "@router\.\(get\|post\|put\|delete\|patch\)" <route_files> \
  | grep -v "hc\|health\|swagger\|openapi\|docs" \
  | sort -u
```

Then **read the full files** — don't rely on grep alone. You need the function bodies to understand context.

---

## Step 3 — Detect namespace

Look for these signals in order, stop at the first hit:

1. A constant named `NAMESPACE`, `namespace_code`, or similar in the codebase
2. A namespace argument passed by the user (`/pacl-setup myns`)
3. Infer from the repo name — `mp-invoice-api` → `invoice`, `mp-catalog-api` → `catalog`

Only ask the user if none of these yield a clear answer.

---

## Step 4 — Determine operating mode

Before inferring permissions, decide which mode applies.

Use **implementation mode** if the user provides an existing roles/permissions
matrix, permission list, SQL seed data, CSV, JSON, spreadsheet, or says they
already have permissions and only want PACL implemented.

Use **design mode** if the user asks what permissions they need, has no matrix,
or wants the model created from the endpoint scan.

In implementation mode:
- Treat the provided permissions as the source of truth.
- Do not rename, merge, split, or invent permissions unless the user approves.
- Map each partner-facing endpoint to the closest provided permission.
- If a provided permission looks wrong, missing, overly broad, duplicated,
  inconsistent, or unsafe, report it clearly before or alongside the insertion
  points.
- Ask only for ambiguities that block implementation.

---

## Step 5 — Infer or validate permissions

### Design mode

Map every endpoint to `namespace.resource_type.action`.

**Resource type** = the core entity in the path, usually the segment before the action verb.  
`/invoice/item/create` → resource = `item`  
`/dsp/ad/list` → resource = `ad`  
`/project/get` → resource = `project`

**Action** = verb derived from path or HTTP method:

| Path ends with / method | Action |
|------------------------|--------|
| `/list` | `list` |
| `/get`, `GET /{id}` | `get` |
| `/create` | `create` |
| `/update` | `update` |
| `/delete` | `delete` |
| `/approve` | `approve` |
| `/reject` | `reject` |
| `/submit` | `submit` |
| `/publish` | `publish` |
| `/upload` | `upload` |
| `/download` | `download` |
| anything else | use the last path segment as-is |

### Implementation mode validation

When the user provides existing permissions, validate them against discovered
endpoints:

- **Missing coverage**: endpoint has no matching permission.
- **Unused permissions**: permission has no discovered endpoint.
- **Over-broad mapping**: multiple sensitive actions share one generic
  permission.
- **Unsafe grants**: destructive/sensitive actions appear available to
  low-privilege roles.
- **Naming drift**: permission namespace/resource/action does not match the
  service shape.
- **Duplicate/conflicting permissions**: same endpoint maps to multiple
  permissions.
- **Hierarchy issues**: role inheritance is reversed, cyclic, or grants more
  than intended.
- **Condition gaps**: sudoer/impersonation behavior is unclear or inconsistent.

Do not block implementation for non-critical concerns. Separate findings into:
- **Must fix before implementation**
- **Recommended changes**
- **Informational notes**

---

## Step 6 — Ask targeted questions

Only ask what can't be inferred. Use AskUserQuestion.

**Q1 — Sudoer support**
> Some endpoints should be accessible by impersonated users (sudoers).
> Should I add `conditions={"is_sudoer": True}` to all pacl.check() calls,
> some, or none?

In implementation mode, ask this only if the provided permissions do not already
make sudoer/impersonation behavior clear.

**Q2 — Role confirmation**
> Based on the endpoints I found, I suggest these roles:
> **viewer** (read-only), **admin** (operational), **owner** (full access).
> Does this match your system, or should we rename/add/remove any?

Ask Q2 in design mode, suggest first, then confirm. In implementation mode,
preserve the user's provided roles unless there is ambiguity or a clear flaw.

Only ask follow-ups if something is genuinely ambiguous — don't ask about things you can figure out from the code.

---

## Step 7 — Output the roles & permissions table

### Role hierarchy

In design mode, use this default pattern and adjust if the codebase suggests
different actors:
- **viewer** — base role, `list` + `get` only
- **admin** → inherits viewer — everything except destructive/sensitive
- **owner** → inherits admin — full access including `delete` and sensitive actions

In implementation mode, output the provided table as the implementation source
of truth. Do not rewrite it silently. Add short annotations for any issues found
in validation.

### Table format

```
ROLES
child_role                          parent_role                        description                sort
──────────────────────────────────────────────────────────────────────────────────────────────────
<ns>.<resource>.admin               <ns>.<resource>.owner              Operational access         1
<ns>.<resource>.viewer              <ns>.<resource>.admin              Read-only access           2

PERMISSIONS
Permission                          owner         admin         viewer
──────────────────────────────────────────────────────────────────────
<ns>.<resource>.delete              ✓             –             –
<ns>.<resource>.create              ✓ (inh.)      ✓             –
<ns>.<resource>.update              ✓ (inh.)      ✓             –
<ns>.<resource>.list                ✓ (inh.)      ✓ (inh.)      ✓
<ns>.<resource>.get                 ✓ (inh.)      ✓ (inh.)      ✓
```

`✓ (inh.)` = granted through role hierarchy, not directly assigned.

---

## Step 8 — Output pacl.check() insertion points

For each endpoint, produce the exact call to insert. Read the Context class
in the codebase to find the right field names (`user_code`, `partner_code`, etc.)
rather than guessing.

```python
# ── File: src/apppartners/views/invoice.py ──────────────────────────────────
# Endpoint: POST /invoice/item/create
# Insert at the top of the domain execute() or view function body:

from noonhelpers.v1 import pacl

pacl.check(
    principal  = ctx.user_code,       # ← verify field name from Context
    permission = "invoice.item.create",
    resource   = ctx.project_code,    # ← verify field name from Context
    conditions = {"is_sudoer": True}, # ← remove if sudoers not needed
)
```

If `ACLPermissionError` isn't registered in `web.py`, output this addition too:

```python
# Add to src/apppartners/web.py — after app is created:
from noonhelpers.v1.pacl import ACLPermissionError
errhandler_pacl = fastapiutil.generate_exception_handler(
    403, client_error_message=lambda exc: exc.message
)
app.add_exception_handler(ACLPermissionError, errhandler_pacl)
```

---

## Step 9 — Final summary

Output in this order:

1. **Mode:** design mode or implementation mode
2. **Namespace confirmed:** `<ns>`
3. **Endpoints found:** N partner-facing endpoints across M files
4. **Permission review findings:** must-fix / recommended / informational
5. **Roles & permissions table** — inferred or provided, clearly labeled
6. **pacl.check() insertion points** — one block per endpoint
7. **Open questions** — anything still unresolved (keep this short; resolve as much as possible)

End with:
> ✅ Contact partplat (ymadboly@noon.com / ababar@noon.com) when you're ready to
> register the namespace and get it live.
