# Data Model AI Prompts

## 1. Find the Database Connection Details

Field: Host
Value: localhost
Source file: ../learn-ops-infrastructure/docker-compose.yml — the database service
  publishes port 5432:5432 to the host (inside the Docker network the API reaches
  it via hostname database, from learn-ops-api/.env → LEARN_OPS_HOST=database, but
  pgAdmin on your host machine should use localhost)
────────────────────────────────────────
Field: Port
Value: 5432
Source file: ../learn-ops-infrastructure/docker-compose.yml (port mapping
  "5432:5432") — matches LEARN_OPS_PORT=5432 in learn-ops-api/.env
────────────────────────────────────────
Field: Database
Value: learningplatform
Source file: ../learn-ops-infrastructure/.env → POSTGRES_DB — matches
  LEARN_OPS_DB=learningplatform in learn-ops-api/.env
────────────────────────────────────────
Field: Username
Value: learnops
Source file: ../learn-ops-infrastructure/.env → POSTGRES_USER — matches
  LEARN_OPS_USER in learn-ops-api/.env
────────────────────────────────────────
Field: Password
Value: learnops123
Source file: ../learn-ops-infrastructure/.env → POSTGRES_PASSWORD — matches
  LEARN_OPS_PASSWORD in learn-ops-api/.env


## 2. Identify the Database Type

The app uses PostgreSQL.

- Engine declaration: LearningPlatform/settings.py:197 — 'ENGINE': 'django.db.backends.postgresql_psycopg2'
- Version (local/dev): Postgres 16, from ../learn-ops-infrastructure/docker-compose.yml:3 — image: postgres:16 (the database service)
- Version (Digital Ocean/production): Postgres 12, from config/learn-ops-api.yaml:8-12 — engine: PG, version: "12" in the managed-database config for the DO deployment

So local dev runs Postgres 16 via Docker, while the deployed DigitalOcean managed database is pinned to Postgres 12 — worth noting if you're testing version-specific SQL features locally that might not exist in prod.

## 3. Map the ORM to the Database



## 4. Generate a Database Diagram



## 5. Find Relationship Examples
