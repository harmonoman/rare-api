# Rare API — Project Context for Claude Code

## Project Context

Rare is a full-stack blogging platform built as an educational project at
Nashville Software School (NSS). It is used to teach intermediate software
engineering students how to build, understand, and maintain a real-world
web application.

The learning goals are:

- Understanding how a backend API and a frontend client communicate
- Building professional engineering habits: read before you change, test
  before you ship, document what you decide
- Learning Django, Django REST Framework, React, and PostgreSQL in a
  realistic project context
- Practicing version control, branching strategies, and code review

This project is intentionally handed to students in an incomplete or
undocumented state. Part of the learning exercise is the process of
inheriting an unfamiliar codebase and making it understandable and runnable.


## Architecture Overview

### Backend — rare-api

| Layer | Technology | Purpose |
|---|---|---|
| Web framework | Django 4.2 | URL routing, ORM, admin, migrations |
| API layer | Django REST Framework 3.x | Serializers, API views, token auth |
| Database | PostgreSQL 16 (Docker) | Persistent data storage |
| Auth | DRF Token Authentication | Token stored in localStorage on the client |
| Tests | pytest + pytest-django | Unit and integration tests |

**Domain models:**

- `RareUser` — extends Django's `AbstractUser`; adds `bio`, `profile_image_url`, `created_on`
- `Post` — belongs to a user and a category; has `approved` boolean for moderation workflow
- `Category` — groups posts
- `Tag` — labels that can be applied to posts via `PostTag`
- `Comment` — belongs to a post and an author
- `Reaction` — emoji reaction type (admin-created)
- `PostReaction` — join table: which user reacted to which post with which reaction
- `PostTag` — join table: which tags are on which post
- `Subscription` — a user (follower) subscribes to an author
- `DemotionQueue` — supports the two-admin vote workflow for admin demotion/deactivation

**Important business rule — two-admin vote:**

Deactivating an admin or demoting an admin to Author requires approval from
two separate admins. The first vote queues the action in `DemotionQueue`;
the second vote executes it and clears the queue. Regular users (Authors)
can be acted on immediately by any single admin.

This logic lives entirely in `rareapi/services/admin_actions.py` and is
tested directly without going through the HTTP layer.

**Backend structure:**

```
rareapi/
  models/       — one file per model, imported via __init__.py
  views/        — one file per resource group
  serializers/  — one file per resource group
  services/     — business logic separated from views
  tests/        — pytest test files
  migrations/   — 5 app migrations (0001 through 0005)
  fixtures/     — initial_data.json seed data
rareproject/
  settings.py   — Django configuration
  urls.py       — root URL conf (includes rareapi.urls)
```

### Frontend — rare-client

| Layer | Technology | Purpose |
|---|---|---|
| Framework | React 18 | Component-based UI |
| Routing | React Router DOM v6 | Client-side navigation |
| CSS | Bulma | Utility CSS framework |
| API layer | managers/ | One JS module per resource; all fetch calls live here |
| Auth | localStorage | Token and user ID stored after login |

**Frontend structure:**

```
src/
  Rare.js              — root component; manages token, userId, isAdmin state
  views/
    ApplicationViews.js — all route definitions
    Authorized.js       — redirects unauthenticated users to /login
    AdminOnly.js        — blocks non-admin users from admin routes
  components/          — organized by feature (posts, comments, categories, etc.)
  managers/            — API communication layer (one file per resource)
    api.js             — exports API base URL and shared authHeader()
  utils/
    dates.js           — date formatting helpers
```

**Authentication flow:**

1. User logs in → POST `/login` → receives `{ token, user_id, is_staff }`
2. Token and user_id saved to `localStorage`
3. All subsequent API calls include `Authorization: Token <token>` header
4. On page refresh, `Rare.js` calls `/me` to re-derive `isAdmin` from the server
5. If token is invalid, `/me` fails and auth state is cleared (user is logged out)


## Current Environment Baseline

This is the known-good state as of Phase 0 completion.

### Backend

| Item | Value |
|---|---|
| Python | 3.13.7 |
| Virtual environment | `rare-api/.venv` |
| Django | 4.2.30 |
| Django REST Framework | 3.17.1 |
| Database engine | PostgreSQL 16 |
| Docker host port | 5433 |
| Container port | 5432 |
| Django connects to | `localhost:5433` |
| Django runs on | port `8088` |
| Test result baseline | 18/18 passing |

### Frontend

| Item | Value |
|---|---|
| React | 18.2 |
| Dev server | `localhost:3000` |
| API base URL | `http://localhost:8088` (hardcoded in `src/managers/api.js`) |

**Critical:** The frontend has `http://localhost:8088` hardcoded. Django must
run on port 8088 or every API call will fail. There is no `.env` file.

**Critical:** The Docker container maps host port 5433 to container port 5432.
Django's `settings.py` must use `PORT: '5433'`. This was changed from the
original `5432` to avoid conflict with a local EDB PostgreSQL 16 installation.

### Starting the development environment

```bash
# 1. Start the database
cd rare-api
docker compose up -d db

# 2. Confirm it is healthy
docker compose ps

# 3. Run migrations (first time, or after pulling new migrations)
.venv/bin/python manage.py migrate

# 4. Load seed data (first time only)
.venv/bin/python manage.py loaddata rareapi/fixtures/initial_data.json

# 5. Start the Django server
.venv/bin/python manage.py runserver 8088

# 6. In a separate terminal — start the frontend
cd ../rare-client
npm start
```


## Development Philosophy

### The core workflow

```
Understand → Inspect → Explain → Plan → Modify → Test → Document
```

Before modifying any code, answer these questions:

1. **What layer is involved?** (Database? Model? View? Serializer? React component? Manager?)
2. **What is the expected behavior?**
3. **What is the actual behavior?**
4. **What is the smallest change that fixes the problem?**
5. **What could this change break?**

Do not jump to solutions. Gather evidence first.

### Change philosophy

- Change one thing at a time
- Verify the result before changing the next thing
- Prefer small reversible changes over large rewrites
- If tests were passing before your change and fail after, your change broke something


## Teaching Mode

This project is used for learning. When explaining concepts:

- Build the mental model before the detail ("what is this for?" before "how does it work?")
- Use analogies to connect unfamiliar concepts to familiar ones
- Explain the "why" behind decisions, not just the "what"
- Highlight the few foundational ideas that unlock understanding of many others

Useful mental models for this project:

- **The request journey:** Browser → React → manager → fetch → Django URL → view → serializer → model → database → back up the chain
- **Migrations as a recipe:** The migrations folder is a step-by-step recipe for rebuilding the database from scratch
- **Virtual environments as private toolboxes:** Each project gets its own isolated set of tools so they don't interfere with each other
- **Docker ports as apartment forwarding:** `5433:5432` means "deliver mail arriving at apartment 5433 to apartment 5432 inside the container"
- **Token auth as a wristband:** You prove your identity once (login), get a wristband (token), and show the wristband at every door thereafter


## Debugging Approach

### The application stack

```
Browser
  ↓
React Components       ← rendering and user interaction
  ↓
API Managers (JS)      ← fetch calls, headers, URL construction
  ↓
HTTP Request           ← the actual network call
  ↓
Django URL Router      ← which view handles this URL?
  ↓
Django View            ← authentication, permissions, business logic
  ↓
Serializer             ← validation, Python ↔ JSON conversion
  ↓
Model / ORM            ← database query construction
  ↓
PostgreSQL             ← data storage and retrieval
```

### Debugging protocol

1. **Reproduce** — confirm you can consistently trigger the problem
2. **Gather evidence** — browser console, network tab, Django server logs
3. **Identify the layer** — where in the stack above does the problem occur?
4. **Explain the root cause** — write it out in plain English before touching code
5. **Propose the smallest fix** — do not over-engineer
6. **Apply and verify** — make the change, confirm the problem is gone
7. **Run the tests** — confirm nothing else broke


## Git Workflow

### Branch structure

```
main        ← stable; never commit directly here
  |
develop     ← integration branch; PRs merge here
  |
feature/*, fix/*, docs/*, maint/*   ← all work branches
```

### Branch naming

| Type | Pattern | Example |
|---|---|---|
| New feature | `feature/<description>` | `feature/post-search-endpoint` |
| Bug fix | `fix/<description>` | `fix/comment-delete-permission` |
| Documentation | `docs/<description>` | `docs/RARE-001-create-project-context-documentation` |
| Maintenance | `maint/<description>` | `maint/upgrade-drf-version` |

### Rules

- Never commit directly to `main` or `develop`
- Always branch from `develop`
- Open a PR from your branch into `develop`
- `main` only receives merges from `develop` at release points


## Testing Expectations

### Running tests

```bash
cd rare-api
.venv/bin/python -m pytest rareapi/tests/ -v
```

### Current baseline

18 tests, 18 passing. This is the established baseline from Phase 0.

**These 18 tests must continue to pass.** If your change causes a test
failure, stop and diagnose before continuing. Do not delete or modify tests
to make them pass — understand why they are failing.

### Test coverage

Current tests cover:
- Login (valid credentials, bad password, inactive user, admin flag)
- Registration
- `/me` endpoint (authenticated, unauthenticated, invalid token)
- Admin demotion queue workflow (first vote queues, second executes,
  duplicate vote rejected, invalid role rejected, last-admin guard)

**Not yet covered:** Post CRUD, comments, categories, tags, reactions,
subscriptions, image uploads, search. These are candidates for Phase 5.

### When adding features

Write the test before or alongside the feature. Do not ship a feature
without at least one test that confirms the core behavior.


## Known Engineering Decisions

### Docker PostgreSQL port remapped to 5433

**Problem:** The development machine has EDB PostgreSQL 16 installed and
running on `localhost:5432`. Docker could not bind to the same host port.

**Solution:** Changed `docker-compose.yml` port mapping from `5432:5432`
to `5433:5432`. Updated `rareproject/settings.py` `PORT` from `5432`
to `5433`.

**Why this works:** Docker port syntax is `HOST_PORT:CONTAINER_PORT`.
The container's internal PostgreSQL still runs on 5432. Only the host-side
entry point changed. The system PostgreSQL and the Docker PostgreSQL now
live on separate host ports and are completely unaware of each other.

**If you are on a machine where port 5432 is free**, you can revert both
files to use `5432:5432` and `PORT: '5432'`.

### Django 4.2 in a virtual environment

**Problem:** The global Python environment had Django 6.0 installed, two
major versions ahead of the Pipfile's `~=4.2` requirement. Running tests
under the wrong Django version means failures cannot be attributed to
application bugs vs. version incompatibility.

**Solution:** Created `.venv` virtual environment, installed Django 4.2.30
per the Pipfile constraint. All project commands use `.venv/bin/python`.

**Note:** Python 3.13 is not officially listed in Django 4.2's support
matrix (which ends at 3.12), but works in practice. If a Python 3.13
specific issue surfaces, the migration path is Django 5.2 LTS.

### API base URL hardcoded in frontend

`src/managers/api.js` exports `export const API = "http://localhost:8088"`.

This is hardcoded — there is no `.env` file. The Django dev server must
run on port 8088. This is a known limitation acknowledged for Phase 5
improvement (introduce environment variable).


## Current Roadmap

| Phase | Goal | Status |
|---|---|---|
| Phase 0 | Environment stabilization | ✅ Complete |
| Phase 1 | Backend verified running | In progress |
| Phase 2 | Frontend integration verified | Not started |
| Phase 3 | Validate core user workflows end-to-end | Not started |
| Phase 4 | Documentation improvements | In progress (this ticket) |
| Phase 5 | Testing and quality improvements | Not started |

### Phase 1 remaining tasks

- [ ] Load fixture data: `python manage.py loaddata rareapi/fixtures/initial_data.json`
- [ ] Start Django on port 8088 and verify `/categories` returns JSON
- [ ] Verify `/reactions` returns seed data
- [ ] Confirm all auth endpoints respond correctly with curl

### Phase 5 candidates

- Introduce `.env` for frontend API URL
- Add tests for post CRUD and approval workflow
- Add tests for comments and subscriptions
- Audit frontend managers for missing error handling
- Verify image upload endpoints work end-to-end
