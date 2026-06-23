# Engineering Notes — Rare API

This file is a living engineering journal. Each entry documents a problem
encountered, how it was investigated, how it was resolved, and what was
learned. Future engineers (and future Claude Code sessions) can read this
to understand why things are the way they are.

Entries are dated and ordered newest-first.

---

## 2026-06-22 — Dependency Environment Recovery

### Problem

The project could not be run reliably because dependencies were installed
globally on the developer machine rather than in a project-specific
virtual environment.

The machine had **Django 6.0** installed globally. The `Pipfile` declared
`django = "~=4.2"` — meaning Django 4.2 or any compatible 4.x patch, but
nothing jumping to version 5 or beyond.

Running the project under Django 6.0 created an ambiguity problem: if
tests failed or the server threw errors, there was no way to determine
whether the cause was:

- A bug in the application code, or
- An incompatibility introduced by the Django version difference

This ambiguity is unacceptable during environment stabilization. The
principle is: eliminate variables before diagnosing problems.

### Investigation

Checked the current Python environment:

```
python3 --version        → 3.13.7
python3 -m django --version  → 6.0.6
which python3            → /usr/local/bin/python3 (system Python, no venv)
```

Confirmed the machine was running Django two major versions ahead of the
project's declared requirement, and with no project isolation at all.

### Root Cause

The earlier setup work installed packages directly into the system Python
using `pip install` rather than into a project-scoped virtual environment.
This is a common mistake when `pipenv` is not available, but it leaves the
project's dependencies entangled with whatever else is installed on the
machine.

### Resolution

Created a project-specific virtual environment:

```bash
python3 -m venv .venv
```

Installed all packages per the Pipfile constraints:

```bash
.venv/bin/pip install "django~=4.2" "djangorestframework~=3.15" \
  "django-cors-headers~=4.3" "psycopg2-binary~=2.9" pytest pytest-django
```

Result: Django 4.2.30 installed, isolated from the system.

Verified:

```bash
.venv/bin/python -m django --version   → 4.2.30
```

Django setup and model import confirmed working:

```bash
DJANGO_SETTINGS_MODULE=rareproject.settings .venv/bin/python -c \
  "import django; django.setup(); from rareapi.models import RareUser; print('OK')"
```

### Lesson

A `Pipfile` or `requirements.txt` is a contract, not a suggestion. It
declares the exact versions of tools the project was built and tested
against. Honoring that contract by using a virtual environment means:

- Any failure you encounter is a project failure, not an environment failure
- Another developer on a different machine can reproduce your exact environment
- The project's behavior is predictable regardless of what else is installed

All future commands for this project use `.venv/bin/python`, not the
system Python.

### Note on Python 3.13

Python 3.13 is not in Django 4.2's official support matrix (which ends
at Python 3.12). However, Django 4.2.30 works in practice on Python 3.13
as confirmed by successful `django.setup()` and all 18 tests passing.

If a Python 3.13 specific issue is ever encountered, the upgrade path is
Django 5.2 LTS — which officially supports Python 3.10 through 3.13 and
requires no application code changes.

---

## 2026-06-22 — Docker PostgreSQL Port Conflict

### Problem

The project's Docker PostgreSQL container could not start. The error was:

```
Ports are not available: exposing port TCP 0.0.0.0:5432 -> 0.0.0.0:0:
listen tcp 0.0.0.0:5432: bind: address already in use
```

### Investigation

Checked what was running on port 5432:

```bash
ps aux | grep postgres
```

Found:

```
/Library/PostgreSQL/16/bin/postgres -D /Library/PostgreSQL/16/data
```

This is an EDB (EnterpriseDB) PostgreSQL 16 installation — a separate
system-level PostgreSQL that starts automatically on machine boot and
listens on TCP port 5432.

Confirmed with netstat:

```bash
netstat -an | grep 5432 | grep LISTEN
→ tcp4  *.5432  *.*  LISTEN
→ tcp6  *.5432  *.*  LISTEN
```

Also discovered that `crypto_postgres` — a Docker container from an
unrelated project — was also running on port 5432. That container was
stopped first, but the EDB system PostgreSQL remained.

Checked port 5433 — completely free. No process claiming it.

### Root Cause

Docker's port mapping `"5432:5432"` instructs Docker to bind TCP port 5432
on the host machine and forward traffic to port 5432 inside the container.

Two processes cannot bind the same TCP port simultaneously. The EDB
PostgreSQL had already claimed 5432, so Docker's attempt to also claim it
was rejected by the operating system.

The EDB PostgreSQL could not be stopped without `sudo` — it is a
system-level service. Stopping it would also risk breaking other
applications on the machine that depend on it.

### Resolution

The fix separates the host port from the container port.

In `docker-compose.yml`, changed:

```yaml
ports:
  - "5432:5432"
```

to:

```yaml
ports:
  - "5433:5432"
```

In `rareproject/settings.py`, changed:

```python
'PORT': '5432',
```

to:

```python
'PORT': '5433',
```

### Why This Works

Docker port syntax is `HOST_PORT:CONTAINER_PORT`.

- `HOST_PORT` is the port number on the developer's machine
- `CONTAINER_PORT` is the port inside the Docker container

The PostgreSQL process inside the container still runs on port 5432 —
nothing inside the container changed. Only the host-side entry point
changed from 5432 to 5433.

The connection chain is now:

```
Django (settings.py PORT=5433)
  ↓
localhost:5433  (host machine — now free, not claimed by anything)
  ↓
Docker networking bridge
  ↓
container:5432  (PostgreSQL inside the container)
```

The EDB PostgreSQL on port 5432 is completely unaffected. Two PostgreSQL
instances exist on the machine and neither knows about the other.

### Verification

```bash
docker compose up -d db
docker compose ps
→ rare-api-db-1   Up   0.0.0.0:5433->5432/tcp

pg_isready -h localhost -p 5433 -U rare_user -d rare
→ localhost:5433 - accepting connections

.venv/bin/python manage.py migrate
→ 27 migrations applied, all OK

.venv/bin/python -m pytest rareapi/tests/ -v
→ 18 passed in 5.28s
```

### Lesson

Docker port mappings separate host networking from container networking.
The container's internal configuration never needs to change — only the
host-side mapping. This is the correct and minimal intervention.

If you are working on a machine where port 5432 is available, both
`docker-compose.yml` and `settings.py` can be reverted to use 5432. The
application itself does not care which port number is used — only that
Django and Docker agree on the same host port.

---

## 2026-06-22 — Initial Codebase Reconnaissance

### Context

The project was handed over with no useful documentation. Both README files
contained a single empty line. The goal was to understand the codebase
before touching anything.

### Findings

**Backend domain:** A blogging platform. Core resources are Post, Category,
Tag, Comment, Reaction, and RareUser. Posts have an `approved` boolean
supporting a content moderation workflow. Users are either Authors or Admins
(`is_staff` maps to admin).

**Most interesting feature:** The admin demotion queue. Demoting or
deactivating an admin user requires two separate admins to vote. This logic
lives in `rareapi/services/admin_actions.py` as a pure Python service —
no HTTP layer, no view dependencies. It returns a typed `ActionResult`
dataclass that views translate to HTTP responses. This is a clean
architectural pattern worth understanding.

**Frontend API dependency:** `src/managers/api.js` exports:

```js
export const API = "http://localhost:8088"
```

This is hardcoded. Django must run on port 8088. There is no environment
variable, no `.env` file, no proxy configuration. This is the single most
important piece of environment knowledge for running the full stack.

**Migration history:** 5 application migrations exist (0001 through 0005).
Of note: migration 0001 added an `active` field to `RareUser` that
migration 0005 later removed. The current `RareUser` model does not have
this field. This is evidence of normal iterative development and means all
5 migrations must be applied in order on a fresh database.

**Fixture data:** `rareapi/fixtures/initial_data.json` exists. Expected to
contain seed data for reactions (emoji reaction types) and initial
categories. Must be loaded after migrations on a fresh database or those
features will appear empty.

**CLAUDE.md in .gitignore:** The project's `.gitignore` explicitly listed
`CLAUDE.md`. This was intentional by the course instructor — students were
meant to discover and write their own project context documentation.
That document has now been created as part of Phase 0 completion.

### Lesson

Never modify an unfamiliar codebase without first reading it. Time spent
reading is not wasted — it is the investment that makes every subsequent
hour of work faster and more confident.

The recommended reading order for this project (and any Django project):

1. Models — understand the data
2. Migrations — understand how the data evolved
3. URLs — understand the API surface
4. Views — understand how requests are handled
5. Serializers — understand what JSON looks like in and out
6. Services — understand the business logic
7. React routes — understand what pages exist
8. React components — understand the UI layer
9. API managers — understand how the frontend calls the backend
