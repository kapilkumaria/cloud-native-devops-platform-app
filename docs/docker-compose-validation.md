# Docker Compose Validation

## Objective

Record the historical validation of the complete Cloud-Native DevOps Platform application stack using Docker Compose, while providing safe guidance for a future re-run.

## Scope and Status

The tested Compose deployment comprised:

- **Frontend:** React/Vite application served by Nginx.
- **Backend:** FastAPI application.
- **Database:** PostgreSQL.

**Historical evidence:** All three services reported healthy; the frontend login and dashboard, backend Swagger UI, authentication, and backend/database integration were tested in the earlier environment.

**November 2026 source update:** The frontend's API base URL now uses `import.meta.env.VITE_API_URL || window.location.origin`. This change passed a Docker-based frontend build, but the earlier Compose screenshots do **not** establish that a new Compose stack or EKS deployment has been tested with the updated image.

## Prerequisites

Run from the root of `cloud-native-devops-platform-app`.

- Docker Engine and Compose plugin are operational.
- Required application secrets are provided through appropriately secured local runtime configuration; see `backend/.env.example` and the current backend settings.
- Do not assume the repository-root `.env.example` (which may be empty) supplies valid runtime credentials.
- Supply compatible backend/database settings, a secure `SECRET_KEY`, and initial administrator credentials before starting.
- Review the actual Compose configuration for host port mappings and environment-file handling before exposing ports beyond localhost.

> **Vite note:** Variables beginning `VITE_` are incorporated into the frontend at **build time**. Setting `VITE_API_URL` only on a running Nginx container does not rebuild the browser bundle. For different-origin deployments configure the build accordingly. For same-origin ingress routing, the fallback uses the browser origin when no explicit build-time URL is provided.

## Deployment Procedure

Review the resolved Compose configuration. It may contain environment-specific values, so do **not** publish or paste unredacted output:

```bash
docker compose config --quiet
```

Build and start the application stack:

```bash
docker compose up -d --build
```

Verify service state:

```bash
docker compose ps
```

If services do not become healthy, inspect the affected service using:

```bash
docker compose logs --tail 100 <service-name>
```

Replace `<service-name>` with the actual service name from `docker compose ps`.

## Validation Procedure

### Container Health

Confirm the frontend, backend, and PostgreSQL services reach their expected running/healthy states. Health checks are useful indicators but do not replace application-function tests.

### Frontend Access

For the historical EC2/Compose configuration, the frontend was accessible at:

```text
http://<SERVER_IP>:3000
```

For a local workstation test with the same port mapping, use `http://localhost:3000`.

### Backend Swagger UI

For the historical EC2/Compose configuration, Swagger UI was accessible at:

```text
http://<SERVER_IP>:8000/docs
```

For a local test with the same port mapping, use `http://localhost:8000/docs`.

### Functional Checks

- Open the frontend login page.
- Authenticate using an authorized test account; do not record its password.
- Confirm the dashboard loads and performs an operation requiring the backend API.
- Confirm the backend can connect to PostgreSQL.
- Inspect browser network requests and backend logs if authentication or API calls fail.

If the browser is running on a different host than the frontend server, remember that `localhost` in the browser refers to the **browser's own host**, not the EC2 instance. Match the browser API URL to the deployed routing model.

## Historical Validation Results

The earlier Docker Compose test documented:

- All services reported a healthy state.
- The frontend login page was accessible.
- A seeded application account authenticated successfully.
- Swagger UI was accessible.
- Database-backed application functionality was verified.

These statements refer to the original validation session and are not claims of a fresh test after the November source changes.

## Screenshots — Historical Evidence

### Healthy Containers

![Healthy Containers](screenshots/docker-compose-healthy-containers.png)

### Frontend Dashboard

![Frontend Dashboard](screenshots/docker-compose-dashboard.png)

### Swagger UI

![Swagger UI](screenshots/docker-compose-swagger-ui.png)

The screenshots document the historical Compose runtime and should not be interpreted as proof of the revised EKS networking/persistence configuration.

## Troubleshooting

| Symptom | First checks |
| --- | --- |
| PostgreSQL unhealthy | Database logs, runtime password/configuration, startup timing, available storage |
| Backend startup failure | Backend logs, Secret/environment inputs, PostgreSQL hostname and port |
| Frontend loads but API fails | Browser network requests, build-time `VITE_API_URL`, CORS, API routing |
| Authentication fails | Backend/database initialization, test user configuration, relevant API response |
| Compose build fails | Docker daemon, Dockerfile/build context, dependency and registry availability |

The current frontend dependency audit also reported **20 production dependency-tree advisories (8 moderate, 8 high, 4 critical)** in the November review. This is a remediation item before production promotion, not evidence that the earlier Compose functional tests failed. An npm audit result does not by itself establish which vulnerable code is present or reachable in the deployed browser bundle.

## Cleanup

For disposable environments only:

```bash
docker compose down
```

This stops and removes Compose-managed containers/networks while retaining named volumes by default. **Do not use `docker compose down -v` unless database data loss is intended and approved.**

## Outcome

**Historical result:** Docker Compose successfully orchestrated the React/Nginx frontend, FastAPI backend, and PostgreSQL database, and the tested application passed container-health and functional checks.

**Current evidence boundary:** Updated frontend source passed a build, but the modified source, newly built image, revised Kubernetes ingress, and EBS-backed database have not been validated together in a newly deployed EKS cluster.
