Great project. Below is a pragmatic, mainstream stack and workflow that fits your requirements, keeps things simple, and scales well with a team.

Summary of the approach
- Integration tests: Use Microsoft Playwright Test (TypeScript) to hit your running Spring Boot API via real HTTP. It’s very popular, fast, parallel by default, robust reporters, and easy for developers to understand. It works great even without any browser (pure API tests).
- Database for tests: PostgreSQL in Docker, reset to a known state for every run via Flyway clean+migrate and profile-specific seed scripts.
- Database for development: A separate PostgreSQL container and profile with its own seed data. Persistent volume for convenience.
- App build tool: Maven (Spring Boot plugin). Optional: you may use Testcontainers for Java-side unit/integration tests that stay in-JVM, but your primary E2E/integration tests run from Playwright outside the JVM.
- Orchestration: Justfile drives everything. Docker Compose for infra (databases) and simple shell for lifecycle.

Technologies and why they’re chosen
- Spring Boot 3.3+ (Java 22), Maven: current mainstream in the Java world.
- Flyway: mainstream DB migrations; simple, controlled resets; easy to have different seed sets by profile.
- PostgreSQL in Docker: your requirement.
- Docker Compose v2: simplest clean setup for local infra.
- Playwright Test (TypeScript): very popular, stable runner; first-class parallelism and fixtures; excellent reporters; many teams use it for API testing without UI.
    - Alternative if you prefer Python: pytest + httpx/requests + pytest-xdist. Equally mainstream. I’ll provide TS first, but you can swap to pytest with minimal changes.

Project structure (high-level)
- backend/
    - src/main/java/... (Spring Boot)
    - src/main/resources/application.yml
    - src/main/resources/db/migration/ (Flyway migrations)
    - src/main/resources/db/seeds/dev/ (dev seed scripts)
    - src/main/resources/db/seeds/e2e/ (e2e seed scripts)
    - pom.xml
- tests-e2e/ (TypeScript Playwright repo)
    - package.json
    - playwright.config.ts
    - tests/*.spec.ts
- infra/
    - docker-compose.dev.yml (postgres-dev)
    - docker-compose.e2e.yml (postgres-e2e)
- Justfile (root)

Spring Boot configuration
- Use Spring profiles: dev and e2e.
- Configure datasource and Flyway per profile.
- Make sure actuator health endpoint is enabled (useful to wait-for readiness).

Example application.yml (key parts)
```yaml
spring:
  application:
    name: my-api

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:postgresql://localhost:5433/myapp_dev
    username: myapp
    password: myapp
  flyway:
    enabled: true
    clean-disabled: false
    locations:
      - classpath:db/migration
      - classpath:db/seeds/dev
  jpa:
    hibernate:
      ddl-auto: validate
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      probes:
        enabled: true

---
spring:
  config:
    activate:
      on-profile: e2e
  datasource:
    url: jdbc:postgresql://localhost:5434/myapp_e2e
    username: myapp
    password: myapp
  flyway:
    enabled: true
    clean-disabled: false
    locations:
      - classpath:db/migration
      - classpath:db/seeds/e2e
  jpa:
    hibernate:
      ddl-auto: validate
```


Flyway
- Keep all schema migrations in db/migration (V1__init.sql, V2__..., etc.).
- Seed data per profile in db/seeds/dev and db/seeds/e2e as repeatable scripts (R__seed.sql). Example:
    - db/seeds/e2e/R__seed.sql creates a test user with known password and any reference data. This will be applied every migrate, ensuring deterministic state.
- For a “fresh” test database: do clean + migrate before each E2E run.

Docker Compose files
infra/docker-compose.dev.yml
```yaml
version: "3.9"
services:
  postgres-dev:
    image: postgres:16
    container_name: postgres-dev
    environment:
      POSTGRES_DB: myapp_dev
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: myapp
    ports:
      - "5433:5432"
    volumes:
      - myapp_dev_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp -d myapp_dev"]
      interval: 2s
      timeout: 2s
      retries: 30

volumes:
  myapp_dev_data:
```


infra/docker-compose.e2e.yml
```yaml
version: "3.9"
services:
  postgres-e2e:
    image: postgres:16
    container_name: postgres-e2e
    environment:
      POSTGRES_DB: myapp_e2e
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: myapp
    ports:
      - "5434:5432"
    tmpfs:
      - /var/lib/postgresql/data:rw
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp -d myapp_e2e"]
      interval: 2s
      timeout: 2s
      retries: 30
```

Notes:
- For dev we use a volume (persistent, convenient).
- For e2e we use tmpfs (ephemeral, pristine each run). Alternatively remove tmpfs and run Flyway clean—both approaches are fine. Using tmpfs makes runs fast and always clean.

Maven and running the API
- Use Spring Boot Maven plugin.
- For dev: mvn spring-boot:run -Dspring-boot.run.profiles=dev
- For e2e: mvn spring-boot:run -Dspring-boot.run.profiles=e2e
- Add a goal to force Flyway clean+migrate on start for the e2e workflow if you don’t use tmpfs. Two options:
    1) Use tmpfs and rely on migrate only (clean not needed).
    2) Or run a task before app start: mvn -P e2e -Dflyway.cleanDisabled=false flyway:clean flyway:migrate

Playwright Test (TypeScript) setup
tests-e2e/package.json
```json
{
  "name": "myapp-e2e",
  "private": true,
  "type": "module",
  "devDependencies": {
    "@playwright/test": "^1.47.0"
  },
  "scripts": {
    "test": "playwright test",
    "test:report": "playwright show-report"
  }
}
```


tests-e2e/playwright.config.ts
```textmate
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  timeout: 30_000,
  expect: { timeout: 5_000 },
  fullyParallel: true,
  reporter: [['list'], ['html', { outputFolder: 'playwright-report', open: 'never' }]],
  use: {
    baseURL: process.env.API_BASE_URL || 'http://localhost:8080',
    extraHTTPHeaders: {
      'Content-Type': 'application/json',
    },
  },
});
```


tests-e2e/tests/health.spec.ts
```textmate
import { test, expect, request } from '@playwright/test';

test('health is up', async ({ baseURL }) => {
  const api = await request.newContext({ baseURL });
  const res = await api.get('/actuator/health');
  expect(res.ok()).toBeTruthy();
  const json = await res.json();
  expect(json.status).toBe('UP');
});
```


tests-e2e/tests/auth.spec.ts
```textmate
import { test, expect, request } from '@playwright/test';

test('login works with default test user', async ({ baseURL }) => {
  const api = await request.newContext({ baseURL });
  const res = await api.post('/api/auth/login', {
    data: { username: 'testuser', password: 'password' },
  });
  expect(res.status()).toBe(200);
  const body = await res.json();
  expect(body.token).toBeTruthy();
});
```


Workflow: running E2E/integration tests
High level:
1) Start PostgreSQL for e2e.
2) Ensure DB is clean and seeded.
3) Start API with e2e profile.
4) Wait for actuator health.
5) Run Playwright tests (hitting real HTTP).
6) Tear down.

Justfile (root)
```
# .env files optional; we inline for clarity
set shell := ["bash", "-cu"]

# Common
api_port := "8080"
api_url := "http://localhost:{{api_port}}"

# ---- Development ----

dev-up:
    docker compose -f infra/docker-compose.dev.yml up -d
    echo "Waiting for postgres-dev..."
    until docker inspect --format='{{json .State.Health.Status}}' postgres-dev | grep -q healthy; do sleep 1; done
    echo "Starting API (dev)..."
    MAVEN_OPTS="--enable-preview" mvn -q -DskipTests spring-boot:run -Dspring-boot.run.profiles=dev

dev-down:
    docker compose -f infra/docker-compose.dev.yml down

dev-psql:
    docker exec -it postgres-dev psql -U myapp -d myapp_dev

# ---- Unit tests (Java) ----
unit:
    mvn -q -Dtest='*Test' test

# ---- E2E / Integration ----

e2e-up-db:
    docker compose -f infra/docker-compose.e2e.yml up -d
    echo "Waiting for postgres-e2e..."
    until docker inspect --format='{{json .State.Health.Status}}' postgres-e2e | grep -q healthy; do sleep 1; done

# If not using tmpfs, uncomment these lines to enforce clean+migrate
# e2e-migrate:
#     MAVEN_OPTS="--enable-preview" mvn -q -Dflyway.configFiles=src/main/resources/application.yml -Pe2e -Dflyway.cleanDisabled=false flyway:clean flyway:migrate

e2e-run-api:
    echo "Starting API (e2e)..."
    MAVEN_OPTS="--enable-preview" mvn -q -DskipTests spring-boot:run -Dspring-boot.run.profiles=e2e & echo $! > .api.pid
    echo "Waiting for API health at {{api_url}}/actuator/health..."
    until curl -fsS {{api_url}}/actuator/health >/dev/null; do sleep 1; done
    echo "API is up."

e2e-test:
    cd tests-e2e && API_BASE_URL={{api_url}} npm ci && npm test

e2e:
    just e2e-up-db
    just e2e-run-api
    just e2e-test
    just e2e-down

e2e-down:
    test -f .api.pid && kill $$(cat .api.pid) || true
    rm -f .api.pid
    docker compose -f infra/docker-compose.e2e.yml down -v
```


Notes:
- e2e target spins up DB, starts API in e2e profile, runs Playwright tests, then tears down.
- We use -v for e2e down to remove all data if you’re not using tmpfs.
- If you prefer the app in a Docker container too, you can add it to docker-compose.e2e.yml; the approach above keeps the developer feedback loop fast with mvn spring-boot:run.

Workflow: day-to-day development
- just dev-up: starts postgres-dev with a persistent volume.
- mvn spring-boot:run -Dspring-boot.run.profiles=dev (the Justfile target dev-up already does this).
- Use Postman, JetBrains HTTP Client, or curl to hit http://localhost:8080.
- Dev data is loaded by Flyway on startup from db/migration + db/seeds/dev (repeatables).
- For quick DB inspection: just dev-psql.

Workflow: unit tests (API only)
- just unit: runs fast JVM-based tests (JUnit, MockMvc/WebTestClient if needed).
- Optionally use Testcontainers inside unit/integration tests (Java) when you want DB-backed tests without booting the full external stack.

Best practices addressed
- Real network requests to real API: Playwright tests issue real HTTP calls to the port your API listens on.
- Separate DB for tests vs dev: Two Docker Compose files, two ports, two Spring profiles, two seed sets.
- Fresh test DB: Either tmpfs ephemeral storage or Flyway clean+migrate before each run; both are standard.
- Popularity and simplicity: Spring Boot + Maven + Flyway + Docker Compose + Playwright is mainstream and well-documented.
- Scriptability: The Justfile is the single entry point to orchestrate workflows.
- CI readiness: The same commands translate easily into CI steps (spin up postgres-e2e service, run app, run Playwright).

Optional enhancements
- OpenAPI (springdoc-openapi) and contract checking with Schemathesis (Python) or Dredd. This can be run alongside Playwright if you want schema-level guarantees.
- Test data management: If you need large fixtures, consider using Testcontainers for Postgres snapshots or pg_dump/restore to speed initialization, though Flyway seeds are usually sufficient.
- Use @DynamicPropertySource and Testcontainers for JVM tests to avoid manual port management in unit/integration tests.

Questions for you
1) Do you prefer TypeScript (Playwright) or Python (pytest) for the integration test suite? I can provide the Python variant if you prefer.
2) Do you want the API to run inside Docker for e2e as well, or keep it as a local mvn spring-boot:run? Both are fine; local run is faster for dev feedback.
3) Do you want ephemeral test DB via tmpfs (no clean step) or persistent test DB with Flyway clean+migrate each run?