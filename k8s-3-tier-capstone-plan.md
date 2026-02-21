# Kubernetes 3-Tier Capstone Plan (Focused Track)

> Audience: SWE transitioning toward production infrastructure roles  
> Scope: One portfolio-ready 3-tier app (frontend + backend + PostgreSQL) on Kubernetes  
> Timebox: 2-3 weeks at 5-8 hours/week  
> Local runtime: minikube

---

## Recommended App Idea: ShipLog

If you do not already have an app, use **ShipLog**:

- A lightweight internal tool to track service deploy events.
- Frontend: view and create deploy records.
- Backend: REST API for CRUD.
- Database: PostgreSQL for persistent records.

### What Deployments It Tracks

- It tracks deployments for **your own services/projects** (not a fixed external app catalog).
- One repo can map to multiple services, so `service_name` remains a required field.
- A GitHub repo URL is useful metadata, but not enough by itself to define a deployment event.

Why this is a good fit:

- Small enough to finish quickly.
- Realistic infra narrative for interviews ("I built and operated a deploy-tracking service").
- Supports clear read/write flows for testing service discovery, probes, scaling, and persistence.

### Minimal Feature Set (keep it tight)

- Create a deploy record: `service_name`, `environment`, `version`, `status`, `deployed_at`.
- List deploy records with newest first.
- Filter by environment (`dev`, `staging`, `prod`).
- Health endpoint (`/health`) and readiness endpoint (`/ready`) in backend.

### Ingestion Modes

- V1 (required): manual entry from frontend form and direct backend API calls.
- V2 (stretch): GitHub Actions posts deploy metadata to backend after successful deploy.

### Data Model (initial)

- Table: `deployments`
  - `id` (PK)
  - `service_name` (text)
  - `environment` (text)
  - `version` (text)
  - `status` (text) -- e.g. `success`, `failed`, `rollback`
  - `deployed_at` (timestamp)
  - `repo_url` (text, nullable)
  - `commit_sha` (text, nullable)
  - `workflow_run_url` (text, nullable)
  - `source` (text default `manual`) -- `manual` or `github_actions`
  - `created_at` (timestamp default now)

### API Contract (V1)

- `POST /api/deployments` creates an event.
- `GET /api/deployments` lists events (supports `?environment=` filter).
- `GET /health` liveness check.
- `GET /ready` readiness check (must validate DB connectivity).

---

## Capstone Outcomes

By the end, you should be able to demonstrate:

1. A running 3-tier app on Kubernetes (FE + BE + Postgres).
2. Correct use of Deployments, Services, ConfigMaps, Secrets, PVCs, and Ingress.
3. Operational readiness basics: probes, resource requests/limits, rollout verification, and troubleshooting workflow.
4. A short "production-style" runbook and demo script.

---

## Implementation Plan (No Code Yet)

### Phase 0: Scope and Repo Setup (0.5 day)

- Decide repo layout (`frontend/`, `backend/`, `k8s/`).
- Write app contract (routes + payloads + DB schema).
- Define non-goals to prevent scope creep:
  - No auth in v1
  - No async workers
  - No multi-service backend
  - No GitHub OAuth or webhook signature verification in v1

Exit criteria:

- One-page scope and endpoint list complete, including manual-only ingestion rules.

### Phase 1: App + Containers (2-3 days)

- Build minimal TypeScript Node backend with Postgres access.
- Build minimal frontend UI (form + table list).
- Add Dockerfiles for FE/BE.
- Validate app locally with Docker Compose (optional but recommended).

Exit criteria:

- CRUD flow works locally.
- FE and BE images build successfully.

### Phase 2: Core Kubernetes Deploy (2-3 days)

- Namespace: `shiplog`.
- PostgreSQL:
  - Secret for credentials
  - PVC for persistence
  - Deployment + ClusterIP Service
- Backend:
  - ConfigMap for non-secret config
  - Secret env for DB credentials
  - Deployment + ClusterIP Service
  - Liveness/readiness probes
- Frontend:
  - Deployment + Service
  - Route API calls to backend service DNS

Exit criteria:

- FE can create/read deploy records through backend.
- Data persists after pod restart.

### Phase 3: Networking + Hardening (2-3 days)

- Add Ingress for clean local URL routing.
- Add resource requests/limits for all workloads.
- Add NetworkPolicy basics:
  - FE -> BE allowed
  - BE -> Postgres allowed
  - FE -> Postgres denied
- Add rollout strategy checks and rollback practice.

Exit criteria:

- Controlled traffic paths are verified.
- Rolling restart works with zero app breakage in normal path.

### Phase 4: Production-Role Showcase Layer (1-2 days)

- Add Helm chart OR Kustomize overlays (`dev`, `staging`).
- Add CI flow (lint/test/build images + deploy to local/staging cluster).
- V2 stretch: add GitHub Actions step to call ShipLog API and create deployment events.
- Add basic metrics endpoint and at least one dashboard/alert rule.
- Write runbook:
  - "Service down"
  - "DB not reachable"
  - "Rollback last backend deploy"

Exit criteria:

- You can perform a 5-8 minute demo with failure drill + recovery.

---

## Kubernetes Artifacts Checklist

- `k8s/namespace.yaml`
- `k8s/database/{secret,pvc,deployment,service}.yaml`
- `k8s/backend/{configmap,deployment,service}.yaml`
- `k8s/frontend/{deployment,service}.yaml`
- `k8s/ingress.yaml`
- `k8s/networkpolicy.yaml`
- `k8s/README.md` (apply order + verification commands)

---

## Verification Scenarios (Must Pass)

1. Fresh deploy: all pods become Ready.
2. API health: `/health` returns OK.
3. API readiness: `/ready` fails when DB is unavailable.
4. Persistence: delete Postgres pod; data remains.
5. Networking controls: FE cannot connect directly to Postgres.
6. Rollout: deploy new backend image and verify no request failures during rollout window.
7. Deployment events include service/env/version and can optionally include repo/commit metadata.

---

## Portfolio Deliverables

- Architecture diagram (simple and readable).
- Short technical write-up:
  - Why these Kubernetes primitives.
  - What failed during setup and how you debugged it.
  - What you would change for cloud production.
- Demo script + command list.

---

## Decision: Why Separate From the Main Plan

Keep the broad 8-week learning plan as the "curriculum."
Use this capstone as the "execution track" with strict deliverables.

This separation avoids mixing:

- Theory progression (broad topics) with
- Project delivery milestones (showcase artifact)

---

## Optional Alternate App Ideas (if ShipLog does not appeal)

1. **Incident Notes Board**: log incidents and mitigation notes.
2. **Feature Flag Admin Lite**: CRUD flags by environment.
3. **Release Checklist Tracker**: pre/post deploy checklist status.

Use the same 3-tier architecture and Kubernetes plan regardless of domain.
