# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Flask + SQLite web app for ACEest Fitness & Gym (ported from earlier Tkinter desktop scripts), with CI/CD in GitHub Actions and Jenkins. Python 3.12+.

## Commands

```bash
pip install -r requirements-dev.txt                 # runtime + pytest, pytest-cov, flake8
python app.py                                       # dev server on :5000 (PORT env overrides)

pytest                                              # all tests (pytest.ini adds -v, testpaths=tests)
pytest tests/test_api.py::test_bmi_endpoint         # single test
pytest --cov=fitness --cov=app --cov-report=term-missing

python -m compileall -q app.py fitness tests        # CI syntax gate
flake8 .                                            # max-line-length 110

docker build --target test -t aceest-fitness:test . && docker run --rm aceest-fitness:test   # tests in container
docker build -t aceest-fitness . && docker run -p 5000:5000 aceest-fitness                   # runtime (gunicorn)
```

CI (both `.github/workflows/main.yml` and `Jenkinsfile`) runs compileall + flake8, then builds the Docker `test` target and runs pytest inside it, then builds the `runtime` target; GitHub Actions also smoke-tests `/health` on the running container. Keep code flake8-clean or the pipeline stops before the Docker stage.

## Architecture

- **`app.py`** — `create_app(test_config)` app factory; all routes are defined inside `register_routes(app)`. A module-level `app = create_app()` exists for gunicorn (`app:app`), so importing `app` initializes the DB at `ACEEST_DB` or `instance/aceest.db`. App version: `APP_VERSION` env var, else the root `VERSION` file (exposed at `/health`, `/version`, and the home page footer); tests pin it to `9.9.9-test` via conftest.
- **`fitness/calculations.py`** — pure functions (calories, BMI, adherence) with no Flask/DB imports. They raise `ValueError` with user-facing messages; routes catch these and return `error(str(exc))` → `{"error": ...}` with 400. Follow this pattern for new validation.
- **`fitness/programs.py`** — in-memory `PROGRAMS` dict keyed by code (`FL`, `MG`, `BG`); `get_program` is case-insensitive and returns `None` for unknown codes. Calories = weight_kg × `calorie_factor`.
- **`fitness/db.py`** — per-request `sqlite3` connection on `flask.g` (`row_factory = sqlite3.Row`), closed on teardown. Schema is a `CREATE TABLE IF NOT EXISTS` script run on every app start; there are no migrations, so schema changes to existing tables need manual handling. Tables `progress` and `workouts` reference clients by `client_name` (text), not by id.
- Error conventions: 400 validation, 404 unknown client/program, 409 duplicate client name (from `sqlite3.IntegrityError` on the UNIQUE `name`). Membership status is computed as `"Expired"` at read time when `membership_end` is in the past.

## Testing

`tests/conftest.py` builds the app via `create_app({"TESTING": True, "DATABASE": tmp_path/...})`, so each test gets an isolated SQLite file. The `member` fixture creates client "Arjun" (FL, 80 kg) for progress/workout/membership tests. Unit tests target `fitness/`; API tests go through the Flask test client. README states the test count (currently 60) — update it when adding tests.

## Docker

Multi-stage `Dockerfile`: `test` stage copies the whole repo; the `runtime` stage (default) copies only `app.py`, `VERSION`, `fitness/` and `templates/`. Any new top-level module or directory needed at runtime must be added to the runtime `COPY` lines. Runs as non-root `appuser`.

## Conventions

- Branches: `feature/`, `fix/`, `test/`, `ci/`, `docs/` prefixes; merge to `main` via PR with `--no-ff`.
- Commit messages: `type: summary` (e.g. `feat: add BMI endpoint`, `ci: run pytest inside docker`).
