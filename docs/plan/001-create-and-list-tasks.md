---
plan_id: 001-create-and-list-tasks
spec: 001-create-and-list-tasks
feature: Create a task (title, optional due date) and list all tasks, held in memory in task-api
gitops_app: ika100-demo/taskboard-gitops
status: completed
repos:
- id: task-api
  shape: service-python
  summary: Create and list tasks in memory (POST /tasks, GET /tasks)
  acs:
  - AC-001.1
  - AC-001.2
  - AC-001.3
  - AC-001.4
  - AC-001.5
  - AC-001.6
  - AC-001.7
  - AC-001.8
  - AC-001.9
  - AC-001.10
  - AC-001.11
  - AC-001.12
  arguments: |
    FastAPI app in src/task_api/main.py; add the routes there or in a new router module next to it.
    The store is one process-wide in-memory list guarded by a lock; no database, no new dependency.
    Keep the Dockerfile CMD as a single uvicorn process (no --workers): a second worker would hold a
    second list and break the shared list just as a second replica would.
  depends_on: []
  done: true
gitops_pin: []
---

This feature fits a single repo: every criterion is behaviour of `task-api`. Nothing in this gitops-app repo has
to change, so the plan has no `gitops-app` entry (details under "gitops-app PR").

## task-api — Create and list tasks in memory (POST /tasks, GET /tasks)

**Shape:** `service-python` · **Criteria:** AC-001.1 to AC-001.12 · **Depends on:** []

`task-api` is the only component of the system and the only place tasks exist. It owns all of it:

- `POST /tasks`: validation of `title` and `due_date` (AC-001.4, AC-001.5, AC-001.6), storing the title exactly as
  sent (AC-001.11), accepting any valid calendar date including past ones (AC-001.2, AC-001.10), a new unique `id`
  per task (AC-001.1, AC-001.3).
- `GET /tasks`: the bare array in creation order (AC-001.7, AC-001.8).
- The store: one list in process memory, created empty at startup (AC-001.7, AC-001.9), shared by every request
  (AC-001.12). Inside the repo AC-001.12 means: one module-level store per process, no per-request or per-worker
  state, and a single uvicorn process (the current Dockerfile CMD). The "one instance in every environment" half
  is deployment and is already true (see "gitops-app PR").

Test concerns:

- A test fixture must reset the store between tests (or build a fresh app), otherwise AC-001.7 depends on test order.
- AC-001.9 (restart): test it by constructing a fresh app/store and asserting it is empty, not by restarting a container.
- AC-001.4 "no task is created": assert `GET /tasks` is unchanged after each rejected request, for every invalid case.
- AC-001.5: `"2026-02-30"` must be rejected; a plain regex on `YYYY-MM-DD` is not enough, parse the date. Also reject
  `"2026-1-5"`, a date with a time (`"2026-12-31T10:00:00"`) and non-strings (`42`); Pydantic's lax date parsing
  accepts some of these, so use a strict string-then-`date.fromisoformat` check with an exact `YYYY-MM-DD` pattern.
- AC-001.4: a non-string title (`42`, `null`, `["x"]`) must be `422`, not coerced; use strict typing.
- AC-001.6: the limit counts characters (Python `len` of the string), boundary 200 accepted, 201 rejected; whitespace
  counts toward the length.
- AC-001.12: create one task, then 10 consecutive `GET /tasks`, each must include it.

## Contract

The contract is between `task-api` and any HTTP caller (there is no other repo in this system yet). Base path: the
service root (in the cluster `http://task-api`, port 80 on the Service, 8080 in the container; in dev through the
Gateway at `http://task-api.taskboard-dev.localhost`). All bodies are JSON (`Content-Type: application/json`, UTF-8).

### Task object

| Field | Type | Notes |
|---|---|---|
| `id` | string | non-empty, assigned by the server, unique among the tasks of the running instance; opaque to callers (implementation: UUID4 as string) |
| `title` | string | exactly as sent, untrimmed, 1 to 200 characters, not only whitespace |
| `due_date` | string or `null` | calendar date `YYYY-MM-DD` (ISO 8601, no time, no zone), or `null` when none was given |

No other fields are returned.

### `POST /tasks`

Request body: a JSON object.

| Field | Required | Rule |
|---|---|---|
| `title` | yes | JSON string; after stripping whitespace (for the check only) at least 1 character; at most 200 characters as sent |
| `due_date` | no | absent or `null` means no due date; otherwise a JSON string matching exactly `YYYY-MM-DD` that is a real calendar date (past, today and future all accepted) |

Unknown extra fields are ignored. A body that is not a JSON object (or not valid JSON) is `422`.

Responses:

- `201 Created` with the created task object. No `Location` header is required.
- `422 Unprocessable Entity` for any invalid input (see "Errors"); nothing is stored.

### `GET /tasks`

No parameters (query parameters are ignored). Response `200 OK` with a bare JSON array of task objects in creation
order, oldest first; `[]` when there are none. Each element equals the create response of that task.

### Existing endpoints (unchanged)

`GET /health`, `GET /ready`, `GET /metrics`, `GET /ping`, and the FastAPI docs (`/docs`, `/openapi.json`) keep
their current behaviour. `/openapi.json` must describe the two new operations and the task object.

### Errors

Error format: FastAPI's standard validation error body, status `422`:

```json
{"detail": [{"loc": ["body", "title"], "msg": "<human-readable message>", "type": "<error type>"}]}
```

- `detail` is a non-empty array. Each entry's `loc` is the path to the offending field; for a field error the last
  element is the field name (`"title"` or `"due_date"`), which is how an error "names the field" (AC-001.4 to AC-001.6).
- A missing `title` yields an entry with `loc` `["body", "title"]`. When both fields are invalid, both are listed.
- A body that is not a JSON object yields an entry whose `loc` is `["body"]`.
- `msg` and `type` are informative; callers and tests must not depend on their exact text.

Other statuses: `405` for another method on `/tasks`, `404` for unknown paths (FastAPI defaults,
`{"detail": "..."}`). `500` only for an unexpected server fault.

A caller treats any status the contract does not list for an operation (any `5xx`, an unexpected `4xx`) as a failure
of that call: it does not assume a task was created and does not parse the body as a task. A `POST` that failed that
way or timed out may or may not have stored the task; the caller may check with `GET /tasks` before retrying, since
`POST /tasks` is not idempotent (a retry creates a second task).

### Timeouts

- Server side: both operations are in-memory and answer within 100 ms under normal load; no call made by
  `task-api` to anything else (it has no dependencies), so there is no downstream timeout or fallback.
- Callers: use a 5 s total timeout per `POST /tasks` and `GET /tasks` request (2 s to connect). On a timeout or a
  connection error the call has failed (see "Errors" for the non-idempotent `POST`); retry at most once for `GET`.
- When `task-api` is down or restarting, the Gateway or Service answers with a connection error or `502`/`503`;
  callers treat that as failure. After a restart the list is empty (AC-001.9): that is expected, not an error.
- Probes (`/health`, `/ready`) stay as they are; `/ready` needs no change because the store is ready at startup.

## gitops-app PR

No gitops-app change is needed for this feature, so there is no `gitops-app` entry and no gitops operation:

- No resource: tasks are in memory (non-goal: persistence).
- Exposure already exists: `components.task-api.expose.host: task-api` in `applications/taskboard/system.yaml`
  publishes it in dev at `http://task-api.taskboard-dev.localhost`.
- Single instance (AC-001.12): `task-api` sets neither `replicas` nor `scale` in `system.yaml`, and the renderer's
  default is one replica (`replicas_for` in `scripts/render.py` returns `svc.get("replicas", 1)`), so it already runs
  as one instance. No `replicas: 1` edit is required. Whoever ships or later promotes `task-api` must keep it that
  way: do not add `replicas` greater than 1 or a `scale:` block, on the component or under any `instances.<env>`
  (staging, prod), while tasks are in memory. If the team wants the rule explicit in the file, adding
  `replicas: 1` under `components.task-api` in `system.yaml` (then `devbox run render`) is the only change, and it
  renders the same Deployment.

Pins: `gitops_pin` is empty. `task-api` runs only in `dev` (`instances: {dev: {}}`), and dev follows `main`: the
merge of the task-api PR opens the self-merging `pin/dev-task-api-<sha7>` PR here (pin-dev.yml). Staging and prod
are out of scope until someone runs `/instance:promote task-api --to staging`.

After Argo reconciles the dev pin, verify:

1. The `task-api` Deployment in `taskboard-dev` has 1 ready replica and the image `sha-<7>` of the merge.
2. `GET http://task-api.taskboard-dev.localhost:8088/tasks` returns `200` and `[]` (fresh pod).
3. `POST /tasks` with `{"title": "Buy milk"}` returns `201` with an `id`; a following `GET /tasks` (repeat 10 times)
   returns it each time.
4. `POST /tasks` with `{"title": ""}` returns `422` naming `title`.
