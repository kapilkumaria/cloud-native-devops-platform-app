# Manual Container Testing

## Objective

Validate that the backend API can successfully connect to PostgreSQL before introducing Docker Compose.

## PostgreSQL Container

Create a dedicated Docker network:

```bash
docker network create cloud-platform
```

Run PostgreSQL container:

```bash
docker run -d \
  --name postgres \
  --network cloud-platform \
  -e POSTGRES_DB=app \
  -e POSTGRES_USER=app \
  -e POSTGRES_PASSWORD=password \
  postgres:16
```

## Backend Container

Build image:

```bash
docker build -t cloud-platform-backend:local ./backend
```

Run backend container:

```bash
docker run -d \
  --name backend \
  --network cloud-platform \
  -p 8000:8000 \
  -e DATABASE_URL=postgresql://app:password@postgres:5432/app \
  cloud-platform-backend:local
```

## Validation Performed

### API Health

```bash
curl http://localhost:8000
```

### Swagger Documentation

```text
http://localhost:8000/docs
```

### Database Connectivity

Verified backend startup logs showed successful database connection and migrations.

### Authentication

Successfully authenticated using seeded application user account.

## Outcome

The backend container successfully connected to PostgreSQL and served API requests. This validated the application before introducing Docker Compose orchestration.
