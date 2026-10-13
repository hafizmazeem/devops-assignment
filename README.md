# DevOps Assignment: Containerization, CI/CD & Orchestration

![CI](https://github.com/hafizmazeem/devops-assignment/actions/workflows/ci.yml/badge.svg)
![Docker](https://img.shields.io/badge/Docker-multi--stage-2496ED?logo=docker&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-3.0.3-000000?logo=flask&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-7-DC382D?logo=redis&logoColor=white)

**Author:** Hafiz Muhammad Azeem ([@hafizmazeem](https://github.com/hafizmazeem))
**Internship:** Progree, DevOps

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Task 2: Application Containerization & Asset Optimization](#task-2-application-containerization--asset-optimization)
- [Task 3: CI/CD Pipeline](#task-3-cicd-pipeline)
- [Task 4: Infrastructure as Code & Orchestration](#task-4-infrastructure-as-code--orchestration)

## Overview
This repository contains three connected DevOps tasks built around one small web application.
The image built in Task 2 is tested and built automatically by the pipeline in Task 3, and deployed on Kubernetes in Task 4.

## Architecture

```
            Client (curl / browser)
                      |
                 host :8000
                      |
        +-------------v--------------+
        |  web container (Flask)     |
        |  Gunicorn on :8000         |
        |  non-root user: appuser    |
        +-------------+--------------+
                      |  internal Compose network only
        +-------------v--------------+
        |  redis container (:6379)   |
        |  not published to the host |
        +----------------------------+
```

## Project Structure

```
devops-assignment/
├── app/
│   ├── app.py
│   ├── test_app.py
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── docker-compose.yml
│   └── .env.example
├── .github/workflows/ci.yml
├── terraform/
├── k8s/
└── README.md
```

---

## Task 2: Application Containerization & Asset Optimization

### Objective
Build a modular, reliable container environment for a multi-dependency web application using a multi-stage Dockerfile, a minimal final image, secure environment variable mapping and functional port routing.

### The Application
A Flask app that counts page visits in Redis. It has two dependencies, Flask (served by Gunicorn) and Redis, so it only works when both containers run together.

| Endpoint | Purpose |
|---|---|
| `GET /` | Increments and returns the visit counter stored in Redis |
| `GET /health` | Returns `ok`, used for health checks |

### How Each Requirement Is Met

| Requirement | Implementation |
|---|---|
| Multi-stage Dockerfile | The builder stage installs the Python packages. The final stage starts from a clean base and copies only the installed packages with `COPY --from=builder`, so pip caches and build leftovers are discarded. |
| Minimal image footprint | `python:3.12-slim` base, `pip install --no-cache-dir`, a `.dockerignore`, and `requirements.txt` copied before the source code so the dependency layer is cached. Result: **191 MB on disk, 46.1 MB compressed**. |
| Secure environment variables | The Dockerfile holds only non-secret defaults. Real values are loaded at run time from `.env` through `env_file`. `.env` is excluded by `.gitignore` and `.dockerignore`, and only `.env.example` is committed. |
| Least privilege | The container runs as the non-root user `appuser`. |
| Functional port routing | `8000:8000` maps the host port to Gunicorn. Redis publishes no port and is reachable only on the internal Compose network. |

### Dockerfile Design

```dockerfile
# Stage 1: builder, installs dependencies
FROM python:3.12-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Stage 2: final, small runtime image
FROM python:3.12-slim
WORKDIR /app
RUN useradd -r -u 1001 appuser
COPY --from=builder /install /usr/local
COPY app.py .
USER appuser
ENV APP_NAME=myapp REDIS_HOST=redis
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

### How to Run

```bash
git clone https://github.com/hafizmazeem/devops-assignment.git
cd devops-assignment/app
cp .env.example .env
docker build -t myapp:1.0 .
docker compose up -d
curl localhost:8000
curl localhost:8000/health
```

To stop and clean up: `docker compose down`

### Result
The multi-dependency stack builds and runs from a single command. The final image is small, runs as a non-root user, keeps its configuration out of the image and out of Git, and exposes only the one port it needs.

---
