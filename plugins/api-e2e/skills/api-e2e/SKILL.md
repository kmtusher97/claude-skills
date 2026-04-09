---
name: api-e2e
description: Analyze branch changes, identify affected API endpoints across any framework, build a test plan, prompt for auth tokens, execute all tests with curl, and report a pass/fail summary
user-invocable: true
allowed-tools: Bash, Read, Glob, Grep
---

# API E2E Test & Verify

Analyze the current branch diff, discover every API endpoint that was added or modified, build a structured test plan, execute it against the running local server, and report a pass/fail summary.

Works with any backend framework — Express, Fastify, NestJS, FastAPI, Flask, Django, Rails, Go/chi, Go/gin, Laravel, Spring Boot, and others.

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

## Step 4 — Discover endpoints from client/frontend code

In addition to reading server-side route definitions, search for **client-side code** that calls the API. Frontend service files, API client modules, and SDK wrappers often reveal the exact endpoints, request/response types, required headers, and gateway path prefixes.

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

## Step 5 — Extract affected endpoints

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

## Step 6 — Analyze the semantic nature of changes (classify change type)

**THIS STEP IS CRITICAL — DO NOT SKIP IT.** The test plan quality depends entirely on correctly classifying what the changes do. Without this step, you will only generate basic CRUD tests and miss the most important behavioral/concurrency/async tests.

Analyze the **actual git diff content** (not just file names) to understand what the changes do:

```bash
# Read the actual diff to understand what changed
git diff $MAIN_REF...HEAD --stat
git diff $MAIN_REF...HEAD -- <key-changed-files> | head -500

# Look specifically for behavioral indicators in the diff
git diff $MAIN_REF...HEAD | grep -E "redis|INCR|DECR|semaphore|acquire|release|event_handler|webhook|queue|worker|rate.limit|concurrency|throttl" | head -30
```

Classify the changes into one or more categories. **You MUST output this classification table explicitly** before proceeding to Step 7:

| Category | Indicators in diff | Testing strategy | Applies to this branch? |
|----------|-------------------|------------------|------------------------|
| **New CRUD endpoints** | New route definitions, new controller methods | → Part A: endpoint CRUD tests | YES / NO |
| **Business logic changes** | Modified service/handler code, new domain rules | → Part B: behavioral verification | YES / NO |
| **Rate limiting / throttling** | Redis keys, counters, semaphores, `INCR`/`DECR` | → Part C: load/concurrency testing | YES / NO |
| **Async / event-driven flows** | Event handlers, webhook consumers, background workers, queues | → Part B: temporal testing | YES / NO |
| **Concurrency control** | Locks, semaphores, mutexes, slot acquire/release | → Part C: concurrent batches | YES / NO |
| **Infrastructure / config** | Settings, env vars, middleware, caching | → Part D: infrastructure checks | YES / NO |
| **Database schema** | Migrations, model changes | → Part A: state persistence | YES / NO |

**MANDATORY RULE:** If ANY category other than "New CRUD endpoints" is marked YES, you MUST include the corresponding test plan parts (B, C, or D). Generating only Part A (CRUD tests) when the diff contains event handlers, Redis operations, or concurrency logic is an incomplete test plan.

## Step 7 — Prompt for auth token

Output this to the user and wait:

> **Auth token needed.** Please open your browser DevTools → Network tab, make any authenticated request to the local server, copy the `Authorization: Bearer <token>` value from the request headers, and paste it here.
>
> - If the API uses a different auth scheme (cookie, API key header, session), paste that value and label the header name.
> - If you need tokens for **multiple roles** (e.g. admin vs. regular user), paste each one labeled separately.
> - If the curl command includes other required headers (e.g., `x-opal-instance-id`), include those too.

Store tokens as `$TOKEN_<ROLE>` (e.g. `$TOKEN_ADMIN`, `$TOKEN_USER`). If only one role exists, use `$TOKEN`. Also store any additional required headers.

## Step 8 — Build the test plan

Based on the change classification from Step 6, build an appropriate test plan. **Include every Part (A/B/C/D) that was marked YES in the Step 6 classification table.** If you only generate Part A when other categories were marked YES, your test plan is incomplete — go back and add the missing parts.

For example, if the diff contains both new API endpoints AND Redis concurrency semaphores AND event handler changes, your test plan MUST include Part A + Part B + Part C + Part D.

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

### Part B — Behavioral / async flow tests (include when changes affect event-driven or async processing)

When the diff reveals changes to event handlers, background workers, webhook consumers, or async state machines, design tests that:

1. **Trigger the async flow** — make the API call that initiates the event/workflow
2. **Monitor for completion** — write a polling/monitoring script that checks the expected outcome:
   - Query the database for expected state transitions
   - Check Redis for expected key changes
   - Poll a status endpoint until completion or timeout
   - Read service logs for expected log entries
3. **Verify the end state** — confirm the final state matches expectations

Example monitoring script pattern:
```bash
# Trigger the action
curl -s -X POST "$BASE_URL/trigger-endpoint" -H "Authorization: Bearer $TOKEN" -d '...'

# Monitor for completion (poll every N seconds, timeout after M seconds)
for i in $(seq 1 <max_attempts>); do
  result=$(curl -s "$BASE_URL/status-endpoint" -H "Authorization: Bearer $TOKEN")
  status=$(echo "$result" | python3 -c "import sys,json; print(json.load(sys.stdin).get('status',''))")
  if [ "$status" = "completed" ]; then
    echo "✓ Flow completed after ${i} checks"
    break
  fi
  sleep <interval>
done
```

### Part C — Load / concurrency tests (include when changes affect rate limiting, concurrency control, or throttling)

When the diff reveals rate limiting, concurrency semaphores, or throttling logic:

1. **Design batched concurrent requests** — send N requests in parallel batches with controlled delays:
   ```bash
   # Send batch of 5 concurrent requests
   for i in $(seq 1 5); do
     curl -s -X POST "$BASE_URL/endpoint" -H "Authorization: Bearer $TOKEN" -d '...' &
   done
   wait
   sleep <delay_between_batches>
   ```

2. **Monitor rate limiting behavior** — check that the system correctly:
   - Accepts requests within the limit
   - Rejects/queues requests that exceed the limit
   - Returns appropriate error codes (429 Too Many Requests, or custom reschedule responses)

3. **Verify recovery** — after the rate limit window passes, confirm new requests are accepted

4. **Check infrastructure state** — examine Redis counters, queue depths, or database state to confirm the limiting mechanism is working:
   ```bash
   # Check Redis counters (if accessible)
   docker exec <redis-container> redis-cli GET "rate_limit_key:*"

   # Check database state
   docker exec <db-container> mysql -u... -p... -e "SELECT count(*) FROM ... WHERE status='queued'"
   ```

### Part D — Infrastructure observability checks (include when changes affect Redis, queues, or background processing)

When the system uses Redis, message queues, or other infrastructure that the changes touch:

1. **Pre-test baseline** — capture initial state of relevant infrastructure
2. **Post-test verification** — confirm expected state changes occurred
3. **Log verification** — check service logs for expected entries (rate limit hits, queue operations, etc.)

```bash
# Example: Check service logs for expected patterns
docker logs <service> --since="<start_time>" 2>&1 | grep -c "rate_limit_exceeded"
docker logs <service> --since="<start_time>" 2>&1 | grep -c "concurrency_slot_acquired"
```

Present the full numbered checklist to the user, then ask:

> Ready to execute? (yes / skip \<number\> / abort)

## Step 9 — Execute tests

Run each test case sequentially using `curl`. Use `python3 -c` for JSON parsing (do **not** assume `jq` is installed).

Adapt the auth flag to the scheme in use:
- Bearer token → `-H "Authorization: Bearer $TOKEN"`
- API key header → `-H "X-API-Key: $TOKEN"` (or whichever header the API uses)
- Cookie / session → `-b "session=<value>"`
- Additional headers → include all required headers discovered in Steps 1 and 4

For each test case output:
- `✓ [N] <description>` on pass
- `✗ [N] <description>` on fail — show expected vs actual status code and relevant response body fields

**Token expiry:** if a request returns 401 unexpectedly (not a test specifically for auth failure), pause and ask for a fresh token, then retry from that test case.

**Cleanup:** after all tests, run any registered cleanup calls (soft-deletes, reverting state) even if earlier tests failed.

**For behavioral/load tests:** execute the monitoring scripts, capture their output, and include results in the report.

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
- `curl -s ... -X GET` or `curl -s ...` (GET is the default) — read-only API calls
- `python3 -c` — for JSON parsing
- Project CLI tools (discovered in Step 1) — read-only equivalents of the above

**Ask the user before running (mutating commands):**
- `curl -X POST/PUT/PATCH/DELETE` — these modify server state
- Present the curl command to the user and wait for approval before executing

**Auto-permit mode:** If the user responds with "yes to all", "auto-permit", or grants blanket approval for mutating requests, execute all remaining `curl POST/PUT/PATCH/DELETE` commands without asking for each one individually.

## Rules

- Never hardcode tokens in output — always refer to them as `$TOKEN`, `$TOKEN_ADMIN`, etc.
- Always clean up test data created during the run
- If the server is not reachable, stop at Step 2 and tell the user
- Use `python3` for JSON parsing — never assume `jq` is available
- For sequenced tests (create → verify → cleanup), track created IDs and use them in subsequent steps
- If `$ARGUMENTS` is provided, treat it as the base ref to diff against instead of the auto-detected `$MAIN_REF`
- When the framework is ambiguous, read a sample route file before guessing — don't assume
- Always read project documentation (CLAUDE.md, README) before starting — service architecture context is essential
- When changes affect event handlers, workers, or async flows, behavioral tests are mandatory — not just endpoint CRUD tests
- Use client/frontend code as a primary source for discovering the correct API URLs, headers, and payload shapes
- Construct curl URLs using the gateway/proxy prefix discovered from client code, not the raw server-side route path
