# Journey App DevOps Project

A **production-style, end-to-end DevOps pipeline** for a full-stack web application, built and deployed on AWS using Docker, Kubernetes (EKS), Helm, and Jenkins CI/CD.

> **Live demo status:** The application was deployed and running on **AWS EKS** behind an AWS LoadBalancer. The environment has since been decommissioned (AWS account expired and the EKS cluster was deleted), so the public URL below is **no longer reachable**. The full infrastructure-as-code, Helm chart, and pipeline in this repository reproduce the exact deployment.
>
> ~~`http://a12dbdb2a78324118ba89471eb8275c4-e04e8f518750cc31.elb.eu-north-1.amazonaws.com/`~~ *(offline)*

---

## Architecture

```
                          GitHub push
                              │
                              ▼
                     ┌──────────────────┐
                     │  Jenkins Pipeline │
                     └──────────────────┘
                              │  build → push → deploy
                              ▼
            Docker Hub (obito991/backend, obito991/frontend)
                              │
                              ▼
                   ┌────────────────────────┐
                   │  Kubernetes (AWS EKS)   │
                   │                         │
   NGINX Ingress ──┤  /     → frontend (Nginx static)
                   │  /api  → backend (Node.js/Express)
                   │            │            │
                   │            ▼            │
                   │      PostgreSQL (in-cluster)
                   └────────────────────────┘
```

- **Frontend** — static site served by **Nginx** (`frontend/`).
- **Backend** — **Node.js / Express** API (`backend/server.js`) that connects to PostgreSQL via the `pg` pool using `DB_*` environment variables; the root route returns `SELECT NOW()` to prove DB connectivity.
- **Database** — **PostgreSQL**, with credentials injected as environment variables.
- **Routing** — NGINX Ingress Controller: `/` → frontend, `/api` → backend.

---

## Tech Stack

| Category          | Tools                                   |
| ----------------- | --------------------------------------- |
| Frontend          | Nginx (static HTML)                     |
| Backend           | Node.js, Express, `pg`                  |
| Database          | PostgreSQL                              |
| Containerization  | Docker, Docker Compose                  |
| Orchestration     | Kubernetes (AWS EKS)                    |
| Packaging/Release | Helm                                    |
| CI/CD             | Jenkins (declarative pipeline)          |
| Ingress           | NGINX Ingress Controller                |
| Cloud             | AWS (EC2 for Jenkins, EKS, LoadBalancer)|

---

## Repository Layout

```
frontend/            # Static site + Nginx config + Dockerfile
backend/             # Node.js/Express API + Dockerfile
helm/journey-app/    # Helm chart — the source of truth for deployment
k8s/manifests/       # Raw Kubernetes manifests (reference only)
docker-compose.yml   # Local development stack
.env.example         # Template for local environment variables
Jenkinsfile          # CI/CD pipeline definition
```

> **Note:** Deployments are driven by the **Helm chart** (`helm/journey-app/`). The raw manifests in `k8s/manifests/` are kept for reference and are not what the pipeline applies.

---

## Local Development

Configuration is read from a `.env` file (git-ignored) via Docker Compose variable substitution.

```bash
# 1. Create your local env file from the template
cp .env.example .env        # adjust values if desired

# 2. Build and start the full stack
docker-compose up --build
```

| Service  | URL                      |
| -------- | ------------------------ |
| Frontend | http://localhost:5050    |
| Backend  | http://localhost:3000    |
| Postgres | localhost:5432 (in-network) |

Environment variables (see `.env.example`): `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`.

---

## CI/CD Pipeline (Jenkins)

The `Jenkinsfile` defines a fully automated pipeline triggered by a GitHub webhook on every push:

1. **Checkout** the repository.
2. **Verify environment** — Docker, AWS identity, `kubectl` nodes, Helm, project structure.
3. **Build** backend and frontend Docker images (tagged with the Jenkins `BUILD_NUMBER` and `latest`).
4. **Login & push** images to Docker Hub (`obito991/backend`, `obito991/frontend`).
5. **Deploy with Helm** — `helm upgrade --install` with image repository/tag overrides.
6. **Verify rollout** — waits on `kubectl rollout status` for both deployments, then prints pods/services/ingress.

The result is a **zero-downtime rolling update** on the EKS cluster with no manual steps.

---

## Manual Deployment

Deploy the chart to any configured Kubernetes context:

```bash
helm upgrade --install journey-app ./helm/journey-app
```

Override images at deploy time (as the pipeline does):

```bash
helm upgrade --install journey-app ./helm/journey-app \
  --set backend.image.repository=obito991/backend  --set backend.image.tag=<tag> \
  --set frontend.image.repository=obito991/frontend --set frontend.image.tag=<tag>
```

---

## Secrets Management

Database credentials and backend environment variables are delivered through **Kubernetes Secrets** (`helm/journey-app/templates/secret.yaml`, values under `secrets:` in `values.yaml`). For local development the same values live in `.env`, which is git-ignored.

---

## Author

**Mohamed Newish** — DevOps Engineer | Cloud · Kubernetes · CI/CD
