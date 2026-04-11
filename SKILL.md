# /pacl-setup: PACL Roles & Permissions Setup

Crawls a codebase, extracts partner-facing endpoints, infers permissions,
asks targeted questions, and outputs a ready-to-use roles + permissions sheet
plus `pacl.check()` insertion points for codebases that don't have PACL yet.

---

## Usage

```
/pacl-setup [path]
```

- `path` — path to the codebase root (defaults to current working directory)
- Run from the root of any service codebase

---

## Step 1: Detect framework and find route files

```bash
CODEBASE="${1:-$(pwd)}"
echo "Scanning: $CODEBASE"

# Detect language/framework
if find "$CODEBASE" -name "*.py" | head -1 | grep -q "."; then
  LANG="python"
elif find "$CODEBASE" -name "*.ts" -o -name "*.js" | head -1 | grep -q "."; then
  LANG="node"
elif find "$CODEBASE" -name "*.go" | head -1 | grep -q "."; then
  LANG="go"
else
  LANG="unknown"
fi
echo "Language: $LANG"
```

Find all route files, **excluding team apps** (directories with "team" in name):

```bash
# Python/FastAPI — find views/routes files not in team app dirs
# Usually would contain Context.partners():

find "$CODEBASE/src" -name "*.py" \
  -not -path "*/appteam/*" \
  -not -path "*/app*team*/*" \
  -not -path "*/team/*" \
  | xargs grep -l "@router\.\|@app\." 2>/dev/null
```

For Node/Express:
```bash
find "$CODEBASE/src" -name "*.ts" -o -name "*.js" \
  | grep -v "team\|internal\|admin" \
  | xargs grep -l "router\.\(get\|post\|put\|delete\|patch\)\|app\.\(get\|post\|put\|delete\|patch\)" 2>/dev/null
```

---

## Step 2: Extract endpoints
For each route file found, extract all endpoints. For Python/FastAPI:
```bash
grep -rh "@router\.\(get\|post\|put\|delete\|patch\)\|@app\.\(get\|post\|put\|delete\|patch\)" \
  <route_files> \
  | grep -v "hc\|health\|swagger\|openapi\|docs" \
  | sort -u
```

Read each route file to get:
- HTTP method (GET, POST, PUT, DELETE, PATCH)
- Path (e.g. `/project/user/create`)
- Summary/description (from `summary=` or `tags=`)

**Filter out:**
- `/public/hc`, `/health` — healthchecks
- `/swagger`, `/openapi.json`, `/docs` — documentation
- Any route in an `appteam` or `*team*` app directory

---

## Step 3: Infer permissions from endpoints

Map each endpoint to a permission name using this convention:
`namespace.resource_type.action`

**Action mapping:**
| Method | Path ends with | Inferred action |
|--------|---------------|-----------------|
| GET/POST | `/list` or `/list/*` | `list` |
| GET/POST | `/get` or `/:id` | `get` |
| POST | `/create` | `create` |
| POST/PUT/PATCH | `/update` | `update` |
| DELETE/POST | `/delete` | `delete` |
| POST | `/upload` | `upload` |
| POST | `/download` | `download` |
| POST | `/approve` | `approve` |
| POST | `/reject` | `reject` |
| POST | `/cancel` | `cancel` |
| POST | `/submit` | `submit` |
| POST | `/publish` | `publish` |
AND SO ON. DOESN'T NEED TO BE EXPLICITLY ON THIS ACTION MAP. BUT YOU GET THE IDEA.

**Resource type:** extracted from the path segment before the action.
think of it as the core entity. those are examples but not necessary:
Example: `/project/user/create` → resource = `user`, action = `create`
Example: `/dsp/ad/list` → resource = `ad`, action = `list`

**Namespace:** detect from the codebase first — look for `namespace_code`, `NAMESPACE`, or similar constants. If the user provided a namespace as an argument, use that. If neither, infer it from the service name (e.g. `mp-invoice-api` → `finance` or `invoice`). Only ask if truly ambiguous.

## Step 4: Ask targeted questions

Ask only what can't be inferred from the code. Use AskUserQuestion for each.

**Q1 — Conditions** Do you want your endpoints to be accessed by sudoers?:
> If so, we should add conditions={"is_sudoer": True}) in the pacl.
**Q4 — Role names** (suggest, ask to confirm):
> Based on the endpoints I found, I suggest these roles: [list].
> Does this match the actors in your system, or should we rename/add/remove any?

---

## Step 5: Generate roles + permissions table

Based on inferred permissions and confirmed actors, produce:

### Hierarchy
Follow this standard pattern unless the codebase suggests otherwise:
- `owner` → full access including sensitive/destructive actions
- `admin` → operational access, no delete/sensitive
- `viewer` → read-only (list + get)

Assign permissions to roles following the principle:
- **viewer** = base role with only view stuff
- Each level up adds more write capabilities
- Hierarchy: owner → admin → viewer (viewer is the base, everyone inherits from it)

Note: this is the default. If the codebase has more roles or different actors, adjust accordingly.

### Output format (sheet-ready)

```
ROLES
child_role    role_name    role_desc    sort

<namespace>.<resource>.<child>    <namespace>.<resource>.owner    <desc>    1
<namespace>.<resource>.<child>    <namespace>.<resource>.admin    <desc>    2
...

PERMISSIONS
Permission    owner    admin    viewer
<namespace>.<resource>.delete    Y    N    N
<namespace>.<resource>.create    Y(inherited)    Y    N
...
```

Use `↑` or `Y (inherited)` for permissions inherited via hierarchy.
Use `Y` for permissions directly on that role.
Use `N` for denied.

---

## Step 6: Generate pacl.check() insertion points:

For each endpoint, output the exact `pacl.check()` call to insert, with:
- The function/class where it should go
- The permission string
- Whether to use `raise_permission_error=False` (for fallback checks)

Format:
```python
# File: src/apppartner/views/requests.py
# Endpoint: POST /requests/create
# Insert at the start of the domain execute() method:

from noonhelpers.v1 import pacl

pacl.check(
    principal=ctx.user_code,       # adapt: read the Context class to find the right user identifier (uid, user_code, partner_code, etc.)
    permission='<namespace>.request.create',
    resource=ctx.project_code,     # adapt: read the Context class to find the right resource key (project_code, partner_code, bu_code, etc.)
)
```

Also output the exception handler to add to `web.py` if missing:
```python
from noonhelpers.v1.pacl import ACLPermissionError
errhandler_pacl = fastapiutil.generate_exception_handler(403, client_error_message=lambda exc: exc.message)
app.add_exception_handler(ACLPermissionError, errhandler_pacl)
```

---

## Step 7: Output summary

Print:
1. **Roles & permissions table** (copy-paste into sheet)
2. **pacl.check() insertion points** (one per endpoint, with file + line context)
3. **Open questions** — anything that couldn't be inferred and wasn't answered

---

## CI Hook (optional)

If the user wants auto-update on PR, suggest adding this to CI:

```yaml
# .github/workflows/pacl-check.yml or cloudbuild step
- name: Check PACL coverage
  run: |
    # Re-run pacl-setup and diff against current sheet
    # Flag any new endpoints missing pacl.check()
    grep -r "@router\.\|@app\." src/ \
      --include="*.py" \
      -not -path "*/appteam/*" \
      | grep -v "hc\|health\|swagger" \
      > /tmp/current_endpoints.txt
    # Compare against last known state
    diff /tmp/known_endpoints.txt /tmp/current_endpoints.txt || \
      echo "WARNING: New endpoints detected — run /pacl-setup to update permissions"
```

Store `known_endpoints.txt` in the repo after each `/pacl-setup` run so CI can diff.
