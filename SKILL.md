---
name: pacl-setup
version: 2.0.0
description: |
  PACL Roles & Permissions Setup. Crawls a service codebase, finds all
  partner-facing endpoints, infers a permissions model, asks targeted
  clarifying questions, and outputs a ready-to-use roles/permissions sheet
  plus exact pacl.check() insertion points.
  Use when asked to "set up PACL", "add authorization", "wire up pacl",
  or "what permissions do I need".
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - AskUserQuestion
---

# /pacl-setup — PACL Roles & Permissions Setup

Crawls a codebase, extracts partner-facing endpoints, infers a permissions
model, asks the minimum questions needed, and hands back a ready-to-use
roles + permissions table and `pacl.check()` insertion points.

**Usage:** `/pacl-setup [path]`  
Defaults to the current working directory.

---

## Step 1 — Scan the codebase

```bash
CODEBASE="${1:-$(pwd)}"
echo "Scanning: $CODEBASE"
```

Detect language:

```bash
if find "$CODEBASE" -name "*.py" -maxdepth 6 | head -1 | grep -q "."; then
  LANG="python"
elif find "$CODEBASE" -name "*.ts" -o -name "*.js" -maxdepth 6 | head -1 | grep -q "."; then
  LANG="node"
elif find "$CODEBASE" -name "*.go" -maxdepth 6 | head -1 | grep -q "."; then
  LANG="go"
else
  LANG="unknown"
fi
echo "Language: $LANG"
```

Find **partner-facing** route files only — skip team/admin/internal apps:

```bash
# Python / FastAPI
find "$CODEBASE/src" -name "*.py" \
  -not -path "*/appteam/*" \
  -not -path "*/appadmin/*" \
  -not -path "*/app*team*/*" \
  -not -path "*/team/*" \
  | xargs grep -l "@router\.\|@app\." 2>/dev/null

# Node / Express
find "$CODEBASE/src" \( -name "*.ts" -o -name "*.js" \) \
  | grep -Ev "team|internal|admin" \
  | xargs grep -l "router\.\(get\|post\|put\|delete\|patch\)\|app\.\(get\|post\|put\|delete\|patch\)" 2>/dev/null
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

## Step 4 — Infer permissions

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

---

## Step 5 — Ask targeted questions

Only ask what can't be inferred. Use AskUserQuestion.

**Q1 — Sudoer support**
> Some endpoints should be accessible by impersonated users (sudoers).
> Should I add `conditions={"is_sudoer": True}` to all pacl.check() calls,
> some, or none?

**Q2 — Role confirmation** (always ask — suggest first, confirm)
> Based on the endpoints I found, I suggest these roles:
> **viewer** (read-only), **admin** (operational), **owner** (full access).
> Does this match your system, or should we rename/add/remove any?

Only ask follow-ups if something is genuinely ambiguous — don't ask about things you can figure out from the code.

---

## Step 6 — Output the roles & permissions table

### Role hierarchy

Default pattern (adjust if the codebase suggests different actors):
- **viewer** — base role, `list` + `get` only
- **admin** → inherits viewer — everything except destructive/sensitive
- **owner** → inherits admin — full access including `delete` and sensitive actions

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

## Step 7 — Output pacl.check() insertion points

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

## Step 8 — Final summary

Output in this order:

1. **Namespace confirmed:** `<ns>`
2. **Endpoints found:** N partner-facing endpoints across M files
3. **Roles & permissions table** — copy-paste ready
4. **pacl.check() insertion points** — one block per endpoint
5. **Open questions** — anything still unresolved (keep this short; resolve as much as possible)

End with:
> ✅ Contact partplat (ymadboly@noon.com / ababar@noon.com) when you're ready to
> register the namespace and get it live.
