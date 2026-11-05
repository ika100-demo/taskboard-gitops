---
plan_id: 002-tasks-in-postgres-and-a-web-ui
spec: 002-tasks-in-postgres-and-a-web-ui
feature: Keep tasks in Postgres (task-api) and add a web UI that lists and adds tasks (taskboard-web),
  wired by the gitops-app
gitops_app: ika100-demo/taskboard-gitops
status: completed
repos:
- id: taskboard-gitops
  shape: gitops-app
  summary: Declare postgres for task-api, publish taskboard-web in dev and point it at task-api
  acs:
  - AC-002.1
  - AC-002.2
  - AC-002.7
  arguments: |
    Data only: /system:ship runs the gitops operations below with cplat resource add / cplat component set,
    then `devbox run render`. Do not touch replicas: task-api stays at 1 instance in dev.
  gitops:
  - resource: postgres
  - uses: postgres
    component: task-api
  - expose: taskboard-web
    host: taskboard-web
  - env:
      TASK_API_URL: http://task-api
    component: taskboard-web
  depends_on: []
  done: true
- id: task-api
  shape: service-python
  summary: Store tasks in Postgres instead of memory, answer 503 and report not ready while the database
    is down
  acs:
  - AC-002.1
  - AC-002.2
  - AC-002.3
  - AC-002.4
  - AC-002.5
  - AC-002.6
  arguments: |
    Replace the in-memory TaskStore (src/task_api/tasks/store.py) with a Postgres store behind the same
    add()/list() interface; keep router.py and models.py and the spec 001 tests (adapt only the store fixture).
    Driver: psycopg 3 with psycopg_pool (add "psycopg[binary]" and "psycopg-pool" through devbox/uv, never pip).
    Connection: read DATABASE_URL (set by the gitops-app from the postgres resource; PG* vars are also present).
    Schema: created by the service itself with idempotent DDL the first time a connection succeeds (not only
    at import/startup, so a pod started while the database is down recovers without a restart), e.g.
    tasks(seq bigint generated always as identity primary key, id uuid not null unique, title text not null,
    due_date date null). Order GET /tasks by seq; never by a timestamp.
    The process must start (and /health answer 200) while the database is down; only /ready and the task
    routes report it. The single-uvicorn-process constraint of spec 001 is lifted.
    Tests run against a real Postgres (testcontainers-python, Docker), not SQLite; the "database down" cases
    stop the container or point the pool at a closed port. Spec 001's in-memory restart test (AC-001.9) is
    superseded: delete it or turn it into the AC-002.1 test (new app instance, same database).
  depends_on:
  - taskboard-gitops
  done: true
- id: taskboard-web
  shape: web-nextjs
  summary: Taskboard page that lists all tasks and adds a task through task-api
  acs:
  - AC-002.7
  - AC-002.8
  - AC-002.9
  - AC-002.10
  - AC-002.11
  - AC-002.12
  - AC-002.13
  - AC-002.14
  - AC-002.15
  arguments: |
    Replace the template home page (app/page.tsx and its feature cards) with the board. The browser never
    calls task-api: the page is rendered on the server and the form posts to this app's own /api/tasks route
    handler, which calls task-api (see Contract). Read TASK_API_URL at request time with process.env on the
    server only (no NEXT_PUBLIC_ variable; the image is built once and configured per environment); default
    http://localhost:8080 for `devbox run dev`. Keep /api/health and /api/ready independent of task-api.
    Unit tests mock task-api with fetch mocks; Playwright e2e runs against a stub task-api server.
  depends_on: []
  done: true
gitops_pin: []
---

Three repos take part: `task-api` (persistence, AC-002.1 to AC-002.6), `taskboard-web` (the UI, AC-002.7 to
AC-002.15) and this gitops-app (the database, exposure and wiring the two need to work in dev). The task API
contract of spec 001 is unchanged except for the new `503`, so `taskboard-web` builds against the contract below in
parallel with `task-api`; it needs no `depends_on`.

Levels:

- Level 0: `taskboard-gitops` and `taskboard-web` (in parallel).
- Level 1: `task-api`, after `taskboard-gitops`. The dependency is real: dev follows `main`, so the database-backed
  `task-api` is deployed the moment its PR merges, and it can only become ready when the postgres resource exists
  and `DATABASE_URL` is in its environment. The reverse order is harmless: the current in-memory `task-api` ignores
  `DATABASE_URL`. Exposing `taskboard-web` and setting `TASK_API_URL` before the new web image exists is also
  harmless (the template page is served until then).

## taskboard-gitops — Declare postgres for task-api, publish taskboard-web in dev and point it at task-api

**Shape:** `gitops-app` · **Criteria:** AC-002.1, AC-002.2, AC-002.7 · **Depends on:** []

The gitops part is data, run in this order:

1. `{resource: postgres}`: declares the system's Postgres database (CloudNativePG cluster per environment, with the
   connection Secret `taskboard-postgres-app`). It is the store that outlives `task-api` restarts and redeploys
   (AC-002.1, AC-002.2) and the one list that every `task-api` instance shares (AC-002.5).
2. `{uses: postgres, component: task-api}`: gives `task-api` `DATABASE_URL`, `PGHOST`, `PGPORT`, `PGDATABASE`,
   `PGUSER`, `PGPASSWORD` from that Secret.
3. `{expose: taskboard-web, host: taskboard-web}`: publishes the web app through the Gateway in dev at
   `http://taskboard-web.taskboard-dev.localhost` (the "web app's root URL" of AC-002.7 onward).
4. `{env: {TASK_API_URL: 'http://task-api'}, component: taskboard-web}`: the in-namespace address of `task-api`
   (Service port 80), used only by the web app's server.

No replica change: the spec leaves the instance count a deployment setting, 1 in dev for now. AC-002.5 is proven in
the `task-api` repo with two app instances against one database; scaling dev to 2 is a later `system.yaml` change
that needs nothing else. Nothing for staging or prod: both components run only in dev (non-goal).

## task-api — Store tasks in Postgres instead of memory, answer 503 and report not ready while the database is down

**Shape:** `service-python` · **Criteria:** AC-002.1 to AC-002.6 · **Depends on:** [taskboard-gitops]

`task-api` owns the persistence: the store moves into the database, the HTTP contract stays as it is.

- AC-002.1, AC-002.2: tasks, their `id` (UUID4, generated by the service as before) and their order (`seq`) live in
  Postgres; nothing is kept in process memory between requests. Test with a second app instance on the same
  database (restart) and with a fresh pool plus a new create (redeploy; C's `id` differs from A's and B's).
- AC-002.3: a database error, a connection timeout or a pool timeout on `POST /tasks` or `GET /tasks` is answered
  `503` with the body in "Errors"; never `201`, never `[]`, never a `500`. The insert is one transaction: when it
  does not commit, nothing is stored. Validation still runs first: an invalid body is `422` even when the database
  is down (validation needs no database).
- AC-002.4: `/ready` runs `SELECT 1` on a pooled connection with a 2 s timeout on each call (no cached result) and
  answers `503` when it fails. The pool reconnects on its own; a pod that started with the database down becomes
  ready once it is up, within one probe period (kubelet default 10 s, well inside 30 s). `/health` stays `200`
  regardless, so Kubernetes does not restart the pod.
- AC-002.5: one shared table; test with two app instances (two `TestClient`s, two pools) on the same database.
- AC-002.6: spec 001's acceptance tests keep passing against the database store; the fixture truncates the table
  between tests instead of building a fresh in-memory store (AC-001.7 now means "empty table").

## taskboard-web — Taskboard page that lists all tasks and adds a task through task-api

**Shape:** `web-nextjs` · **Criteria:** AC-002.7 to AC-002.15 · **Depends on:** []

`taskboard-web` owns everything the user sees. Decided architecture: the browser talks only to the web app (same
origin); the web app's server talks to `task-api` at `TASK_API_URL`. This avoids CORS entirely (`task-api` needs no
CORS change and no second public origin to trust) and keeps `task-api`'s address an in-cluster detail.

- AC-002.7, AC-002.8, AC-002.14 (load): `/` is a dynamic server component (`export const dynamic = "force-dynamic"`,
  `fetch` with `cache: "no-store"`) that calls `GET {TASK_API_URL}/tasks` on every request and renders the list in
  the order received, title and `due_date` as given (`YYYY-MM-DD`, no date parsing or reformatting, so no time-zone
  shift) or "No due date"; "No tasks yet." for `[]`; "Tasks could not be loaded. Try again later." for any failure
  (see "Errors"). The add form is shown in all three states.
- AC-002.9 to AC-002.12, AC-002.15 (add): a client component holds the list and the form. It validates before
  sending: title empty or whitespace-only gives "Enter a title."; more than 200 characters (counted in Unicode code
  points, `[...title].length`, the way Python's `len` counts) gives "The title can have at most 200 characters.".
  The message is shown at the title field and nothing is sent. Otherwise it posts to `/api/tasks`; on `201` it appends
  the returned task at the end of the list and empties both fields; the title is sent exactly as typed (no trim). The
  due date comes from `<input type="date">`; empty means `"due_date": null`.
- AC-002.14 (save): any non-`201`/non-`422` answer, a network error or a timeout shows "The task could not be saved.
  Try again." and keeps both fields.
- AC-002.13: titles are rendered as React text nodes only; no `dangerouslySetInnerHTML`, no markdown. Test with the
  exact title of the criterion.
- `/api/ready` must not depend on `task-api`: otherwise the page would be unreachable exactly when AC-002.14 must
  show its message.

## Contract

Two contracts: `task-api` for any HTTP caller (the web app's server among them), and the web app's own
`/api/tasks` route between its browser code and its server. The browser never calls `task-api`.

### task-api (unchanged from spec 001 except the `503`)

Base URL: in the namespace `http://task-api` (Service port 80, container 8080); in dev also through the Gateway at
`http://task-api.taskboard-dev.localhost`. All bodies are JSON (`Content-Type: application/json`, UTF-8).

Task object:

| Field | Type | Notes |
|---|---|---|
| `id` | string | non-empty, assigned by the server, unique among all tasks in the environment's database; opaque (UUID4 string) |
| `title` | string | exactly as sent, untrimmed, 1 to 200 characters (Unicode code points), not only whitespace |
| `due_date` | string or `null` | `YYYY-MM-DD` calendar date, or `null` when none was given |

No other fields are returned.

`POST /tasks`: request object with `title` (required, JSON string, not empty or only whitespace, at most 200
characters as sent) and `due_date` (optional; absent or `null` = none; else exactly `YYYY-MM-DD` and a real
calendar date, past dates accepted). Unknown fields are ignored. Responses:

- `201` with the created task object; it is durably stored when the response is sent.
- `422` for invalid input (format in "Errors"), checked before the database is touched; nothing is stored.
- `503` when the task store is unavailable; nothing is stored.

`GET /tasks`: no parameters. `200` with a bare JSON array of task objects in creation order, oldest first; `[]`
only when the database holds no task. `503` when the task store is unavailable (never `[]` in that case).

`GET /ready`: `200` `{"status": "ready"}` when a `SELECT 1` on the database succeeds; `503`
`{"status": "not ready"}` otherwise. `GET /health`: always `200` while the process runs (does not check the
database). `/metrics`, `/ping`, `/docs`, `/openapi.json` unchanged; `/openapi.json` lists the `503` on both
task operations.

### taskboard-web internal route (same origin, browser to web server)

`POST /api/tasks` on the web app, body `{"title": <string>, "due_date": <"YYYY-MM-DD" or null>}`, forwarded
unchanged to `POST {TASK_API_URL}/tasks`:

- task-api `201`: answer `201` with the task object as received.
- task-api `422`: answer `422` with task-api's body as received (the client shows the title message when an entry's
  `loc` ends in `"title"`, choosing the text by its own length/whitespace rule, and "The task could not be saved.
  Try again." for any other `422`).
- anything else, a connection error or a timeout: answer `502` `{"detail": "The task could not be saved."}`.

The page `/` itself reads `GET {TASK_API_URL}/tasks` on the server; there is no `GET /api/tasks`.

### Errors

`task-api` error formats:

- `422`: FastAPI's validation body `{"detail": [{"loc": ["body", "title"], "msg": "...", "type": "..."}]}`; the last
  `loc` element names the field (`"title"` or `"due_date"`); `["body"]` for a body that is not a JSON object. `msg`
  and `type` are informative only.
- `503`: `{"detail": "The task store is unavailable."}` (exactly this string) for `POST /tasks` and `GET /tasks`.
- `404`/`405`: FastAPI defaults `{"detail": "..."}`. `500` only for an unexpected fault unrelated to the database.

How a caller treats a response the contract does not list (any other `4xx`/`5xx`, a body that is not the expected
JSON shape, a `200` from `GET /tasks` that is not an array of task objects): the call failed. It does not assume a
task was created and does not show a partial list. For `taskboard-web`: a failed `GET /tasks` (including `503`)
renders "Tasks could not be loaded. Try again later."; a failed `POST` renders "The task could not be saved. Try
again." with the inputs kept. `POST /tasks` is not idempotent: neither the web server nor the browser retries it
automatically; after a timeout the task may or may not exist, and the user's next page load shows the truth.

Gateway or Service failures (`502`/`503`/`504` without the JSON body, connection refused) count as failures in the
same way.

### Timeouts

- `task-api` to Postgres: connect timeout 2 s, pool checkout timeout 2 s, `statement_timeout` 2 s. When the
  database is down or slow, `POST /tasks` and `GET /tasks` answer `503` within 5 s; `/ready` answers `503` within
  3 s. The pool keeps retrying connections in the background; no request blocks longer than these limits, and the
  process does not exit.
- Readiness recovery: `/ready` reflects the database on every call, so it returns to `200` at the first probe after
  the database is back (kubelet period 10 s, under the 30 s of AC-002.4).
- `taskboard-web` server to `task-api`: 6 s total per request (`AbortSignal.timeout(6000)`), above `task-api`'s own
  5 s worst case so a `503` arrives as a `503` and not as a timeout. No retry for `POST`; no retry for the page's
  `GET` either (the page renders the load-error message; the user reloads).
- Browser to `taskboard-web` `/api/tasks`: 10 s (`AbortSignal.timeout(10000)`); on timeout the save-error message
  is shown and the inputs are kept.
- When `task-api` is down, the web app keeps serving `/` (with the load-error message) and stays ready; its probes
  never call `task-api`.

## gitops-app PR

The gitops-app entry is one PR here with the four operations above (`/system:ship` runs them in order with
`cplat resource add` and `cplat component set`, then `devbox run render`). It merges before the `task-api` PR (level 0).

Pins: `gitops_pin` is empty. Both components run only in `dev`, and dev follows `main`: each component's merge opens
the self-merging `pin/dev-<component>-<sha7>` PR here. Staging and prod are out of scope (non-goal); they follow later
with `/instance:promote`.

After Argo reconciles the gitops PR, verify:

1. The CloudNativePG cluster for `taskboard-postgres` in `taskboard-dev` is healthy and the Secret
   `taskboard-postgres-app` exists.
2. The `task-api` Deployment has `DATABASE_URL` and `PG*` from that Secret (the old in-memory image still runs and
   ignores them).
3. `http://taskboard-web.taskboard-dev.localhost:8088/` answers `200`; the `taskboard-web` Deployment has
   `TASK_API_URL=http://task-api`.

After the `task-api` and `taskboard-web` dev pins reconcile, verify:

4. `POST /tasks` `{"title": "Buy milk"}` on `http://task-api.taskboard-dev.localhost:8088` returns `201`; delete the
   `task-api` pod; after it is ready again `GET /tasks` still returns the task (AC-002.1).
5. `http://taskboard-web.taskboard-dev.localhost:8088/` lists that task; adding "File taxes" with 2026-12-31 from the
   page appends it without a reload, and `GET /tasks` on the API includes it.
6. Scale the postgres cluster's instances to 0 or block it briefly: `GET /tasks` answers `503`, `/ready` is not `200`,
   the web page shows "Tasks could not be loaded. Try again later."; restore it and `/ready` is `200` within 30 s
   without a pod restart.
