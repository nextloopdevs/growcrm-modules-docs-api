# Grow CRM — API Development Guide

How to build API endpoints in the API module consistently. The **Projects** endpoint
(`Modules/Api/Http/Controllers/Projects/Projects.php`) is the canonical reference — copy its
patterns. This guide captures every architectural decision so a new controller is mechanical.

---

## Next Steps

When the developer provides a **web controller path** (and this guide), do this **before
writing any code**:

1. **Read this guide in full**, then read the provided web controller (and the routes that
   point to it) to understand its full surface.
2. **Identify every operation**: the core CRUD (index/store/update/destroy) and all ancillary /
   dropdown actions (status changes, assignments, toggles, clone, etc.).
3. **Propose an endpoint list** as a table — for each: HTTP verb + path, the action pattern it
   maps to (§2), the application repository/logic it will reuse, and the payload. Mark anything
   recommended **out of scope** (end-user-only actions, file uploads, automation, per-user UI
   state, bulk) with a short reason.
4. **Surface open decisions** (e.g. replace-vs-append semantics, allowed enum values) concisely.
5. **Stop and await explicit confirmation.** Do not create files, routes, or docs until the
   developer approves (and amends) the proposed endpoint list.

Once confirmed, implement following §2–§3, then sign off with a live test (§4).

---

## 1. Architecture

The API is split into a shared **platform** (written once) and per-resource code (scaffolded).

### Platform (shared by every resource — never duplicated)

| Component | File | Responsibility |
|---|---|---|
| Auth | `Http/Middleware/Authenticate.php` | Validates the Bearer API key against `mod_api_settings`; authenticates the request **as the main admin (user id 1)**; boots system settings. |
| Force JSON | `Http/Middleware/ForceJsonResponse.php` | Sets `Accept: application/json` so framework errors (throttle, 405, …) return JSON. |
| Base controller | `Http/Controllers/ApiController.php` | Response envelope + error helpers. |
| Base request | `Http/Requests/ApiFormRequest.php` | `authorize()` + JSON `422` validation failures. |
| Base transformer | `Transformers/Transformer.php` | `transform()` contract + `collection()` helper. |
| Routing / tenancy | `Providers/RouteServiceProvider.php` | Mounts `/api`, forces JSON, and (SaaS) resolves the tenant by domain. |

### Per-resource (scaffolded — see `Scaffolding/ApiResource/`)

Controller, Store/Update + action requests, Transformer, read Repository, route lines, docs,
lang keys, live-test script.

### Directory & namespace
The module directory is `Modules/Api` and the namespace is `Modules\Api\…` (they must match
case — required for PSR-4 on case-sensitive Linux). All sub-paths mirror the namespace
(`Http/Controllers`, `Transformers`, `Repositories`, …).

---

## 2. Standards (the contract every endpoint conforms to)

### Authentication
- Header: `Authorization: Bearer <key>`. Keys live in `mod_api_settings`.
- **All keys are administrator-level** and run as the CRM **main administrator (user id 1)** —
  there are no user-scoped keys, so there is no `403` path.
- Per-record attribution can be set explicitly in the payload where supported
  (e.g. `project_creatorid`; defaults to user id 1 via the reused repository).
- Multitenant: clients call their own domain; the domain selects the tenant (see §5).

### HTTP verbs
| Verb | Use |
|---|---|
| `GET` | list / read |
| `POST` | create, and create-from actions (e.g. clone) |
| `PATCH` | partial update of a resource's own fields |
| `PUT` | set/replace an ancillary value or collection (idempotent), and command actions |
| `DELETE` | delete |

### Routes
- Registered in `Modules/Api/Routes/api.php` inside the `Authenticate` middleware group.
- Older Laravel string-action format: `Route::get('/foos', 'Foos\Foos@index');`.
- Resolve records by their **auto-increment id** (`{id}` = `foo_id`). Use the model's actual
  primary key — some tables have no unique-id column at all (e.g. `clients` → `client_id`).
- No version segment in the URL.
- CRUD: `GET /foos` (list), `GET /foos/{id}` (read one), `POST /foos`, `PATCH /foos/{id}`,
  `DELETE /foos/{id}`. The single-record read returns the same transformed object as the writes.

### Sub-resource actions (ancillary actions on a resource)
Relational / ancillary actions are **not** folded into create/update — they are their own
endpoints under the resource id. **Id always in the path; the body carries only the action's
data, never the id.** Bulk variants (id in the body) are a separate, deliberate design.

Patterns we use (all demonstrated on Projects):

| Pattern | Verb / shape | Body | Implementation |
|---|---|---|---|
| **State toggle** | `PUT /{id}/archive`, `/restore` | none | set the state column inline, save |
| **Single-value setter** | `PUT /{id}/status`, `/progress`, `/manager` | one value | set inline (or reuse repo `delete`+`add` for a relation) |
| **Collection replace** | `PUT /{id}/assign`, `/tags` | array (full set; empty clears) | reuse repo `delete()` then `add()` |
| **Create-from** | `POST /{id}/clone` | create-like fields | reuse the clone repository; returns the new resource (`201`) |
| **Command** | `PUT /{id}/stop-timers` | none | reuse the domain repository |

### Payload key naming
- **Create / update / clone** use the underlying **column names** (`project_title`,
  `project_clientid`, …) — they mirror the web create form.
- **Action endpoints** use **friendly keys** that match the transformer's output fields
  (`status`, `progress`, `tags`, `assigned`, `manager`). What you `PUT` reads back the same.

### Success envelope
```json
{ "data": <object|array>, "meta": { ... }, "message": "..." }
```
`meta` is present on list endpoints (pagination) only.

| Action | Helper | Status |
|---|---|---|
| List | `respondPaginated($paginator, $transformer, $message)` | 200 |
| Read (single record) | `respondData($data, $message)` | 200 |
| Create / clone | `respondCreated($data, $message)` | 201 |
| Update / action returning the resource | `respondData($data, $message)` | 200 |
| Delete / command returning an id | `respondDeleted($id, $message)` | 200 |

Actions that change the resource return the **re-fetched, transformed resource** so the caller
sees the new state.

### Error envelope
```json
{ "message": "...", "errors": { "field": ["..."] } }
```
`errors` is present only for validation (`422`).

| Status | When |
|---|---|
| `401` | Missing/invalid API key (middleware) |
| `404` | Record not found — `$this->notFound()` |
| `409` | Business/operation failure — `$this->respondError($msg, 409)` |
| `422` | Validation — handled by `ApiFormRequest` |

> **Never call `abort()` in an API controller.** The application's exception handler
> (`app/Exceptions/Handler.php`) is built for the web/ajax UI and returns HTML or the ajax
> notification shape. Use the `ApiController` error helpers, which throw `HttpResponseException`
> carrying the JSON envelope.

### Pagination & query params (list endpoints)
- `page`, `limit` (cap at 100; default `config('system.settings_system_pagination_limits')`).
- `sort` (whitelisted columns only) + `order` (`asc|desc`).
- Resource filters as needed (e.g. `status`, `client_id`, `search`).

### Validation
- One request class per write endpoint (`Store<Resource>`, `Update<Resource>`, and one per
  action, e.g. `AssignProject`, `ChangeStatus`, `CloneProject`), all extending `ApiFormRequest`.
- Port rules from the resource's **web** request; drop UI-only fields.
- Collection-set actions use `['present','array']` so an empty array is valid (clears the set).

### Reuse, don't reimplement
- Writes reuse the existing application repository: `FooRepository::create()/update()`.
- Deletes reuse `DestroyRepository::destroy<Resource>()` for cascade parity.
- Actions reuse the relation's repository (e.g. `ProjectAssignedRepository`, `TagRepository`,
  `CloneProjectRepository`, `TimerRepository`) — `delete()` + `add()` to replace, or the
  repo's own method.
- Replicate only the **essential** structural setup from the web controller's `store()`
  (e.g. default folders / milestones). 
- **Never modify shared application files** — reuse only.

### Out of API scope (intentionally excluded)
Unless a task says otherwise, API write paths **omit**: activity events, notification emails,
attachment/cover-image handling, per-user UI state (e.g. pinning), automation triggers, and
UI redirects. Bulk actions are a separate track. Flag these in the implementation report.

### Serialization
- Each resource has a `Transformer` (extends `Transformer`). No serialization logic in
  controllers or responses. Keep field naming consistent across resources; friendly action
  keys should match the field names the transformer emits.

### Language
- Developer-facing messages live in the module lang file (`api::lang.*`).
- Append new keys to the **end** of `Modules/Api/Resources/lang/english/lang.php`; reuse core
  `lang.*` keys (e.g. `item_not_found`, `cloning_failed`) where suitable.

### Documentation
- Shared, written once (never repeated per resource):
  - `Docs/getting-started.md` — base URL, envelope, errors, identifiers, pagination, the
    sub-resource convention.
  - `Docs/authentication.md` — API keys, privileges, multitenant, `401`.
- One Markdown file per resource, modelled on `Docs/projects.md`: a link to
  getting-started/authentication (**no auth section**), the object schema, every endpoint, and
  a resource-specific error note.
- Each **Example request** shows **both** a `curl` and a plain-PHP (`curl_*`) example.

---

## 3. Add a resource — process

1. Copy the stubs from `Modules/Api/Scaffolding/ApiResource/` and rename placeholders.
2. Fill in `rules()`, the read repository `search()`, and the transformer.
3. Wire create/update/delete to the existing application repositories; replicate only essential
   structural setup.
4. Add ancillary actions as sub-resource endpoints using the patterns in §2.
5. Add routes to the `Authenticate` group; append lang keys.
6. Write `Docs/<resource>.md` (curl + PHP per example).
7. Add a live-test script (see §4) and sign off.
8. Follow `Scaffolding/ApiResource/_CHECKLIST.md`.

---

## 4. Live testing & sign-off

Live tests run real HTTP requests against a deployed instance and assert status + JSON shape.
Harness in `Modules/Api/Tests/live/`:

- `_lib.sh` — shared curl wrapper, JSON assertions (via `php`, no `jq`), the reusable
  **authentication matrix**, and a `summary` that sets the exit code (`0` = all passed).
- `<resource>.sh` — per-resource phases. Model new ones on `projects.sh`.

Config via env: `API_BASE_URL` (default `http://growcrm.co.zw`), `API_KEY` (default:
`TEMP_API_KEY_FOR_TESTING` in `Modules/Api/.env`), plus any ids the create/action steps need
(e.g. `CLIENT_ID`, `CATEGORY_ID`, `TEAM_USER_ID`).

**The agent runs the tests — the developer should not have to.** Everything needed to run the
scripts is already on disk and reachable by the agent: the test target is the local instance,
the database connection comes from the core `.env`, and the test API key
(`TEMP_API_KEY_FOR_TESTING`) comes from the module `.env` (`Modules/Api/.env`), which `_lib.sh`
reads automatically. The agent can therefore **initiate both** the automatic (`auto`/`readonly`)
and the interactive (`full`) runs itself without being handed credentials or asked to have the
developer launch the shell script. (The local `setup` DB is the same database the test target
serves, so the agent can also verify side effects — e.g. queued emails — directly in the DB.)

### Two run modes
Every resource script supports both:
- **Automatic** — no prompts, asserts + exit code; for CI / quick sign-off.
  - `./<resource>.sh auto` — full lifecycle **and actions** end-to-end (create → action
    endpoints → update → delete, cleaning up any clones), no pauses.
  - `./<resource>.sh readonly` — non-destructive only (auth, list, validation, notfound).
- **Interactive** — operator stays in the loop, confirming each step in the CRM UI.
  - `./<resource>.sh full` — the same lifecycle + actions, with UI-confirmation pauses.

Individual phases can be composed to drive a manual confirmation loop. No argument prints usage.

### Sign-off criteria
"Done" when:
1. `./<resource>.sh auto` exits `0`, **or** `readonly` + an interactive `full` run.
2. The create → … → delete lifecycle is **visually confirmed in the CRM UI** at least once,
   with no residual test data.
3. Behaviour matches `Docs/<resource>.md`.

Test data is labelled `ZZZ_APITEST_*` and the lifecycle deletes what it creates, so runs are
idempotent and leave the database clean.

### Exercising email options
When a create (or action) endpoint supports an email option (e.g. `send_email`), the test's
create step **must set it to `yes`** so email queuing is exercised — this is exactly the path
that caught the missing mail-boot in the `Authenticate` middleware (a reused mailer reading
`config('mail.data')` 500s if mail isn't booted). After the run, verify the row landed in
`email_queue` (the agent can query the local DB directly).

> **Caveat — email residue.** Some queued emails are not tagged with the resource (e.g. the
> welcome email sets only `emailqueue_to`), so `destroy<Resource>()` does **not** remove them and
> the HTTP-only harness cannot clean them either. Repeated runs therefore accumulate `email_queue`
> rows addressed to bogus `*@example.com` addresses that the email cron will try to send. The
> agent should delete these rows after a run (query by the `zzz_apitest_*@example.com` address),
> or the test instance should disable client email delivery
> (`settings_clients_disable_email_delivery`).

---

## 5. Multitenant

Drop-in for both the standard and SaaS builds. `RouteServiceProvider` appends Spatie's
`NeedsTenant` when `MT_TPYE` is set, so tenant resolution (by request domain) runs before auth;
all lookups then target the current tenant database. No app-level changes are required; assume
each tenant database already contains the module's tables.
