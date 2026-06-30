# Rare API — Claude Code Session Briefing

## Documentation Index

Before diving into code, check these files first:

| Document | What it covers |
|---|---|
| `docs/PROJECT_GUIDE.md` | Complete architecture reference: models, views, serializers, business rules, feature map, request flows, decisions |
| `docs/architecture-overview.md` | Beginner-friendly explainer with code examples, reading order, debugging table |
| `docs/engineering-notes.md` | Dated problem journal: what broke, how it was fixed, lessons learned |

---

## Project Context

Rare is a full-stack blogging platform built as an NSS educational project. Two sub-projects:

- `rare-api/` — Django 4.2 + DRF backend, PostgreSQL 16 via Docker
- `rare-client/` — React 18, Bulma CSS, Create React App

They communicate via a JSON REST API. No shared code between them.

**User roles:** Authors (write posts) and Admins (`is_staff=True`, moderate + manage). Key business rules: posts require admin approval before going public; demoting/deactivating an admin requires two separate admin votes.

---

## Current Environment Baseline

| Item | Value |
|---|---|
| Python | 3.13.7 |
| Virtual environment | `rare-api/.venv` |
| Django | 4.2.30 |
| Django REST Framework | 3.17.1 |
| Database | PostgreSQL 16 (Docker) |
| Docker host port | 5433 → container port 5432 |
| Django runs on | port `8088` (hardcoded in frontend — do not change) |
| React runs on | port `3000` |
| Test baseline | 18/18 passing |
| Seed data | 314 objects loaded via `loaddata` |
| Admin credentials | `admin_sarah` / `password`, `admin_marcus` / `password` |

**Critical:** `src/managers/api.js` hardcodes `http://localhost:8088`. Django must run on port 8088 or all API calls fail silently.

**Critical:** Docker maps host port 5433 to container port 5432. `settings.py` must say `PORT: '5433'`. Do not change to 5432 unless port 5432 is free on this machine.

---

## Starting the Development Environment

```bash
# Terminal 1 — rare-api
docker compose up -d db
.venv/bin/python manage.py runserver 8088

# Terminal 2 — rare-client
npm start
```

**First time only** (empty database):
```bash
.venv/bin/python manage.py migrate
.venv/bin/python manage.py loaddata rareapi/fixtures/initial_data.json
```

**Do not run loaddata on a database that already has data** — it creates duplicates.

Shutdown order (reverse dependency):
```bash
# 1. Ctrl+C React terminal
# 2. Ctrl+C or kill Django terminal
# 3. docker compose stop   (preserves volume and data)
#    docker compose down -v (destroys volume — data gone)
```

---

## Teaching Mode

This project is used for learning. When explaining concepts:

- Build the mental model before the detail ("what is this for?" before "how does it work?")
- Use analogies — the codebase already has established ones:
  - **Relay race** — each layer (component → manager → fetch → view → serializer → ORM → DB) passes the baton, knows nothing about the others' internals
  - **Wristband** — token auth: prove identity once at login, show wristband at every door thereafter
  - **Holding room** — post approval: Author posts wait in `approved=False` until an Admin admits them
  - **Safe with two keys** — two-admin vote: first admin deposits a key (DemotionQueue row), second admin uses both keys to execute the action
  - **Recipe card** — migrations: sequential steps that rebuild the database schema from scratch
- Explain the "why" behind decisions, not just the "what"
- Prefer evidence from the code over assertion

---

## Git Workflow

```
main        ← stable; never commit directly here
  |
develop     ← integration branch; PRs merge here
  |
feature/*, fix/*, docs/*, maint/*   ← all work branches
```

Always branch from `develop`. Open a PR into `develop`. Never commit directly to `main` or `develop`.

Branch naming: `feature/description`, `fix/description`, `docs/RARE-NNN-description`, `maint/description`

---

## Testing

```bash
cd rare-api
.venv/bin/python -m pytest rareapi/tests/ -v
```

**18 tests, 18 passing.** This baseline must be preserved. If a change causes a failure, stop and diagnose — do not modify tests to make them pass.

Current coverage: login, registration, `/me`, admin demotion queue workflow.
Not yet covered: post CRUD, comments, categories, tags, reactions, subscriptions, image uploads, search.

---

## Current Roadmap

| Phase | Goal | Status |
|---|---|---|
| Phase 0 | Environment stabilization | ✅ Complete |
| Phase 1 | Backend verified running | ✅ Complete |
| Phase 2 | Frontend integration verified | ✅ Complete |
| Phase 3 | Validate core user workflows end-to-end | Not started |
| Phase 4 | Documentation improvements | ✅ Complete |
| Phase 5 | Testing and quality improvements | Not started |

**Phase 5 candidates:**
- Introduce `.env` for frontend API URL
- Add tests for post CRUD, comments, subscriptions
- Audit frontend managers for missing error handling (`res.ok` checks)
- Verify image upload endpoints work end-to-end
- Add `unique_together` constraint to `PostReaction`
