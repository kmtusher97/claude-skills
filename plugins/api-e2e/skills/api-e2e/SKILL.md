---
name: api-e2e
description: Analyze branch changes, identify affected API endpoints across any framework, build a test plan (saved to temp file), prompt for auth tokens, generate a Python test runner script (saved to temp file), execute it, and report a pass/fail summary
user-invocable: true
allowed-tools: Bash, Read, Glob, Grep
---

# API E2E Test & Verify

Analyze the current branch diff, discover every API endpoint that was added or modified, build a structured test plan (saved to a temp file), generate a Python test runner script (saved to a temp file), execute it against the running local server, and report a pass/fail summary.

Works with any backend framework — Express, Fastify, NestJS, FastAPI, Flask, Django, Rails, Go/chi, Go/gin, Laravel, Spring Boot, and others.

**You MUST follow the steps below in order. Do NOT skip any step. Each step produces output that later steps depend on.**

## Step 1 — Understand the project and service architecture

Before doing anything else, read available project documentation to understand the service topology, request routing, and inter-service communication:

```bash
# Read project-level CLAUDE.md, README, or architecture docs
cat CLAUDE.md README.md docs/ARCHITECTURE.md 2>/dev/null | head -500

# Check for docker-compose to understand service topology
cat docker-compose.yaml docker-compose.yml 2>/dev/null | head -200

# Check for API gateway or reverse proxy configs
find . -maxdepth 4 \( -name "*.yaml" -o -name "*.yml" -o -name "*.conf" \) | xargs grep -l "upstream\|proxy_pass\|routes\|gateway" 2>/dev/null | head -10

# Check for project-specific CLI tools
ls -1 *-cli* cli-* bin/ scripts/ Makefile 2>/dev/null
grep -iE "\bcli\b|wrapper|command.*(exec|run|build|logs)" CLAUDE.md README.md 2>/dev/null | head -10
```

Extract from the docs:
- **Service topology**: Which services exist, their ports, how requests are routed
- **API gateway prefixes**: Path prefixes added by API gateway/reverse proxy (e.g., `/service/api/*` routes to `service:8026`)
- **Auth mechanism**: How authentication works (tokens, cookies, API keys), what headers are required
- **Inter-service communication**: Event buses, message queues, webhook brokers, background workers
- **Database and cache infrastructure**: MySQL, PostgreSQL, Redis, etc.
- **Project CLI tools**: Check if the project has a custom CLI wrapper for running containers, executing commands, viewing logs, rebuilding services, etc. If one exists, **use it instead of raw `docker`/`docker-compose` commands** throughout all subsequent steps (e.g., `<project-cli> exec <service> ...` instead of `docker exec <service> ...`)

This context is critical for later steps — especially for constructing correct URLs, running commands in the right containers, and understanding async/event-driven behavior.

## Step 2 — Detect tech stack and base URL

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

Use `http://localhost:<PORT>` as `$BASE_URL`. If the project uses an API gateway/reverse proxy (discovered in Step 1), use the gateway URL and include path prefixes accordingly. If the server is not reachable, stop here and tell the user.

## Step 3 — Find changed files on this branch

First, determine the correct base ref for diffing. If `$ARGUMENTS` is provided, use that as the base ref. Otherwise, detect the default remote's main branch:

```bash
# Detect the default remote and main branch
DEFAULT_REMOTE=$(git remote | head -1)
MAIN_REF="$DEFAULT_REMOTE/main"
# Verify it exists
git rev-parse --verify "$MAIN_REF" 2>/dev/null || MAIN_REF="$DEFAULT_REMOTE/master"
git rev-parse --verify "$MAIN_REF" 2>/dev/null || echo "ERROR: Could not find main branch"
echo "Using base ref: $MAIN_REF"
```

If the base ref cannot be resolved, **ask the user** which remote/branch to diff against (e.g., `opal/main`, `upstream/main`).

```bash
git diff "$MAIN_REF"...HEAD --name-only
```

Classify each changed file by its architectural role. The concepts are universal even if path conventions differ by framework:

| Role | What it does | Examples by stack |
|------|-------------|-------------------|
| **Route / URL config** | Declares HTTP method + path | `*routes*.ts`, `urls.py`, `routes.rb`, `*router.go`, `*Controller.java` |
| **Controller / Handler** | Handles request, calls services | `*controller*.ts`, `views.py`, `*_handler.go`, `*Resource.php` |
| **Service / Use-case** | Business logic | `*service*.ts`, `services.py`, `*_service.go` |
| **Schema / DTO / Serializer** | Request/response shape & validation | `*.schema.ts`, `serializers.py`, `*Dto.java`, `*Request.java` |
| **Client / Frontend** | Makes HTTP calls to the API | `*.tsx`, `*.jsx`, `*.vue`, `*.svelte`, `*Service.ts`, `*Api.ts` |
| **Event handler / Worker** | Async processing, background jobs | `*handler*.py`, `*worker*.ts`, `*consumer*.py`, `*listener*.py` |
| **Config / Infrastructure** | Rate limits, queues, Redis, caching | `*config*.py`, `*settings*.py`, `*middleware*.ts` |

Trace service → controller → route to determine which HTTP endpoints are actually affected.

## Step 4 — Classify the nature of changes (MANDATORY — do not skip)

**STOP. Before extracting endpoints or building any test plan, you MUST analyze the actual diff content and classify the changes.** This step determines whether you need simple CRUD tests, behavioral tests, load tests, or all of them. Skipping this step will produce an incomplete test plan.

Read the diff content:

```bash
# Diff summary
git diff $MAIN_REF...HEAD --stat

# Read the actual code changes in key files (services, handlers, infrastructure)
git diff $MAIN_REF...HEAD -- <key-changed-files> | head -500

# Scan for behavioral indicators across the entire diff
git diff $MAIN_REF...HEAD | grep -iE "redis|INCR|DECR|semaphore|acquire|release|event.handler|webhook|queue|worker|rate.limit|concurrency|throttl|consumer|publish|subscribe|lock|mutex|slot" | head -30
```

Now fill in this classification table. **You MUST print this table in your output with YES or NO for each row:**

```
Change Classification:
| Category                       | Detected? | Evidence                        | Required test parts |
|--------------------------------|-----------|---------------------------------|---------------------|
| New CRUD endpoints             | YES / NO  | <what you found or "none">      | → Part A            |
| Business logic changes         | YES / NO  | <what you found or "none">      | → Part B            |
| Rate limiting / throttling     | YES / NO  | <what you found or "none">      | → Part C            |
| Async / event-driven flows     | YES / NO  | <what you found or "none">      | → Part B            |
| Concurrency control            | YES / NO  | <what you found or "none">      | → Part C            |
| Infrastructure / config        | YES / NO  | <what you found or "none">      | → Part D            |
| Database schema changes        | YES / NO  | <what you found or "none">      | → Part A            |
```

**Indicators to look for:**

| Category | What to grep/look for in the diff |
|----------|----------------------------------|
| **New CRUD endpoints** | New `@router.get/post/put/delete`, new route definitions, new controller methods |
| **Business logic changes** | Modified service methods, new domain validation rules, changed handler logic |
| **Rate limiting / throttling** | `redis`, `INCR`, `DECR`, `rate_limit`, `throttle`, `window`, `counter` |
| **Async / event-driven flows** | `event_handler`, `webhook`, `consumer`, `publish`, `subscribe`, `queue`, `worker`, `broker` |
| **Concurrency control** | `semaphore`, `acquire`, `release`, `lock`, `mutex`, `slot`, `concurrency` |
| **Infrastructure / config** | New settings, env vars, middleware changes, caching logic |
| **Database schema** | New migrations, model field additions, index changes |

**MANDATORY RULE:** Every row marked YES in the "Detected?" column means the corresponding test Part (A/B/C/D) MUST appear in the test plan at Step 8. If you skip any required Part, your test plan is wrong.

## Step 5 — Discover endpoints from client/frontend code

Search for **client-side code** that calls the API. Frontend service files, API client modules, and SDK wrappers reveal the exact endpoints, request/response types, required headers, and gateway path prefixes.

```bash
# Find frontend/client API service files
find . -maxdepth 6 \( \
  -name "*Service.ts" -o -name "*service.ts" -o \
  -name "*Api.ts" -o -name "*api.ts" -o \
  -name "*Client.ts" -o -name "*client.ts" -o \
  -name "*Repository.ts" -o -name "*repository.ts" \
\) 2>/dev/null | head -20

# Search for fetch/axios calls referencing changed API paths
grep -rn "fetch\|axios\|httpClient\|apiClient\|\$http" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" 2>/dev/null | grep -i "<relevant-path-keyword>" | head -20
```

For each client file found:

1. **Extract the exact API paths** — including any gateway/proxy prefixes the client adds (e.g., `/hypatia/api/v1/...` instead of just `/api/v1/...`)
2. **Extract request/response types** — TypeScript interfaces, Zod schemas, PropTypes, or inline type annotations that define the expected payload shapes
3. **Note HTTP client configuration** — base URL, default headers (auth tokens, content-type, custom headers like `x-opal-*`), interceptors, error handling
4. **Cross-reference with server endpoints** — the client code tells you the "real" URL the browser hits; use this for curl commands

If client code reveals endpoints not found in server-side route files (or vice versa), include both — gaps between client and server are themselves worth testing.

## Step 6 — Extract affected endpoints from server code

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

| Method | Path (as seen by client) | Auth required | Roles / Scopes | Description |
|--------|--------------------------|---------------|----------------|-------------|

Also note:
- Which endpoints are role-restricted vs public
- Validation middleware or schema decorators (determines expected 400/422 shape)
- Any state-mutating operations that need sequencing

## Step 7 — Prompt for auth token

Output this to the user and wait:

> **Auth token needed.** Please open your browser DevTools → Network tab, make any authenticated request to the local server, copy the `Authorization: Bearer <token>` value from the request headers, and paste it here.
>
> - If the API uses a different auth scheme (cookie, API key header, session), paste that value and label the header name.
> - If you need tokens for **multiple roles** (e.g. admin vs. regular user), paste each one labeled separately.
> - If the curl command includes other required headers (e.g., `x-opal-instance-id`), include those too.

Store tokens as `$TOKEN_<ROLE>` (e.g. `$TOKEN_ADMIN`, `$TOKEN_USER`). If only one role exists, use `$TOKEN`. Also store any additional required headers.

## Step 8 — Verify completeness checklist, then build the test plan

**Before writing any test cases, verify you have completed all prior steps.** Print this checklist:

```
Pre-test-plan checklist:
  [x] Step 1: Read project docs — <what you found: service topology, CLI tools, etc.>
  [x] Step 2: Detected tech stack — <framework, port, base URL>
  [x] Step 3: Found changed files — <N files changed>
  [x] Step 4: Classified changes — <list which categories were YES>
  [x] Step 5: Checked client code — <what you found or "no client changes">
  [x] Step 6: Extracted endpoints — <N endpoints found>
  [x] Step 7: Got auth token — <yes/pending>

  Required test plan parts based on Step 4 classification:
  - Part A (CRUD): <YES/NO>
  - Part B (Behavioral/async): <YES/NO>
  - Part C (Load/concurrency): <YES/NO>
  - Part D (Infrastructure): <YES/NO>
```

If any step shows `[ ]` (not completed), go back and complete it before proceeding.

Now build the test plan. **Include every Part that is marked YES above.**

### Part A — Endpoint CRUD tests (always include for new/changed endpoints)

For each endpoint, generate test cases:

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

### Part B — Behavioral / async flow tests (when Step 4 classified business logic, async, or event-driven changes as YES)

When the diff reveals changes to event handlers, background workers, webhook consumers, or async state machines, design tests that:

1. **Trigger the async flow** — make the API call that initiates the event/workflow
2. **Monitor for completion** — write a polling/monitoring script that checks the expected outcome:
   - Query the database for expected state transitions
   - Check Redis for expected key changes
   - Poll a status endpoint until completion or timeout
   - Read service logs for expected log entries
3. **Verify the end state** — confirm the final state matches expectations

These tests are implemented in the Python test runner script as polling functions. Example pattern (included in the generated script):
```python
def test_N_poll():
    """Poll for async completion after triggering the flow."""
    # First trigger the async flow
    status, body, raw = make_request("POST", "/trigger-endpoint",
                                      headers=auth_header("$TOKEN"), body={...})
    # Then poll for completion
    max_attempts = 10
    interval = 2  # seconds
    for attempt in range(1, max_attempts + 1):
        status, body, raw = make_request("GET", "/status-endpoint",
                                          headers=auth_header("$TOKEN"))
        if body.get("status") == "completed":
            record(N, "Async flow completed", True)
            return
        time.sleep(interval)
    record(N, "Async flow completed", False, f"timed out after {max_attempts * interval}s")
```

### Part C — Load / concurrency tests (when Step 4 classified rate limiting or concurrency control as YES)

When the diff reveals rate limiting, concurrency semaphores, or throttling logic:

1. **Design batched concurrent requests** — these are implemented in the Python test runner using `ThreadPoolExecutor`:
   ```python
   def test_N_concurrent():
       batch_size = 5
       def fire():
           return make_request("POST", "/endpoint", headers=auth_header("$TOKEN"), body={...})
       with ThreadPoolExecutor(max_workers=batch_size) as pool:
           futures = [pool.submit(fire) for _ in range(batch_size)]
           for f in as_completed(futures):
               status, body, raw = f.result()
               # tally accepted vs rejected
   ```

2. **Monitor rate limiting behavior** — check that the system correctly:
   - Accepts requests within the limit
   - Rejects/queues requests that exceed the limit
   - Returns appropriate error codes (429 Too Many Requests, or custom reschedule responses)

3. **Verify recovery** — after the rate limit window passes, confirm new requests are accepted

4. **Check infrastructure state** — examine Redis counters, queue depths, or database state to confirm the limiting mechanism is working. Use separate bash commands for infrastructure checks that cannot be done via HTTP:
   ```bash
   # Check Redis counters (if accessible)
   docker exec <redis-container> redis-cli GET "rate_limit_key:*"

   # Check database state
   docker exec <db-container> mysql -u... -p... -e "SELECT count(*) FROM ... WHERE status='queued'"
   ```

### Part D — Infrastructure observability checks (when Step 4 classified infrastructure changes as YES)

When the system uses Redis, message queues, or other infrastructure that the changes touch:

1. **Pre-test baseline** — capture initial state of relevant infrastructure
2. **Post-test verification** — confirm expected state changes occurred
3. **Log verification** — check service logs for expected entries (rate limit hits, queue operations, etc.)

```bash
# Example: Check service logs for expected patterns
docker logs <service> --since="<start_time>" 2>&1 | grep -c "rate_limit_exceeded"
docker logs <service> --since="<start_time>" 2>&1 | grep -c "concurrency_slot_acquired"
```

### Save the test plan to a temporary file

After building the complete test plan, save it as a Markdown table to a temporary file. The file path MUST be `/tmp/api-e2e-test-plan-<timestamp>.md` where `<timestamp>` is the current Unix timestamp.

The test plan file MUST contain a table with these columns:

```markdown
# API E2E Test Plan
Generated: <ISO 8601 timestamp>
Base URL: <$BASE_URL>
Branch: <current branch name>

## Test Cases

| # | Part | Method | Endpoint | Scenario | Expected Status | Auth | Request Body | Depends On | Cleanup |
|---|------|--------|----------|----------|-----------------|------|--------------|------------|---------|
| 1 | A    | POST   | /api/v1/... | Happy path | 201 | $TOKEN | {"key": "value"} | — | DELETE /api/v1/.../{{id}} |
| 2 | A    | POST   | /api/v1/... | Auth missing | 401 | none | {"key": "value"} | — | — |
| 3 | A    | POST   | /api/v1/... | Invalid body | 400 | $TOKEN | {} | — | — |
| 4 | B    | POST   | /api/v1/... | Trigger async flow | 202 | $TOKEN | {"key": "value"} | — | — |
| 5 | B    | GET    | /api/v1/.../status | Poll completion | 200 | $TOKEN | — | 4 | — |
| 6 | C    | POST   | /api/v1/... | Concurrent batch (5x) | 429 (some) | $TOKEN | {"key": "value"} | — | — |
```

Column definitions:
- **#**: Sequential test number
- **Part**: Which test plan part (A/B/C/D)
- **Method**: HTTP method
- **Endpoint**: Full path as seen by client (including gateway prefix)
- **Scenario**: Brief description of what is being tested
- **Expected Status**: Expected HTTP status code(s)
- **Auth**: Which token variable to use, or `none` for auth-missing tests
- **Request Body**: JSON payload (use `—` for GET requests with no body)
- **Depends On**: Test number this depends on (for sequenced tests), or `—`
- **Cleanup**: Cleanup action to run after this test, or `—`

Write this file using `python3`:

```bash
python3 -c "
import os, time
plan = '''<the full markdown table content>'''
path = f'/tmp/api-e2e-test-plan-{int(time.time())}.md'
with open(path, 'w') as f:
    f.write(plan)
print(f'Test plan saved to: {path}')
"
```

Print the test plan table to the user, then tell them the file path, and ask:

> Test plan saved to `<path>`. Ready to execute? (yes / skip \<number\> / abort)

## Step 9 — Generate and execute the Python test runner

**Do NOT run curl commands directly.** Instead, generate a Python test runner script, save it to `/tmp/api-e2e-runner-<timestamp>.py`, and execute it.

The script MUST:
- Use only `urllib.request` and `urllib.error` from the standard library (do **not** assume `requests` is installed)
- Accept auth tokens and base URL via environment variables
- Read the test plan from the saved file (Step 8)
- Execute tests sequentially, respecting `Depends On` ordering
- Track created resource IDs for use in dependent tests and cleanup
- Print `✓ [N] <description>` on pass and `✗ [N] <description>` on fail with expected vs actual details
- Run cleanup actions even if earlier tests failed
- For behavioral tests (Part B): implement polling loops with configurable timeout and interval
- For load/concurrency tests (Part C): use `concurrent.futures.ThreadPoolExecutor` to send parallel batches
- Exit with code 0 if all tests pass, 1 if any fail
- Print a final summary table to stdout

Generate the script with this structure:

```python
#!/usr/bin/env python3
"""API E2E Test Runner — auto-generated"""

import json
import os
import sys
import time
import urllib.request
import urllib.error
from concurrent.futures import ThreadPoolExecutor, as_completed

BASE_URL = os.environ.get("BASE_URL", "<detected_base_url>")
TOKEN = os.environ.get("TOKEN", "")
TOKEN_ADMIN = os.environ.get("TOKEN_ADMIN", "")
TOKEN_USER = os.environ.get("TOKEN_USER", "")

# Additional required headers discovered in Steps 1 and 5
EXTRA_HEADERS = {
    # e.g. "x-custom-header": "value"
}

results = []       # list of (test_num, description, passed, detail)
created_ids = {}   # test_num -> created resource ID for dependent tests
cleanups = []      # list of (method, url, headers) to run at the end


def make_request(method, path, headers=None, body=None, timeout=30):
    """Make an HTTP request and return (status_code, response_body_dict, raw_body)."""
    url = f"{BASE_URL}{path}"
    data = json.dumps(body).encode() if body else None
    req = urllib.request.Request(url, data=data, method=method)
    req.add_header("Content-Type", "application/json")
    if headers:
        for k, v in headers.items():
            req.add_header(k, v)
    for k, v in EXTRA_HEADERS.items():
        req.add_header(k, v)
    try:
        resp = urllib.request.urlopen(req, timeout=timeout)
        raw = resp.read().decode()
        try:
            parsed = json.loads(raw)
        except (json.JSONDecodeError, ValueError):
            parsed = {}
        return resp.status, parsed, raw
    except urllib.error.HTTPError as e:
        raw = e.read().decode() if e.fp else ""
        try:
            parsed = json.loads(raw)
        except (json.JSONDecodeError, ValueError):
            parsed = {}
        return e.code, parsed, raw


def auth_header(token_var):
    """Build auth header dict from token variable name."""
    token = {"$TOKEN": TOKEN, "$TOKEN_ADMIN": TOKEN_ADMIN, "$TOKEN_USER": TOKEN_USER}.get(token_var, "")
    if not token or token_var == "none":
        return {}
    return {"Authorization": f"Bearer {token}"}


def record(test_num, description, passed, detail=""):
    """Record a test result."""
    mark = "✓" if passed else "✗"
    print(f"  {mark} [{test_num}] {description}" + (f" — {detail}" if detail and not passed else ""))
    results.append((test_num, description, passed, detail))


def run_cleanup():
    """Run all registered cleanup actions."""
    if not cleanups:
        return
    print("\n--- Cleanup ---")
    for method, url, headers in cleanups:
        try:
            req = urllib.request.Request(url, method=method)
            req.add_header("Content-Type", "application/json")
            for k, v in (headers or {}).items():
                req.add_header(k, v)
            for k, v in EXTRA_HEADERS.items():
                req.add_header(k, v)
            resp = urllib.request.urlopen(req, timeout=30)
            print(f"  ✓ Cleanup {method} {url} → {resp.status}")
        except urllib.error.HTTPError as e:
            print(f"  ✗ Cleanup {method} {url} → {e.code}")
        except Exception as e:
            print(f"  ✗ Cleanup {method} {url} → {e}")


# ──────────────────────────────────────────────
# Test cases — fill in from the test plan table
# ──────────────────────────────────────────────

def test_N():
    """Test N: <description>"""
    status, body, raw = make_request(
        "<METHOD>", "<path>",
        headers=auth_header("<token_var>"),
        body=<body_dict_or_None>,
    )
    passed = status == <expected_status>
    record(N, "<description>", passed,
           f"expected {<expected_status>}, got {status}" if not passed else "")
    # If this test creates a resource, store its ID:
    # created_ids[N] = body.get("id")
    # If cleanup is needed, register it:
    # cleanups.append(("DELETE", f"{BASE_URL}<cleanup_path>/{body.get('id')}", auth_header("<token_var>")))

# --- Part B: Behavioral / async tests ---

def test_N_poll():
    """Test N: Poll for async completion"""
    max_attempts = 10
    interval = 2  # seconds
    for attempt in range(1, max_attempts + 1):
        status, body, raw = make_request("GET", "<status_path>", headers=auth_header("<token_var>"))
        if body.get("status") == "completed":
            record(N, "<description>", True)
            return
        time.sleep(interval)
    record(N, "<description>", False, f"timed out after {max_attempts * interval}s")

# --- Part C: Load / concurrency tests ---

def test_N_concurrent():
    """Test N: Concurrent batch request"""
    batch_size = 5
    accepted = 0
    rejected = 0

    def fire():
        return make_request("POST", "<path>", headers=auth_header("<token_var>"), body=<body>)

    with ThreadPoolExecutor(max_workers=batch_size) as pool:
        futures = [pool.submit(fire) for _ in range(batch_size)]
        for f in as_completed(futures):
            status, body, raw = f.result()
            if status in (200, 201, 202):
                accepted += 1
            else:
                rejected += 1

    expected_accepted = <N>
    expected_rejected = batch_size - expected_accepted
    passed = accepted == expected_accepted
    record(N, "<description>",
           passed, f"accepted={accepted} rejected={rejected}, expected accepted={expected_accepted}")


# ──────────────────────────────────────────────

if __name__ == "__main__":
    print(f"Running API E2E tests against {BASE_URL}\n")

    # Execute all test functions in order
    # test_1()
    # test_2()
    # ...

    # Always run cleanup
    run_cleanup()

    # Summary
    passed = sum(1 for _, _, p, _ in results if p)
    total = len(results)
    print(f"\n{'='*50}")
    print(f"  {passed}/{total} tests passed.")
    print(f"{'='*50}")
    sys.exit(0 if passed == total else 1)
```

**Fill in every `test_N` function** from the test plan table. Each row in the table becomes one function. For sequenced tests, use `created_ids[<depends_on>]` to reference IDs from earlier tests.

Save the script:

```bash
python3 -c "
script = '''<the full python script content>'''
path = '/tmp/api-e2e-runner-$(date +%s).py'
with open(path, 'w') as f:
    f.write(script)
print(f'Test runner saved to: {path}')
"
```

Tell the user the file path and show them the test runner script content for review.

After user approval, execute the script:

```bash
TOKEN="<user_provided_token>" \
TOKEN_ADMIN="<if_provided>" \
TOKEN_USER="<if_provided>" \
BASE_URL="<detected_base_url>" \
python3 /tmp/api-e2e-runner-<timestamp>.py
```

**Token expiry:** if the script reports unexpected 401s (not from auth-missing tests), pause and ask for a fresh token, then re-run the script with the new token.

**The script handles cleanup automatically** — cleanup actions registered during test execution run at the end even if earlier tests fail.

## Step 10 — Report

Print a summary table:

| # | Method | Endpoint | Scenario | Result |
|---|--------|----------|----------|--------|
| 1 | POST | /api/v1/... | Happy path | ✓ 201 |
| 2 | POST | /api/v1/... | Auth missing | ✓ 401 |
| 3 | GET  | /api/v1/... | Wrong role | ✓ 403 |

For behavioral/load tests, add a separate section:

### Behavioral Test Results
| # | Scenario | Triggered | Expected Outcome | Actual Outcome | Result |
|---|----------|-----------|------------------|----------------|--------|
| 1 | Rate limit enforced under load | 30 requests in 6 batches | 10 accepted, 20 rejected/queued | 10 accepted, 20 queued | ✓ |

Final line: **`X/Y tests passed.`**

If any tests failed, list them with actual vs expected values and suggest likely causes.

## Command permissions

**Run freely (no confirmation needed):**
- `git` — all read-only commands: `diff`, `log`, `show`, `branch`, `remote`, `rev-parse`, `status`
- `gh` — all read-only commands: `pr view`, `issue view`, `api` (GET)
- `docker` — read-only commands: `exec ... <read commands>`, `logs`, `ps`, `inspect`
- `python3 -c` — for JSON parsing and writing temp files
- `python3 /tmp/api-e2e-runner-*.py` — executing the generated test runner
- Project CLI tools (discovered in Step 1) — read-only equivalents of the above

**Ask the user before running (mutating commands):**
- The generated Python test runner contains mutating API calls — present the script content and test plan to the user and wait for approval before executing
- `docker exec` with write operations

**Auto-permit mode:** If the user responds with "yes to all", "auto-permit", or grants blanket approval, execute the test runner without further confirmation.

## Rules

- Never hardcode tokens in output or in saved files — always pass them via environment variables (`$TOKEN`, `$TOKEN_ADMIN`, etc.)
- Always clean up test data created during the run (the Python script handles this automatically)
- If the server is not reachable, stop at Step 2 and tell the user
- Use only Python standard library (`urllib.request`, `json`, `concurrent.futures`) — never assume third-party packages like `requests` or `jq` are installed
- Save the test plan to `/tmp/api-e2e-test-plan-<timestamp>.md` and the test runner to `/tmp/api-e2e-runner-<timestamp>.py`
- For sequenced tests (create → verify → cleanup), track created IDs in the `created_ids` dict and use them in dependent test functions
- If `$ARGUMENTS` is provided, treat it as the base ref to diff against instead of the auto-detected `$MAIN_REF`
- When the framework is ambiguous, read a sample route file before guessing — don't assume
- Always read project documentation (CLAUDE.md, README) before starting — service architecture context is essential
- When changes affect event handlers, workers, or async flows, behavioral tests are mandatory — not just endpoint CRUD tests
- Use client/frontend code as a primary source for discovering the correct API URLs, headers, and payload shapes
- Construct API URLs using the gateway/proxy prefix discovered from client code, not the raw server-side route path
