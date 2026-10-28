# Plan: 01 — Database Setup

> Final destination requested by the user: `.claude/plans/01-database-setup.md` (project-relative). Plan mode only allows editing this file, so copy this content there as the first step after approval.

## Context

`database/db.py` is comments only and `app.py` never touches a database. Step 1 of the Spendly teaching project (spec: `.claude/specs/01-database-setup.md`) adds the SQLite data layer that auth, profile and expense steps (3–9) build on. No new routes, no new files, no new dependencies.

## Files to change

- `database/db.py` — implement `get_db()`, `init_db()`, `seed_db()`.
- `app.py` — import them and run `init_db()` + `seed_db()` at startup.

(`database/__init__.py` already exists; `werkzeug` is already in `requirements.txt`.)

## `database/db.py`

Imports: `sqlite3`, `datetime.date`, `os`, `werkzeug.security.generate_password_hash`.

**DB path.** The spec allows `spendly.db` or `expense_tracker.db`. Use **`expense_tracker.db`** because `.gitignore` already excludes it. Build the path from the file location (`os.path.join(os.path.dirname(os.path.dirname(__file__)), "expense_tracker.db")`) so it lands in the project root regardless of the working directory.

**`get_db()`**
- `sqlite3.connect(DB_PATH)`, set `row_factory = sqlite3.Row`, run `PRAGMA foreign_keys = ON` on every new connection, return it. Callers close it.

**`init_db()`**
- Open via `get_db()`, `executescript`/`execute` two `CREATE TABLE IF NOT EXISTS` statements, commit, close (use `try/finally`).
- `users`: `id INTEGER PRIMARY KEY AUTOINCREMENT`, `name TEXT NOT NULL`, `email TEXT NOT NULL UNIQUE`, `password_hash TEXT NOT NULL`, `created_at TEXT DEFAULT (datetime('now'))`.
- `expenses`: `id INTEGER PRIMARY KEY AUTOINCREMENT`, `user_id INTEGER NOT NULL REFERENCES users(id)`, `amount REAL NOT NULL`, `category TEXT NOT NULL`, `date TEXT NOT NULL`, `description TEXT`, `created_at TEXT DEFAULT (datetime('now'))`.
- Note the parentheses around `datetime('now')`: SQLite requires them for expression defaults.
- Do not add a `CHECK` on category; the spec only fixes the category list for seed data (keep the list as a module constant `CATEGORIES` for later steps to reuse).

**`seed_db()`**
- `SELECT COUNT(*) FROM users`; if > 0, close and return (idempotent).
- Insert the demo user with parameterized SQL: `Demo User`, `demo@spendly.com`, `generate_password_hash("demo123")`.
- Get the id via `cursor.lastrowid`, then `executemany` 8 expenses linked to it.
- Categories: all 7 appear at least once, plus one repeat (Food). Amounts are plain floats representing ₹ (per project rule: INR, never `$`). Dates: build from `date.today()` with `replace(day=...)`, formatted `YYYY-MM-DD`, using fixed days (1, 3, 5, 8, 12, 15, 18, 22) that exist in every month, so all dates fall in the current month.
- Example rows (description optional but include for most):

  | Category | Amount (₹) | Day | Description |
  |---|---|---|---|
  | Food | 450.00 | 1 | Groceries |
  | Transport | 120.00 | 3 | Metro recharge |
  | Bills | 1499.00 | 5 | Broadband |
  | Health | 650.00 | 8 | Pharmacy |
  | Entertainment | 800.00 | 12 | Movie tickets |
  | Shopping | 2200.00 | 15 | Clothes |
  | Other | 300.00 | 18 | Gift wrapping |
  | Food | 380.00 | 22 | Dinner out |

- Commit, close (`try/finally`). Only `?` placeholders; no f-strings or `%` formatting in SQL.

## `app.py`

- Add `from database.db import get_db, init_db, seed_db` after the Flask import (spec asks for `get_db` to be imported too, even though startup only calls the other two; it will be used by later steps. If an unused import bothers the user, drop it).
- After `app = Flask(__name__)`, add:
  ```python
  with app.app_context():
      init_db()
      seed_db()
  ```
  Placed at module level (not under `if __name__ == "__main__"`) so it also runs under `pytest`/`flask run`. Both functions are idempotent, so the debug reloader's second import is harmless. `get_db()` does not use `flask.g`, so the app context is a spec-mandated formality, not a functional requirement.
- Leave every route, including the placeholder stubs, untouched. Keep `app.run(debug=True, port=5001)`.

## Follow-ups outside the code

- Update `CLAUDE.md`: the Architecture line saying `database/db.py` is "comments only" and `app.py` "does not import it yet", plus the "SQLite (not yet implemented)" phrase in Project, will be stale after this step.

## Verification

1. `venv\Scripts\activate`, then `python app.py`: starts on http://127.0.0.1:5001 with no traceback; `expense_tracker.db` appears in the project root and `git status` does not list it (ignored).
2. Schema: `python -c "import sqlite3; c=sqlite3.connect('expense_tracker.db'); print(c.execute('select sql from sqlite_master').fetchall())"` shows both tables with the required constraints.
3. Seed: `select count(*) from users` is 1, `select count(*) from expenses` is 8, `select distinct category from expenses` returns all 7 categories, `password_hash` is not `demo123`, and all dates match `YYYY-MM-DD` in the current month.
4. Idempotency: restart the app (or call `init_db(); seed_db()` twice in a Python shell); counts stay at 1 and 8.
5. Constraints, via `get_db()`:
   - inserting a second user with `demo@spendly.com` raises `sqlite3.IntegrityError` (UNIQUE);
   - inserting an expense with `user_id=999` raises `sqlite3.IntegrityError` (FOREIGN KEY);
   - `row = conn.execute("select * from users").fetchone(); row["email"]` works.
6. Smoke test routes still render: `/`, `/login`, `/register` return 200 and the stub routes return their original strings.
7. Optional: add `tests/test_db.py` using a temp DB path (monkeypatching `DB_PATH`) covering steps 3–5, then run `pytest`. Not required by the spec; only add if the user wants it.
