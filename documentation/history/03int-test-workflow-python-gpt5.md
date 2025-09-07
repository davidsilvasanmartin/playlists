Below is a pragmatic, community-standard stack and workflow that fits your requirements, with minimal moving parts and clear responsibilities.

Recommended stack
- Framework: FastAPI (app), SQLAlchemy 2.x (ORM), Alembic (migrations)
- Test runner: pytest
- HTTP for integration tests: httpx (sync or async) sending requests to the locally running API (http://localhost:8000). Playwright is not necessary unless you also plan to test a browser UI.
- Database: PostgreSQL in Docker. Option A: docker compose to run a dedicated test Postgres service. Option B: testcontainers-python to spin up ephemeral Postgres per test session. Both are popular; compose is simpler to start with and works well when you run the API separately on your host.
- Environment/config: python-dotenv or pydantic-settings to load .env files (.env for dev, .env.test for integration tests)
- Project tool: uv (dependency and runner)
- Scripts: Justfile targets orchestrating DB startup, migrations, seeding, API startup, and tests
- Optional utilities:
    - pytest-xdist for parallel test runs (see isolation notes below)
    - dockerize or a small “wait-for” helper to wait until Postgres/API is ready
    - pre-commit for formatting and linting

High-level workflow
1) Start a clean test database container.
2) Run database migrations to head.
3) Seed initial data required for integration tests.
4) Start the API locally (uvicorn) configured to use the test DB.
5) Run pytest integration tests that call the API via HTTP (httpx).
6) Tear down the API and test DB.

Key design choices explained
- Why httpx over Playwright for API tests?: You want to hit the REST API with real network calls. httpx is the standard, lightweight choice for HTTP client testing in Python with pytest. Playwright is great for browser-driven E2E tests, not necessary for API-only scenarios.
- Why Alembic + dedicated seed script?: Alembic governs schema changes (best practice). Keep seed logic separate so tests can control which fixtures are loaded. This avoids coupling seed data to schema revisions.
- Why Docker Compose for test DB?: Simple, explicit, widely used. If you later need isolated, parallel, ephemeral DBs, consider switching to testcontainers-python (also standard and popular) to spin up a new DB per session or per worker.

Suggested project layout
- pyproject.toml
- .env               (development)
- .env.test          (test)
- alembic.ini
- alembic/
    - versions/
- app/
    - main.py
    - config.py
    - db.py
    - models.py
    - routers/
    - deps.py
- scripts/
    - seed_test_data.py
    - wait_for.py
- tests/
    - unit/
    - integration/
        - test_health.py
        - test_auth.py
- docker-compose.yml
- Justfile

Configuration
- .env (dev) example:
```textmate
DATABASE_URL=postgresql+psycopg://app_user:app_pass@localhost:5433/app_db
APP_ENV=development
```


- .env.test (integration) example:
```textmate
DATABASE_URL=postgresql+psycopg://test_user:test_pass@localhost:6543/test_db
APP_ENV=test
DEFAULT_TEST_USER_EMAIL=test@example.com
DEFAULT_TEST_USER_PASSWORD=Passw0rd!
```


- docker-compose.yml (dev db + test db; separate ports/users for clarity):
```yaml
version: "3.9"

services:
  db_dev:
    image: postgres:16
    ports:
      - "5433:5432"
    environment:
      POSTGRES_DB: app_db
      POSTGRES_USER: app_user
      POSTGRES_PASSWORD: app_pass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app_user -d app_db"]
      interval: 3s
      timeout: 3s
      retries: 10
    volumes:
      - db_dev_data:/var/lib/postgresql/data

  db_test:
    image: postgres:16
    ports:
      - "6543:5432"
    environment:
      POSTGRES_DB: test_db
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: test_pass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test_user -d test_db"]
      interval: 2s
      timeout: 2s
      retries: 30
    tmpfs:
      - /var/lib/postgresql/data

volumes:
  db_dev_data:
```


- pyproject.toml (relevant bits):
```toml
[project]
name = "your-fastapi-app"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
  "fastapi>=0.115",
  "uvicorn[standard]>=0.30",
  "sqlalchemy>=2.0",
  "psycopg[binary]>=3.2",
  "alembic>=1.13",
  "pydantic-settings>=2.4",
  "python-dotenv>=1.0",
]

[tool.uv]
dev-dependencies = [
  "pytest>=8.3",
  "httpx>=0.27",
  "pytest-xdist>=3.6",
  "faker>=25.0",
  # "testcontainers[postgres]>=4.8", # optional alternative to docker compose
]
```


- app/config.py (pydantic-settings):
```python
from pydantic_settings import BaseSettings
from pydantic import Field

class Settings(BaseSettings):
    app_env: str = Field("development", alias="APP_ENV")
    database_url: str = Field(..., alias="DATABASE_URL")
    default_test_user_email: str | None = Field(None, alias="DEFAULT_TEST_USER_EMAIL")
    default_test_user_password: str | None = Field(None, alias="DEFAULT_TEST_USER_PASSWORD")

    class Config:
        env_file = ".env"

settings = Settings()
```


- app/db.py (SQLAlchemy engine/session):
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.config import settings

engine = create_engine(settings.database_url, pool_pre_ping=True, future=True)
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False, future=True)
```


- app/main.py:
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok"}
```


Database migrations and seeding
- Alembic handles schema. Create an initial migration and a few subsequent ones as needed.
- Seeding for tests: write a small script that connects using DATABASE_URL and inserts your default user (idempotently or after a full reset). Keep it separate from Alembic versions.

Example seeding script (scripts/seed_test_data.py):
```python
import os
from sqlalchemy import text, create_engine

DATABASE_URL = os.getenv("DATABASE_URL")
DEFAULT_EMAIL = os.getenv("DEFAULT_TEST_USER_EMAIL", "test@example.com")
DEFAULT_PASSWORD = os.getenv("DEFAULT_TEST_USER_PASSWORD", "Passw0rd!")

def main():
    engine = create_engine(DATABASE_URL, future=True)
    with engine.begin() as conn:
        # Insert base data here. Use INSERT ... ON CONFLICT DO NOTHING if you made unique constraints.
        conn.execute(text("""
            INSERT INTO users (email, password_hash, is_active)
            VALUES (:email, crypt(:password, gen_salt('bf')), true)
            ON CONFLICT (email) DO NOTHING
        """), {"email": DEFAULT_EMAIL, "password": DEFAULT_PASSWORD})

if __name__ == "__main__":
    main()
```


Note: This example uses PostgreSQL’s crypt extension for brevity. In real apps, hash with passlib/bcrypt from Python so you don’t rely on DB extensions (widely considered best practice).

Integration tests with pytest + httpx
- tests/integration/test_health.py:
```python
import httpx
import os

BASE_URL = os.getenv("TEST_BASE_URL", "http://localhost:8000")

def test_health():
    r = httpx.get(f"{BASE_URL}/health", timeout=5.0)
    assert r.status_code == 200
    assert r.json() == {"status": "ok"}
```


Justfile
- A Justfile to orchestrate dev and test workflows. It uses uv to run Python, docker compose to manage the DB, Alembic for migrations, and a simple wait script to ensure readiness.

```textmate
# Variables
set dotenv-load := false

DEV_ENV := .env
TEST_ENV := .env.test
COMPOSE := docker compose

# Helper to wait for a TCP host:port to be ready
wait-for PORT HOST="127.0.0.1":
    uv run python scripts/wait_for.py {{HOST}} {{PORT}} 60

# ---- Development ----
dev-db-up:
    {{COMPOSE}} up -d db_dev

dev-db-down:
    {{COMPOSE}} stop db_dev

dev-db-destroy:
    {{COMPOSE}} down -v

dev-migrate:
    uv run --env-file {{DEV_ENV}} alembic upgrade head

dev-api:
    uv run --env-file {{DEV_ENV}} uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

dev: dev-db-up wait-for 5433 ; dev-migrate ; dev-api

# ---- Unit tests (no network, app-internal) ----
test-unit:
    uv run pytest -q tests/unit

# ---- Integration tests ----
testdb-up:
    {{COMPOSE}} up -d db_test

testdb-down:
    {{COMPOSE}} stop db_test

testdb-destroy:
    {{COMPOSE}} rm -sfv db_test

test-migrate:
    uv run --env-file {{TEST_ENV}} alembic upgrade head

test-seed:
    uv run --env-file {{TEST_ENV}} python scripts/seed_test_data.py

test-api-start:
    uv run --env-file {{TEST_ENV}} uvicorn app.main:app --host 0.0.0.0 --port 8000 & echo $! > .api.pid

test-api-stop:
    if [ -f .api.pid ]; then kill -9 `cat .api.pid` || true; rm .api.pid; fi

test-integration:
    # 1) Start test DB
    just testdb-up
    just wait-for 6543
    # 2) Reset schema + seed
    uv run --env-file {{TEST_ENV}} alembic downgrade base || true
    uv run --env-file {{TEST_ENV}} alembic upgrade head
    just test-seed
    # 3) Start API pointing to test DB
    just test-api-start
    just wait-for 8000
    # 4) Run integration tests
    TEST_BASE_URL=http://localhost:8000 uv run pytest -q -m "not unit" tests/integration
    # 5) Teardown
    just test-api-stop
    just testdb-down

test-integration-clean:
    just test-api-stop
    just testdb-destroy
```


The wait_for helper (scripts/wait_for.py):
```python
import socket, sys, time

def wait_for(host: str, port: int, timeout: int):
    t0 = time.time()
    while time.time() - t0 < timeout:
        s = socket.socket()
        s.settimeout(1)
        try:
            s.connect((host, port))
            s.close()
            return 0
        except Exception:
            time.sleep(0.5)
    return 1

if __name__ == "__main__":
    host = sys.argv[1]
    port = int(sys.argv[2])
    timeout = int(sys.argv[3]) if len(sys.argv) > 3 else 60
    sys.exit(wait_for(host, port, timeout))
```


Test data strategy
- Session-level reset: The Just target drops to base, upgrades to head, then runs seeds. This ensures a clean state per run.
- Seeding: Keep to minimal must-have data (e.g., default user, feature flags). Use Python with SQLAlchemy or raw SQL. Avoid mixing seed logic into Alembic.
- Idempotency: If you sometimes re-run tests against a running API, make seeds idempotent (ON CONFLICT DO NOTHING or upserts).

Parallelization notes (optional)
- If you want pytest -n auto for integration tests:
    - Simple approach: run tests sequentially for integration due to shared DB.
    - More advanced: allocate a separate schema per worker (e.g., DATABASE_SCHEMA=test_w{worker_id}), with the API reading the schema from env and you spawn an API per worker on a separate port. Or use testcontainers-python to launch 1 Postgres per worker, then one API per worker. Both are common but more setup. Start with sequential integration tests; run unit tests in parallel.

Alternative: testcontainers for the DB
- Instead of docker compose, you can manage the Postgres container from pytest using testcontainers[postgres], and still start the API locally pointing to that DB. This is also popular and gives fully ephemeral, isolated instances per test session. However, orchestration via Just + compose is usually simpler for team workflows.

Answers to your explicit questions
- Framework for integration testing via HTTP: pytest + httpx, hitting a real uvicorn server bound to localhost. Playwright is not needed unless you test a browser UI.
- Workflow to start DB, run API, run tests:
    1) docker compose up db_test
    2) wait for db to become healthy
    3) alembic downgrade base; alembic upgrade head
    4) seed data via a seed script
    5) start uvicorn app.main:app with .env.test
    6) wait for API port
    7) run pytest tests/integration
    8) stop API and stop/destroy db_test
- How to seed data:
    - A simple Python script (scripts/seed_test_data.py) that reads DATABASE_URL from .env.test and inserts required rows. Use SQLAlchemy or raw SQL. Keep it idempotent. Run it after migrations as part of the Just target.

Open questions for you
1) Do you anticipate browser E2E tests soon? If yes, we can add Playwright in a separate “e2e” layer and keep API integration tests as-is.
2) Do you want integration tests to run in parallel? If yes, we can outline per-worker schema or testcontainers-based isolation.
3) Any CI you plan to use (GitHub Actions, GitLab CI, etc.)? I can provide a ready-to-run workflow leveraging uv cache and Docker services.