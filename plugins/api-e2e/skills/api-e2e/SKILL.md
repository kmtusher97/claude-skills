---
name: api-e2e
description: Analyze branch changes, identify affected API endpoints across any framework, build a test plan, prompt for auth tokens, execute all tests with curl, and report a pass/fail summary
user-invocable: true
allowed-tools: Bash, Read, Glob, Grep
---

# API E2E Test & Verify

Analyze the current branch diff, discover every API endpoint that was added or modified, build a structured test plan, execute it against the running local server, and report a pass/fail summary.

Works with any backend framework — Express, Fastify, NestJS, FastAPI, Flask, Django, Rails, Go/chi, Go/gin, Laravel, Spring Boot, and others.

## Step 1 — Detect tech stack and base URL

Identify the tech stack:

```bash
# Detect by manifest files
ls package.json pyproject.toml Pipfile Gemfile go.mod pom.xml composer.json 2>/dev/null

# Node: peek at dependencies
cat package.json 2>/dev/null | python3 -c "
import sys, json
d = json.load(sys.stdin)
deps = list(d.get('dependencies', {}).keys())
print(deps[:20])
" 2>/dev/null

# Python: check framework
grep -i "fastapi\|flask\|django\|starlette" pyproject.toml Pipfile requirements*.txt 2>/dev/null | head -5

# Check for listening ports
ss -tlnp 2>/dev/null | grep LISTEN | head -10
```

Find the configured port:

```bash
# Search common entry-point files for PORT or listen()
find . -maxdepth 5 \( \
  -name "index.ts" -o -name "index.js" -o \
  -name "main.ts"  -o -name "main.py"  -o \
  -name "app.ts"   -o -name "app.py"   -o \
  -name "server.ts" -o -name "server.js" -o \
  -name "manage.py" \
\) 2>/dev/null | head -10 | xargs grep -n "PORT\|listen(" 2>/dev/null | head -10

# .env files
grep "PORT" .env .env.local .env.development .env.example 2>/dev/null | head -5
```

Use `http://localhost:<PORT>` as `$BASE_URL`. If the server is not reachable, stop here and tell the user.

## Step 2 — Find changed files on this branch

```bash
git diff origin/main...HEAD --name-only
# If $ARGUMENTS provided: git diff $ARGUMENTS...HEAD --name-only
```

Classify each changed file by its architectural role. The concepts are universal even if path conventions differ by framework:

| Role | What it does | Examples by stack |
|------|-------------|-------------------|
| **Route / URL config** | Declares HTTP method + path | `*routes*.ts`, `urls.py`, `routes.rb`, `*router.go`, `*Controller.java` |
| **Controller / Handler** | Handles request, calls services | `*controller*.ts`, `views.py`, `*_handler.go`, `*Resource.php` |
| **Service / Use-case** | Business logic | `*service*.ts`, `services.py`, `*_service.go` |
| **Schema / DTO / Serializer** | Request/response shape & validation | `*.schema.ts`, `serializers.py`, `*Dto.java`, `*Request.java` |
| **Client / Frontend** | Makes HTTP calls to the API | `*.tsx`, `*.jsx`, `*.vue`, `*.svelte` |

Trace service → controller → route to determine which HTTP endpoints are actually affected.

## Step 3 — Extract affected endpoints

Read each changed route/controller file and extract endpoint definitions. Use the patterns below for the detected stack:

**Node.js — Express / Fastify / Hono**
```
router\.(get|post|put|patch|delete)\s*\(
app\.(get|post|put|patch|delete)\s*\(
```

**Node.js — NestJS decorators**
```
@(Get|Post|Put|Patch|Delete)\(
@Controller\(
```

**Python — FastAPI / Flask**
```
@(app|router)\.(get|post|put|patch|delete)\(
```

**Python — Django**
```
path\(|re_path\(|url\(   # in urls.py
```

**Ruby — Rails**
```
(get|post|put|patch|delete|resources|resource)\s   # in routes.rb
```

**Go — chi / gin / echo**
```
\.(Get|Post|Put|Patch|Delete)\(
r\.(GET|POST|PUT|PATCH|DELETE)\(
```

**Java / Kotlin — Spring Boot**
```
@(GetMapping|PostMapping|PutMapping|PatchMapping|DeleteMapping|RequestMapping)
```

**PHP — Laravel**
```
Route::(get|post|put|patch|delete)\(
```

Build a table of all affected endpoints:

| Method | Path | Auth required | Roles / Scopes | Description |
|--------|------|---------------|----------------|-------------|

Also note:
- Which endpoints are role-restricted vs public
- Validation middleware or schema decorators (determines expected 400/422 shape)
- Any state-mutating operations that need sequencing

## Step 4 — Prompt for auth token

Output this to the user and wait:

> **Auth token needed.** Please open your browser DevTools → Network tab, make any authenticated request to the local server, copy the `Authorization: Bearer <token>` value from the request headers, and paste it here.
>
> - If the API uses a different auth scheme (cookie, API key header, session), paste that value and label the header name.
> - If you need tokens for **multiple roles** (e.g. admin vs. regular user), paste each one labeled separately.

Store tokens as `$TOKEN_<ROLE>` (e.g. `$TOKEN_ADMIN`, `$TOKEN_USER`). If only one role exists, use `$TOKEN`.

## Step 5 — Build the test plan

For each endpoint, generate test cases. Adapt auth mechanism to what the API uses:

**Standard cases (always include)**
1. **Happy path** — valid request with correct auth → expect 2xx
2. **Auth missing** — no auth header/cookie → expect 401 (or 302 for session-based)
3. **Wrong role** — lower-privilege token hitting a restricted endpoint → expect 403
4. **Invalid body** — missing required fields or wrong types → expect 400 or 422

**For state-mutating endpoints (POST / PUT / PATCH / DELETE)**
5. **State verification** — after mutation, issue a GET to confirm the change persisted
6. **Cleanup** — restore original state (soft-delete or re-issue the prior state)

**For ordering / reorder endpoints**
7. **Order preserved after add** — add item, verify it appears at the expected position
8. **Order preserved after update** — update item, verify position is unchanged
9. **Idempotent reorder** — apply the same order twice, verify result is stable

**For paginated / filtered endpoints**
10. **Empty result** — query that matches nothing → expect 200 with empty list
11. **Boundary values** — `page=0`, `limit=0`, very large offset

Present the full numbered checklist to the user, then ask:

> Ready to execute? (yes / skip \<number\> / abort)

## Step 6 — Execute tests

Run each test case sequentially using `curl`. Use `python3 -c` for JSON parsing (do **not** assume `jq` is installed).

Adapt the auth flag to the scheme in use:
- Bearer token → `-H "Authorization: Bearer $TOKEN"`
- API key header → `-H "X-API-Key: $TOKEN"` (or whichever header the API uses)
- Cookie / session → `-b "session=<value>"`

For each test case output:
- `✓ [N] <description>` on pass
- `✗ [N] <description>` on fail — show expected vs actual status code and relevant response body fields

**Token expiry:** if a request returns 401 unexpectedly (not a test specifically for auth failure), pause and ask for a fresh token, then retry from that test case.

**Cleanup:** after all tests, run any registered cleanup calls (soft-deletes, reverting state) even if earlier tests failed.

## Step 7 — Report

Print a summary table:

| # | Method | Endpoint | Scenario | Result |
|---|--------|----------|----------|--------|
| 1 | POST | /api/v1/... | Happy path | ✓ 201 |
| 2 | POST | /api/v1/... | Auth missing | ✓ 401 |
| 3 | GET  | /api/v1/... | Wrong role | ✓ 403 |

Final line: **`X/Y tests passed.`**

If any tests failed, list them with actual vs expected values and suggest likely causes.

## Rules

- Never hardcode tokens in output — always refer to them as `$TOKEN`, `$TOKEN_ADMIN`, etc.
- Always clean up test data created during the run
- If the server is not reachable, stop at Step 1 and tell the user
- Use `python3` for JSON parsing — never assume `jq` is available
- For sequenced tests (create → verify → cleanup), track created IDs and use them in subsequent steps
- Skip client file analysis if only server files changed, and vice versa
- If `$ARGUMENTS` is provided, treat it as the base ref to diff against instead of `origin/main`
- When the framework is ambiguous, read a sample route file before guessing — don't assume
