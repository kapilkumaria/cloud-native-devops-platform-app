# Manual Container Testing

## Objective

Document the earlier, independent-container validation of the Cloud-Native DevOps Platform application **before Docker Compose was introduced**. This is historical test evidence, not a claim that the November 2026 frontend change or current EKS manifests were tested by these commands.

## Scope and Validation Status

The original validation established that PostgreSQL and the FastAPI backend could run in separate Docker containers on a shared Docker network, that the backend could connect to the database, and that the API, Swagger documentation, and seeded-user authentication worked in the tested environment.

> **Security correction (November 2026):** Earlier examples embedded a demonstration password and a password-bearing `DATABASE_URL` directly in terminal commands. Those examples are **not suitable for reuse**. The procedure below uses runtime-provided secrets instead. The exact environment keys accepted by the current backend image must be checked against its source and environment template before executing a fresh test.

## Prerequisites

- Docker Engine is installed and running.
- Run commands from the root of `cloud-native-devops-platform-app`.
- Inspect `backend/.env.example`, backend settings, and the Dockerfile for the current required variables (including `SECRET_KEY`, `POSTGRES_PASSWORD`, and `FIRST_SUPERUSER_PASSWORD`).
- Create a local secret-bearing environment file **outside Git**, with restrictive permissions. Never commit it or paste its contents into issue reports or documentation.

For a fresh rebuild, use the current app repository's documented configuration and the secured Docker Compose workflow where possible. The examples below illustrate the manual-container workflow but must be reconciled with the current backend's startup contract before running.

## PostgreSQL Container

Create an isolated Docker network:

```bash
docker network create cloud-platform
```

Provide a database password securely in the execution environment. Avoid literal passwords in shell history. One option is to prepare an untracked, permission-restricted environment file with the required PostgreSQL variables and pass it to the container:

```bash
# Example file contents (placeholders only; use real values securely):
# POSTGRES_DB=app
# POSTGRES_USER=app
# POSTGRES_PASSWORD=<secure-runtime-password>
chmod 600 /path/to/postgres-runtime.env

docker run -d \
  --name postgres \
  --network cloud-platform \
  --env-file /path/to/postgres-runtime.env \
  postgres:16
```

> For a disposable manual test this command does not configure a named PostgreSQL volume. Data may be lost when the container is removed. It is **not** the current EKS persistence design, which declares a PVC backed by EBS gp3.

## Backend Container

Build the backend image:

```bash
docker build -t cloud-platform-backend:local ./backend
```

Prepare a separate untracked `/path/to/backend-runtime.env` with the variables **actually required by the current backend**, including database settings, `SECRET_KEY`, and initial administrator settings as applicable. The backend currently assembles its database connection from PostgreSQL configuration inputs; do not assume the old standalone `DATABASE_URL` example remains valid.

```bash
chmod 600 /path/to/backend-runtime.env

docker run -d \
  --name backend \
  --network cloud-platform \
  -p 127.0.0.1:8000:8000 \
  --env-file /path/to/backend-runtime.env \
  cloud-platform-backend:local
```

The two runtime files must provide consistent database credentials and host configuration. The database host for a container on this Docker network is `postgres`.

## Validation Procedure

Inspect container state and backend logs (do not expose secret values):

```bash
docker ps --filter name=postgres --filter name=backend
docker logs --tail 100 backend
```

Check the API docs:

```bash
curl -I http://127.0.0.1:8000/docs
```

Use `curl -i http://127.0.0.1:8000/docs` if the application does not support `HEAD`. Confirm database connectivity and initialization through logs and an actual application request; an HTTP response alone does not prove database health.

Historically validated in the earlier test:

- Backend successfully connected to PostgreSQL and completed expected initialization/migration activity.
- API and Swagger documentation were accessible.
- Authentication succeeded with a configured test application account.

These are **historical outcomes**, not results of a new November 2026 execution.

## Cleanup (Disposable Manual Test Only)

```bash
docker rm -f backend postgres
docker network rm cloud-platform
```

Run cleanup only when these resources belong to this disposable test; do not delete shared containers or data. Delete local runtime environment files according to your security policy when no longer required.

## Relationship to Other Project Components

- **Docker Compose:** See [docker-compose-validation.md](docker-compose-validation.md) for the integrated three-service test.
- **Kubernetes:** The separate Kubernetes repository provides Traefik routing, Kubernetes Secret provisioning, and PostgreSQL EBS-backed PVC configuration.
- **Frontend API configuration:** The November 2026 source uses `OpenAPI.BASE = import.meta.env.VITE_API_URL || window.location.origin`; this frontend source change was build-tested, but no new end-to-end EKS/browser test is claimed here.

## Outcome

**Historical result:** Independent Docker containers established backend-to-database connectivity and application functionality before Compose orchestration was introduced. **Current status:** This document has been corrected for secure rebuild guidance; the updated manual commands have not been re-executed as part of the documentation change.
