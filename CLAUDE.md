# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

"Spendly" — a Flask expense tracker built as a step-by-step teaching project. Server-rendered Jinja templates, SQLite, no frontend build step.

## Commands

```powershell
venv\Scripts\activate              # project virtualenv lives at ./venv (git-ignored)
pip install -r requirements.txt    # flask, werkzeug, pytest, pytest-flask
python app.py                      # dev server on http://127.0.0.1:5001 (debug=True, auto-reload)
pytest                             # run all tests (no tests/ directory exists yet)
pytest tests/test_x.py             # one test file
pytest tests/test_x.py::test_name  # single test
```

- The port is 5001, set in `app.py` (not the Flask default 5000).
- There is no linter, formatter or build step configured.

## Architecture

- `app.py` — single Flask module holding all routes. Replace a stub in place when implementing its step rather than adding a new route.
- `database/db.py` — SQLite data layer using `expense_tracker.db` in the project root. `get_db()` returns a connection (`sqlite3.Row`, foreign keys on; callers close it), `init_db()` creates the `users` and `expenses` tables, `seed_db()` inserts a demo user (`demo@spendly.com` / `demo123`) and 8 sample expenses once. `CATEGORIES` holds the fixed category list. `app.py` runs `init_db()` and `seed_db()` at import time inside `app.app_context()`.
- `templates/base.html` — shared layout (navbar, `{% block title %}`, `{% block head %}`, `{% block content %}`); page templates extend it. Navbar links are hard-wired to the `login`/`register` endpoints via `url_for`, so keep those endpoint names.
- `static/css/style.css` is the single stylesheet; `static/js/main.js` is an empty placeholder.

## Routes: implemented vs stub

| Route | Endpoint | Status |
|---|---|---|
| `/` | `landing` | Implemented — renders `landing.html` |
| `/register` | `register` | Implemented — renders the form only; no POST handling |
| `/login` | `login` | Implemented — renders the form only; no POST handling |
| `/terms` | `terms` | Implemented |
| `/privacy` | `privacy` | Implemented |
| `/logout` | `logout` | Stub — "coming in Step 3" |
| `/profile` | `profile` | Stub — "coming in Step 4" |
| `/expenses/add` | `add_expense` | Stub — "coming in Step 7" |
| `/expenses/<int:id>/edit` | `edit_expense` | Stub — "coming in Step 8" |
| `/expenses/<int:id>/delete` | `delete_expense` | Stub — "coming in Step 9" |

Stubs return a plain string and sit under the "Placeholder routes" comment in `app.py`. Update this table when a step lands.

## Code style

Match the existing code:

- Python: 4-space indentation, double-quoted strings, one blank line between a decorator-and-function pair and two between route groups. View functions are named after their endpoint (`landing`, `add_expense`) and each has a single `@app.route`.
- Routes are grouped under banner comments (`# ---- Routes ----`); keep new routes in the matching group.
- Templates: 4-space indentation, extend `base.html`, and build URLs with `url_for(...)` (including static files: `url_for('static', filename='css/style.css')`).
- Keep comments sparse; the existing code only comments section banners and step placeholders.

## Constraints

- Stack is Flask + SQLite with server-rendered Jinja templates. Don't introduce an ORM, a frontend framework, or a JS build step.
- Dependencies are pinned in `requirements.txt`; don't add or bump packages without being asked.
- Don't edit anything under `venv/`.
- Implement placeholder routes only when the user asks for that step; leave the other stubs as they are.
- Keep the dev server on port 5001.
- Currency is INR (₹), not USD. Use ₹ in templates, labels, seed data and tests; never `$`.

## Warnings and things to avoid

- Don't hardcode URLs in templates or redirects; renaming an endpoint breaks `url_for` references, especially `login`/`register` in the navbar.
- `/expenses/<id>/delete` and `/logout` are plain GET routes. When implementing them, make state changes POST-only rather than copying the GET stub signature.
- `app.py` sets no `SECRET_KEY`. Add one (from config or the environment, never a committed literal) before using `session` or `flash` for login.
- When implementing auth, hash passwords with `werkzeug.security`; never store them in plain text.
- Use parameterized SQL (`?` placeholders) in `database/db.py` and routes; never build queries with string formatting.
- `app.run(debug=True)` is for local development only; don't expose it beyond localhost.
- `landing.html` has uncommitted changes and is under active redesign (hero section matched to a mockup). Check `git status` and `git log` before restructuring or overwriting it.
- `.gitignore` already excludes `venv/`, `expense_tracker.db`, `.env` and `__pycache__/`; keep the SQLite file named `expense_tracker.db` so it stays untracked, and never commit secrets.
