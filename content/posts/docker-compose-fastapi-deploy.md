---
title: "Deploying a FastAPI App with Docker Compose: PostgreSQL + Redis"
date: "2026-07-20"
description: "Step-by-step guide to containerizing a FastAPI application with PostgreSQL and Redis using Docker Compose."
tags: ["docker", "fastapi", "devops", "postgresql"]
categories: ["DevOps"]
author: "Nahid Hasan"
---

One of the most common tasks I do as a DevOps engineer is containerizing applications. Here's the pattern I use for FastAPI apps that need PostgreSQL and Redis.

<!--more-->

## The Stack

- **FastAPI** (Python) — the web framework
- **PostgreSQL** — the primary database
- **Redis** — caching and task queues
- **Docker Compose** — orchestration

## Dockerfile

```dockerfile
FROM python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip wheel --no-cache-dir --no-deps -w /wheels -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /wheels /wheels
COPY . .

RUN pip install --no-cache /wheels/*

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## docker-compose.yml

```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s

volumes:
  pgdata:
```

## Key Takeaways

1. **Healthchecks matter** — `depends_on` with `condition: service_healthy` prevents the app from starting before the DB is ready
2. **Multi-stage builds** keep the image small (~150MB vs ~1GB)
3. **Never bake secrets** into the image — use `env_file` or Docker secrets
4. **Volume for Postgres** so data survives container restarts

This exact pattern is what I used for my [Portfolio Tracker](https://github.com/nahid-devops2021/portfolio-tracker) and [Money Manager](https://github.com/nahid-devops2021/money-management) projects. 🐳