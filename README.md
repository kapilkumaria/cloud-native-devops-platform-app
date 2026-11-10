# Cloud-Native DevOps Platform — Application Repository

> **Project 1 | Application layer**  
> **Documentation updated:** November 10, 2026  
> **Status:** Docker Compose implementation previously validated; frontend production build validated following the November 9 API URL correction. The revised EKS deployment has **not** yet been tested on a running cluster.

## Overview

This repository contains the application layer of the **Cloud-Native DevOps Platform**:

- **Frontend:** React, TypeScript, Vite, served by Nginx in the container image.
- **Backend:** FastAPI with OpenAPI/Swagger documentation and application authentication.
- **Database:** PostgreSQL.
- **Containerization:** Dockerfiles for the frontend and backend.
- **Local / EC2 orchestration:** Docker Compose, including service health checks.
- **Kubernetes integration:** Application images and configuration consumed by the separate Kubernetes repository.

The application can be run as a Docker Compose stack, including on an EC2 instance provisioned and configured by the infrastructure repository. The associated Kubernetes repository implements a separate EKS deployment path with Traefik-based ingress and an EBS-backed PostgreSQL PersistentVolumeClaim.

**Scope:** This repository provides the application code and Docker Compose definitions; the infrastructure and Kubernetes platform are maintained separately. The presence of Kubernetes manifests in another repository does not mean the latest application image has already been published or tested on EKS.

## Architecture

![Application architecture](docs/diagrams/app-architecture.png)

Editable diagram: `docs/diagrams/app-architecture.drawio`

### Docker Compose / EC2 path

```text
Browser
  |                         Direct-access ports used in the documented Compose tests
  +---- http://SERVER_IP:3000 ----> Nginx / React frontend
  |
  +---- http://SERVER_IP:8000 ----> FastAPI backend ----> PostgreSQL
                                         |
                                         +---- /docs (Swagger UI)
```

The exact access URLs depend on the host, port mappings, firewall rules and frontend API URL embedded at build time. For browser access from another machine, `localhost` and Docker-only hostnames are **not** suitable API destinations.

### Intended EKS path (separate Kubernetes repository)

```text
Browser
   |
   v
Configured DNS hostname / AWS load balancer
   |
   v
Traefik ingress controller
   |
   +---- /api... ----> backend ClusterIP Service ----> FastAPI Pods
   |
   +---- /... --------> frontend ClusterIP Service ---> Nginx / React Pods
                                                        
PostgreSQL Pod ----> PersistentVolumeClaim ----> EBS gp3 volume
```

This is the **current source-defined design**, not a claim of a successful November EKS runtime deployment. The Kubernetes ingress configuration uses a placeholder hostname, `app.example.com`, which must be replaced with an owned/configured DNS name for a real deployment. TLS and end-to-end browser functionality must be validated separately.

## Technology Stack

| Layer | Technology | Responsibility |
| --- | --- | --- |
| Frontend | React + TypeScript + Vite | Web application and browser-side API client |
| Frontend web server | Nginx | Serves compiled static assets |
| Backend | FastAPI | HTTP API, authentication and application logic |
| Database | PostgreSQL | Relational data |
| Packaging | Docker | Frontend and backend container images |
| Local / EC2 orchestration | Docker Compose | Starts and connects application services |
| API documentation | FastAPI OpenAPI / Swagger | `/docs` on the backend |
| EKS routing (other repo) | Traefik | Host/path-based ingress to internal Services |
| EKS storage (other repos) | Kubernetes PVC + AWS EBS CSI | Persistent PostgreSQL storage configuration |

## Repository Structure

```text
cloud-native-devops-platform-app/
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   ├── package-lock.json
│   ├── src/
│   │   └── main.tsx
│   └── ...
├── backend/
│   ├── Dockerfile
│   ├── .env.example
│   └── ...
├── docs/
│   ├── manual-container-testing.md
│   ├── docker-compose-validation.md
│   ├── screenshots/
│   └── diagrams/
├── docker-compose.yaml
├── .env.example
├── README.md
└── .gitignore
```

The tree is abbreviated; inspect the repository for the complete source layout.

## Configuration and Secrets

The backend needs environment configuration for database connectivity, authentication and application initialization. Important sensitive values include:

| Variable | Purpose |
| --- | --- |
| `SECRET_KEY` | Backend authentication/JWT signing key |
| `POSTGRES_PASSWORD` | PostgreSQL application password |
| `FIRST_SUPERUSER_PASSWORD` | Initial administrator password |

Additional settings include the PostgreSQL host, database and username, and the initial administrator email. **Consult the actual backend settings, example environment files and deployment templates for the full required configuration.**

For local development, start from `backend/.env.example` where applicable. Do not publish populated `.env` files, passwords, signing keys, Terraform state, or private keys. Generate a strong signing key locally, for example:

```bash
python3 -c 'import secrets; print(secrets.token_urlsafe(48))'
```

Store the generated value securely; do not paste it into Git, screenshots or documentation. A stable signing key must be provided to all replicas of the same backend deployment and maintained across restarts when continuity of issued tokens is required.

For Kubernetes, the backend Deployment consumes a non-secret ConfigMap and a separately provisioned `backend-secret`; PostgreSQL consumes `postgres-secret`. **The secrets must exist before starting those workloads.** The Kubernetes repository documents the required keys in `docs/secure-secret-provisioning.md`. AWS Secrets Manager with External Secrets Operator is a recommended production integration, **not a claim that this application currently has that integration deployed**.

## Frontend API URL: Build-Time Configuration and Same-Origin Fallback

The frontend initializes its generated OpenAPI client in `frontend/src/main.tsx`:

```typescript
OpenAPI.BASE = import.meta.env.VITE_API_URL || window.location.origin
```

Behavior:

1. **When `VITE_API_URL` is defined during the Vite build**, that value becomes the API base URL in the compiled frontend bundle. This is useful when the browser must access the API at an explicit, separately reachable address (for example, the EC2/Compose test path).
2. **When `VITE_API_URL` is not defined at build time**, the browser uses `window.location.origin`. This is intended for EKS ingress routing where the frontend and API share the same hostname.
3. The FastAPI API routes already include `/api/v1` in their paths. Traefik routes `/api` traffic to the backend **without removing `/api`**. For example, a browser request to `/api/v1/login/access-token` should reach the backend under that same path.

**Important:** A `VITE_*` environment variable injected into an already-built Nginx container at runtime does **not** rewrite the static JavaScript bundle. For EKS, build the frontend image **without** an unwanted `VITE_API_URL` override if the desired behavior is same-origin routing. Confirm the build configuration and test network requests in the browser.

The previous Kubernetes frontend runtime ConfigMap was removed from the Kubernetes repository because it could not reliably set Vite's build-time URL. The currently referenced older `v1` frontend image must **not** be assumed to contain the newer source change; a fresh, uniquely tagged image must be built and published before runtime testing.

## Docker Compose: Build and Run

### Prerequisites

- Docker Engine with a working Docker daemon.
- Docker Compose plugin (`docker compose version`).
- Required backend environment values supplied securely according to the Compose file and backend configuration.
- Available network ports and access rules appropriate to the deployment host.

From the repository root:

```bash
docker compose config --services
# Review the resolved configuration without copying secrets into public logs.

docker compose up -d --build
docker compose ps
```

To check service health and diagnose a failure:

```bash
docker compose ps
docker compose logs --tail=100 backend
docker compose logs --tail=100 frontend
docker compose logs --tail=100 postgres
```

The Compose definitions and service names are the source of truth if they differ from the examples above. Docker Compose health checks help report component health and can support startup dependencies where configured; they do **not** guarantee application-level end-to-end readiness in every circumstance.

### Documented test endpoints

| Service | Endpoint | Notes |
| --- | --- | --- |
| Frontend | `http://SERVER_IP:3000` | Replace `SERVER_IP` with an actual reachable host address. |
| Backend API | `http://SERVER_IP:8000` | Direct backend access in the Compose/EC2 test configuration. |
| Swagger UI | `http://SERVER_IP:8000/docs` | Interactive FastAPI API documentation. |

These URLs reflect the **historical Compose/EC2 validation path**, not the intended EKS public ingress endpoint. Avoid exposing the backend or database to the public Internet unnecessarily in a production environment.

For a local-only test, use the port mappings in `docker-compose.yaml` and the corresponding local host address.

### Stop the stack

```bash
docker compose down
```

**Data safety:** Do not use `docker compose down -v` unless you have verified that deleting attached Compose volumes/data is intended. Docker Compose storage and Kubernetes EBS-backed PVC storage are separate deployment mechanisms.

## Container Health Checks and Validation

The project's documented Docker Compose validation includes:

| Component | Check | Interpretation |
| --- | --- | --- |
| PostgreSQL | `pg_isready` | Database process accepts readiness checks. |
| Backend | `/docs` endpoint | Web server responds; deeper API/database/authentication testing is still necessary. |
| Frontend | Nginx HTTP response | Static web server responds; browser/API requests must be tested separately. |

Previous manual validation covered frontend and backend container builds, PostgreSQL connectivity, frontend login/dashboard, and Swagger UI. Those observations are preserved in the files below and in their associated screenshots. They are **historical test evidence**, not proof that every later Git change has run successfully in EKS.

### Evidence and Screenshots

**Manual container validation:** `docs/manual-container-testing.md`  
**Docker Compose validation:** `docs/docker-compose-validation.md`

![Frontend login](docs/screenshots/frontend-login-page.png)

![Frontend dashboard](docs/screenshots/frontend-dashboard.png)

![Backend Swagger UI](docs/screenshots/backend-swagger-ui.png)

![Manual container test](docs/screenshots/docker-ps-manual-validation.png)

![Docker Compose health](docs/screenshots/docker-compose-healthy-containers.png)

## November 9, 2026 — Verified Frontend Source Change

The frontend API URL fallback was merged into `main` as commit **`6e2c3ae`**:

```typescript
OpenAPI.BASE = import.meta.env.VITE_API_URL || window.location.origin
```

The frontend's TypeScript/Vite production build succeeded using the `node:20-alpine` Docker image as a temporary build environment:

```bash
cd frontend

docker run --rm \
  --user "$(id -u):$(id -g)" \
  -e HOME=/tmp \
  -e npm_config_cache=/tmp/.npm \
  -v "$PWD":/app \
  -w /app \
  node:20-alpine \
  sh -c 'npm ci && npm run build'
```

Observed build evidence included successful TypeScript compilation, Vite bundling and `1258` transformed modules. This validates the **frontend build**, not public browser routing or updated container-image deployment.

### Dependency security finding (open)

The production dependency-tree audit executed with `npm audit --omit=dev` reported:

| Severity | Count |
| --- | ---: |
| Critical | 4 |
| High | 8 |
| Moderate | 8 |
| **Total** | **20** |

This report includes transitive dependencies and does not, by itself, prove which packages are present in the final Nginx image or exploitable in browser runtime. However, high and critical findings require triage and remediation before claiming production readiness.

To rerun the audit from the repository root without installing Node.js on the host:

```bash
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -e HOME=/tmp \
  -e npm_config_cache=/tmp/.npm \
  -v "$PWD/frontend":/app \
  -w /app \
  node:20-alpine \
  npm audit --omit=dev
```

Run `npm audit` or dependency analysis as a separate security change; **do not automatically run `npm audit fix --force`** without reviewing potentially breaking upgrades. Scan the final runtime container image as an additional control.

## Kubernetes / EKS Integration: Deployment Prerequisites

The associated Kubernetes source now declares:

- Frontend `ClusterIP` Service behind Traefik (not a separate frontend `LoadBalancer`).
- Traefik routes `/api` requests to the backend while preserving the existing API prefixes.
- PostgreSQL PVC using a gp3 StorageClass; EBS CSI add-on is declared in the infrastructure repository.
- Backend and PostgreSQL credentials supplied through externally created Kubernetes Secrets.
- No frontend runtime `VITE_API_URL` ConfigMap.

A future EKS rebuild should, at minimum:

1. Review and apply the infrastructure changes only after confirming AWS identity, region, supported EKS/add-on configuration, Terraform plan and costs.
2. Verify the EBS CSI controller/Pod Identity integration before requiring PostgreSQL PVC provisioning.
3. Create the required Kubernetes Secrets securely and verify their **key names**, without displaying values.
4. Build and publish a **new immutable frontend image** containing commit `6e2c3ae`, without an unintended build-time API URL override. Update the Kubernetes frontend Deployment to use that image.
5. Validate Kustomize output and apply resources in a safe dependency order; confirm StorageClass, PVC, Pods, Services and Traefik are healthy.
6. Replace the placeholder ingress hostname with a configured hostname, establish appropriate DNS and TLS, and test frontend requests and `/api/v1` from an actual browser.
7. Confirm database persistence across Pod recreation, including EBS volume lifecycle, backups and recovery expectations.
8. Investigate outstanding dependency vulnerabilities before describing the deployment as production-ready.

**Not yet validated after the November changes:** live EKS provisioning, EBS CSI add-on health, PVC `Bound` state, revised frontend Docker image deployment, Traefik/API browser behavior and PostgreSQL persistence. A successful Kustomize render or TypeScript build is not a substitute for those tests.

## Related Repositories

- [Infrastructure — Terraform / Ansible / EKS](https://github.com/kapilkumaria/cloud-native-devops-platform-infra)
- [Kubernetes — EKS / Traefik / Argo CD / observability](https://github.com/kapilkumaria/cloud-native-devops-platform-kubernetes)

## Additional Documentation

- [Manual container testing](docs/manual-container-testing.md)
- [Docker Compose validation](docs/docker-compose-validation.md)
- [Application architecture diagram](docs/diagrams/app-architecture.png)

## Future Work / Open Validation

- Remediate and re-audit frontend dependency findings; validate fixes with build and functional tests.
- Build and push the updated frontend container image, pin it to an immutable version/digest in Kubernetes.
- Rebuild EKS when needed and verify Traefik path routing, database persistence, Secrets and runtime health.
- Complete end-to-end browser tests, including login and API requests, against the configured EKS hostname.
- Introduce production-grade DNS/TLS, backup/restore, deployment controls and secure secret lifecycle as appropriate.

## License

Refer to the repository's actual license file or declared licensing terms. Do not infer a license from this README alone.
