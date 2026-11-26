# Week 14 : One Container Setup for Local Development and Render

## Goal

Let every team member start the full stack with one command, and deploy the
same backend image to Render without copying secrets into the repository.

## Relevant Context

The backend is built from `backend/Dockerfile`; the frontend has its own
`frontend/Dockerfile`. Render can deploy a Docker service described by a
`render.yaml` blueprint.

## Key Observation

Two failures were common when someone set up the project:

- a missing environment variable was discovered only when a request failed;
- the frontend started before the backend could reach the database, so the
  first page showed connection errors.

## Solution

**`compose.yaml`** runs the backend and the frontend. Required variables use
the `${VAR:?message}` form, so Compose refuses to start and prints what is
missing:

``` yaml
backend:
  environment:
    DATABASE_URL: ${DATABASE_URL:?Set DATABASE_URL to the PostgreSQL connection string in the project .env file}
    JWT_SECRET: ${JWT_SECRET:?Set JWT_SECRET in the project .env file}
  healthcheck:
    test: [CMD, python, -c, "from urllib.request import urlopen; urlopen('http://127.0.0.1:8000/api/health', timeout=5)"]
    interval: 30s
    start_period: 60s
frontend:
  depends_on:
    backend:
      condition: service_healthy
```

The health endpoint checks the database, not just the process:

``` python
@app.get("/api/health")
def health():
    with engine.connect() as connection:
        connection.execute(text("SELECT 1"))
    return {"status": "ok"}
```

**`render.yaml`** deploys the same Dockerfile with
`healthCheckPath: /api/health`. Every secret (database URL, JWT secret,
Supabase key, email key) is declared with `sync: false`, which means its
value is entered once in the Render dashboard and never committed. Safe
defaults are fixed in the file, for example `APP_ENV: production`,
`AUTO_MIGRATE: "false"` and `PAYMENT_PROVIDER: disabled`.

**Because**

A health check that runs `SELECT 1` makes "healthy" mean "able to serve
requests". Failing fast on a missing variable turns a confusing runtime error
into a one-line message at startup.
