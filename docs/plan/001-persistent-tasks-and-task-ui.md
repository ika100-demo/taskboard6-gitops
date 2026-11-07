---
plan_id: 001-persistent-tasks-and-task-ui
spec: 001-persistent-tasks-and-task-ui
feature: Tasks survive restarts (in-cluster Postgres) and can be listed and created in the web app
gitops_app: ika100-demo/taskboard6-gitops
status: draft
repos:
  - id: taskboard6-gitops
    shape: gitops-app
    summary: Declare the system's Postgres, give it to the API and point the web app at the API
    acs: [AC-001.1, AC-001.3]
    gitops:
      - {resource: postgres}
      - {uses: postgres, component: taskboard6-api}
      - {env: {TASKBOARD6_API_URL: 'http://taskboard6-api'}, component: taskboard6-web}
    depends_on: []
    done: false
  - id: taskboard6-api
    shape: service-python
    summary: Store tasks in Postgres and answer 503 and not-ready while the database is unreachable
    acs: [AC-001.1, AC-001.2]
    criteria:
      - "Given tasks A, B and C were created through `POST /tasks` in that order, when the service is stopped and started again with the same `DATABASE_URL` and a client sends `GET /tasks`, then the response is `200` with exactly A, B, C in that order, each with the same `id`, `title`, `due_date` and `created_at` (to the microsecond) returned by its `POST /tasks` (system AC-001.1)"
      - "Given tasks were created in one service process, when a second service process using the same `DATABASE_URL` answers `GET /tasks`, then it returns the same tasks in the same creation order (system AC-001.1)"
      - "Given a running service with a reachable database, when a client uses `POST /tasks` and `GET /tasks`, then paths, fields, trimming, the 1 to 200 character title rule, the `YYYY-MM-DD` due date rule and the `422` error shape behave exactly as in taskboard6-api spec 001 (system AC-001.1)"
      - "Given the database is unreachable, when a client sends `POST /tasks` with a valid body, then the response is `503` with JSON body `{\"detail\": \"Task storage is unavailable\"}` within 5 seconds, and once the database is reachable again `GET /tasks` does not contain that task (system AC-001.2)"
      - "Given the database is unreachable, when a client sends `GET /tasks`, then the response is `503` with JSON body `{\"detail\": \"Task storage is unavailable\"}` within 5 seconds (system AC-001.2)"
      - "Given the database is unreachable, when the readiness endpoint `GET /ready` is called, then it answers `503` with JSON `{\"status\": \"not ready\"}`, while `GET /health` still answers `200` (system AC-001.2)"
      - "Given the service started while the database was unreachable, when the database becomes reachable, then without a restart `GET /ready` answers `200` and `POST /tasks` and `GET /tasks` work again, the schema being created on first use (system AC-001.2)"
    arguments: |
      Config: `DATABASE_URL` (injected by the gitops-app's `uses: postgres`, also `PG*`); the service must start and
      keep `/health` green when it is unset or the database is down (then `/ready` and the task routes answer 503).
      Suggested: psycopg 3 with psycopg_pool, an idempotent `CREATE TABLE IF NOT EXISTS` on the first successful
      connection (no migration tool needed for one table). Table `tasks`: `seq bigserial primary key`, `id text unique`
      (UUID string), `title text not null`, `due_date date null`, `created_at timestamptz not null`; list
      `ORDER BY seq` so ties in `created_at` keep creation order. Keep the TaskStore seam so the route code stays.
      Tests need a real Postgres (devbox service or testcontainers); "restart" = a new app instance on the same DB;
      "unreachable" = a DATABASE_URL pointing at a closed port. In-memory tasks are not migrated (spec A2).
    depends_on: [taskboard6-gitops]
    done: false
  - id: taskboard6-web
    shape: web-nextjs
    summary: Task page at the root URL - list tasks with due dates and create a task with a form
    acs: [AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.7, AC-001.8]
    criteria:
      - "Given the API's `GET /tasks` returns tasks, when a user opens `/`, then the page shows one list item per task in the returned order, each with its title and its `due_date` as `YYYY-MM-DD`, or the text \"No due date\" when `due_date` is `null` (system AC-001.3, AC-001.6)"
      - "Given the API's `GET /tasks` returns `[]`, when a user opens `/`, then the page shows \"No tasks yet\" and the create form with the fields labelled \"Title\" and \"Due date\" and the button \"Add task\" (system AC-001.4)"
      - "Given `/` is open, when the user enters the title \"Buy milk\" and the due date 2026-12-31 and presses \"Add task\", then within 5 seconds and without a page reload the task \"Buy milk\" with \"2026-12-31\" appears as the last list item, both fields are empty, and the API's `GET /tasks` contains it (system AC-001.5)"
      - "Given `/` is open, when the user submits a title with the due date left empty, then the request to the API carries `due_date: null` and the new list item shows \"No due date\" (system AC-001.6)"
      - "Given `/` is open, when the user submits a title that is empty or only whitespace, then \"Please enter a title.\" is shown next to the title field, the entered values stay, and no request reaches the API; for a title longer than 200 characters after trimming the message is \"The title must be at most 200 characters.\" (system AC-001.7)"
      - "Given the API answers `POST /tasks` with `422` naming `title`, when the user submits, then the API's message is shown next to the title field and the entered values stay (system AC-001.7)"
      - "Given the API is unreachable, slower than 3 seconds or answers `GET /tasks` with a `5xx`, when a user opens `/`, then the page loads with status `200` and shows \"Tasks could not be loaded. Please try again later.\" instead of the list (system AC-001.8)"
      - "Given the API is unreachable, slower than 3 seconds or answers `POST /tasks` with a `5xx` or any status the contract does not list, when the user submits the form, then the page shows \"The task could not be saved. Please try again.\" and keeps the entered values (system AC-001.8)"
    arguments: |
      Config: `TASKBOARD6_API_URL` (server-side only, never `NEXT_PUBLIC_*`; set by the gitops-app to
      `http://taskboard6-api`); document it in docs/env-vars.md. Replace the starter page. The browser never calls the
      API directly: `/` is a dynamic Server Component that fetches `GET /tasks`; the form (client component) posts to
      the web app's own `POST /api/tasks` route handler, which forwards to the API (see Contract). `/api/ready` must not
      depend on the API (the page must load while the API is down). Tests: mock the API in Vitest; e2e against a stub
      API server; smoke: page loads (`@smoke`), create a `smoke-<uuid>` task (`@smoke @writes`).
    depends_on: [taskboard6-gitops]
    done: false
gitops_pin: []
---

## taskboard6-gitops — Declare the system's Postgres, give it to the API and point the web app at the API

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.3 · **Depends on:** []

Persistence needs a database the API can reach in dev (spec A9: the system's own in-cluster CloudNativePG Postgres, no
managed service). `resource postgres` declares it, `uses postgres` on `taskboard6-api` injects `DATABASE_URL` and `PG*`
from the generated Secret, and `env TASKBOARD6_API_URL` tells the web app's server where the API lives (the in-namespace
Service `taskboard6-api`, port 80). Both components are already exposed (`taskboard6-api`, `taskboard6-web` hosts), so no
`expose` operation is needed. It goes first: the current in-memory API ignores `DATABASE_URL` and the starter web app
ignores `TASKBOARD6_API_URL`, so the change is harmless on its own, while a Postgres-backed API merged without it would
answer `503` in dev.

## taskboard6-api — Store tasks in Postgres and answer 503 and not-ready while the database is unreachable

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2 · **Depends on:** [taskboard6-gitops]

The API owns durability (AC-001.1) and the database-down behaviour (AC-001.2). Only the storage behind `TaskStore`
changes; the HTTP contract of taskboard6-api spec 001 stays as is (spec A1), and its existing acceptance tests must keep
passing against Postgres. The readiness probe in `system.yaml` is already `/ready`, so Kubernetes takes the pod out of
the Service while the database is down; `/health` stays green so the pod is not restarted in a loop. Test concerns:
`created_at` must round-trip at microsecond precision (timestamptz does); order by an insertion sequence, not by
`created_at` alone.

## taskboard6-web — Task page at the root URL - list tasks with due dates and create a task with a form

**Shape:** `web-nextjs` · **Criteria:** AC-001.3 to AC-001.8 · **Depends on:** [taskboard6-gitops]

The web app owns everything the user sees: the list, the empty state, the form, client-side title validation (spec A7)
and the two error messages. It builds in parallel with the API, against the contract below (the API contract is
unchanged from taskboard6-api spec 001, which is already live in dev). AC-001.5's "`GET /tasks` returns the task" is
the API's existing behaviour; the web app's test checks it through the API stub or, in the smoke test, the real API.

## Contract

The API contract is taskboard6-api spec 001, unchanged, plus the `503` answers of this spec. The web app reads
`TASKBOARD6_API_URL` (no trailing slash; dev: `http://taskboard6-api`) and calls the API only from its server.

**Task (JSON):** `{"id": string (UUID), "title": string (trimmed, 1 to 200 chars), "due_date": "YYYY-MM-DD" | null,
"created_at": "YYYY-MM-DDTHH:MM:SS.ffffffZ" (UTC)}`.

**API `GET /tasks`** (no auth, no query parameters)
- `200`: JSON array of Task, oldest first (creation order), no limit; `[]` when none.
- `503`: `{"detail": "Task storage is unavailable"}` when the database is unreachable.

**API `POST /tasks`** (no auth) body `{"title": string, "due_date": "YYYY-MM-DD" | null}`; `due_date` may be omitted
(= `null`); unknown fields, `id` and `created_at` are ignored.
- `201`: the created Task.
- `422`: FastAPI standard shape `{"detail": [{"loc": ["body", "<field>"], "msg": string, "type": string}, ...]}`; the
  field is `title` or `due_date`. No task is created.
- `503`: `{"detail": "Task storage is unavailable"}`; no task is created.

**API `GET /ready`**: `200` `{"status": "ready"}` when the database answers; `503` `{"status": "not ready"}` otherwise.
**API `GET /health`**: `200` always while the process runs (does not check the database).

**Web `GET /`**: server-rendered on every request (no static caching). Always answers `200`. Shows the create form
(fields labelled "Title" and "Due date" (`<input type="date">`), button "Add task") above either the task list (one
item per task, title and `due_date` or "No due date"), "No tasks yet", or "Tasks could not be loaded. Please try again
later.". The form is shown in all three states.

**Web `POST /api/tasks`** (the web app's own route handler, called by the form): body `{"title": string, "due_date":
"YYYY-MM-DD" | null}`, forwarded unchanged to the API's `POST /tasks`.
- API `201` -> `201` with the Task; the page appends it to the end of the list (removing "No tasks yet") and clears
  both fields.
- API `422` -> `422` with the API's body unchanged; the page shows the first `msg` whose `loc` ends in `title` next to
  the title field (and one whose `loc` ends in `due_date` next to the due date field); values stay.
- anything else (API `5xx`, unlisted status, unreachable, timeout) -> `502` `{"detail": "upstream unavailable"}`; the
  page shows "The task could not be saved. Please try again." near the button; values stay.

**Client-side title check** (before any request): trim; empty -> "Please enter a title."; more than 200 characters ->
"The title must be at most 200 characters."; shown next to the title field, nothing is sent. A failed or successful
new submit replaces the previous message.

### Errors

- API errors are JSON with a top-level `detail`: a string for `503`, the FastAPI list for `422`. The API never answers
  an HTML error page; an unexpected failure is `500` `{"detail": "Internal Server Error"}`.
- Web route handler errors are JSON `{"detail": ...}` as above.
- A caller treats any status the contract does not list for that call, a body that is not the documented JSON, a
  connection error or a timeout exactly like a `5xx`: the web page shows "Tasks could not be loaded. Please try again
  later." (list) or "The task could not be saved. Please try again." (create). The web app never retries a `POST`
  (to avoid duplicates); the user retries.
- The API logs every `503` with the database error (structlog, no credentials in the log).

### Timeouts

- Web server -> API `GET /tasks`: 3 s total; on timeout, the list error message (page still `200`).
- Web server -> API `POST /tasks`: 3 s total; on timeout, `502` to the browser and the save error message.
- Browser -> web `POST /api/tasks`: 4 s client-side abort; on abort, the save error message. Together this keeps
  AC-001.5's 5-second budget.
- API -> Postgres: connect timeout 2 s, statement timeout 2 s; on either, the task routes answer `503` (well inside 5 s)
  and `/ready` answers `503`. A pooled connection that has gone bad is discarded and the next request reconnects; no
  restart is needed once the database is back.
- API `/ready` -> Postgres: `SELECT 1` with a 1 s timeout (the probe's own timeout is the platform default).

## gitops-app PR

One PR in this repo runs the three operations above (`cplat resource add postgres`, `cplat component set taskboard6-api
--uses postgres`, `cplat component set taskboard6-web --env TASKBOARD6_API_URL=http://taskboard6-api`) and re-renders.
No pins: both components run only in dev (spec non-goal: staging and prod are out of scope), and dev tracks `main`, so
each component's merge reaches dev by itself; `gitops_pin` is empty.

After Argo reconciles: the CloudNativePG cluster for taskboard6 is healthy; the `taskboard6-api` pod has `DATABASE_URL`
set and `/ready` answers `200` once the API PR is deployed; `taskboard6-web` has `TASKBOARD6_API_URL`. Then verify the
spec end to end in dev: create a task at `http://taskboard6-web.taskboard6-dev.localhost/`, restart the API deployment,
reload the page and see the task still listed (AC-001.1, AC-001.3).
