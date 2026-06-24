# GitOps - Project Overview

GitOps repository for the INENP Weather App platform (WS2026). It defines what Argo CD reconciles into the Kubernetes cluster: platform components, shared application deployments, tenant deployments, and cloud resources managed through Crossplane.

**Platform entry point:** this repository.

## Scope & Status

| Area | Details |
|---|---|
| **Scope** | Argo CD Applications, ApplicationSet-based tenants, platform component configuration, shared app values, tenant app values, Crossplane resources |
| **Stack** | Argo CD, Helm, Crossplane, External Secrets Operator, cert-manager, ExternalDNS, ingress-nginx, Kyverno, Kargo, Dynatrace, OpenBao |
| **Status** | Active - Terraform bootstraps the base platform, then Argo CD continuously reconciles this repository |

## Repository Structure

```text
.
|-- README.md
|-- docs/
|   |-- overview.md
|   |-- services.md
|   `-- adding-a-tenant.md
|-- argocd/
|   `-- applications/
|       |-- backend.yaml
|       |-- cert-manager.yaml
|       |-- cert-manager-config.yaml
|       |-- crossplane.yaml
|       |-- crossplane-resources.yaml
|       |-- dynatrace-operator.yaml
|       |-- dynatrace-config.yaml
|       |-- external-dns.yaml
|       |-- frontend.yaml
|       |-- ingress-nginx.yaml
|       |-- kargo.yaml
|       |-- kyverno.yaml
|       |-- kyverno-policies.yaml
|       |-- openbao.yaml
|       |-- platform-ingress.yaml
|       `-- tenants.yaml
|-- platform/
|   |-- cert-manager/
|   |-- crossplane/
|   |-- dynatrace/
|   |-- external-dns/
|   |-- ingress/
|   `-- kyverno-policies/
|-- apps/
|   |-- backend/
|   `-- frontend/
`-- tenants/
    |-- versions-backend.yaml
    |-- versions-frontend.yaml
    |-- staging/
    |-- tenant-a/
    `-- tenant-z/
```

## Directory Responsibilities

| Directory | Purpose |
|---|---|
| `argocd/applications/` | Argo CD `Application` and `ApplicationSet` definitions. These files define what Argo CD deploys and from which repository/chart/path. |
| `platform/` | Platform-level manifests and Helm values. These support the cluster and app runtime rather than the weather app itself. |
| `apps/` | Shared application values for frontend and backend deployments outside the tenant ApplicationSet. |
| `tenants/` | Tenant-specific manifests and Helm values. Each folder under `tenants/` becomes one tenant deployment through the `tenants` ApplicationSet. |
| `docs/` | Human-facing documentation for repository structure, services, and tenant operations. |

## Repositories

| Repository | Role | Relevant configuration |
|---|---|---|
| **[SS2026-INENP-GitOps](https://github.com/MCCE-2025/SS2026-INENP-GitOps)** *(this repo)* | GitOps / platform entry point | Argo CD Applications, tenant folders, shared values, Crossplane resources |
| **[SS2026-INENP-IaC](https://github.com/MCCE-2025/SS2026-INENP-IaC)** | Infrastructure bootstrap | GKE, Argo CD bootstrap, Workload Identity, IAM, Secret Manager, root app |
| **[SS2026-INENP-frontend](https://github.com/MCCE-2025/SS2026-INENP-frontend)** | Frontend source and chart | Quasar SPA, Helm chart `charts/weather-app-frontend/`, release image in GAR |
| **[SS2026-INENP-backend](https://github.com/MCCE-2025/SS2026-INENP-backend)** | Backend source and chart | Spring Boot API, Helm chart `charts/weather-app-backend/`, release image in GAR |

## Deployment Model

Terraform is responsible for the bootstrap layer. After bootstrap, Argo CD owns continuous reconciliation.

```text
Terraform
  -> creates GKE, IAM, Workload Identity, Secret Manager basics
  -> installs/bootstraps Argo CD
  -> points Argo CD at this GitOps repository

Argo CD
  -> reconciles argocd/applications/*
  -> installs platform services
  -> installs shared apps
  -> discovers tenants from tenants/*
  -> applies Crossplane-managed cloud resources
```

## Application Deployment

| Deployment | Argo CD file | Source | Destination |
|---|---|---|---|
| Shared frontend | `argocd/applications/frontend.yaml` | Frontend Helm chart + `apps/frontend/values.yaml` | namespace `frontend` |
| Shared backend | `argocd/applications/backend.yaml` | Backend Helm chart + `apps/backend/values.yaml` | namespace `backend` |
| Tenant apps | `argocd/applications/tenants.yaml` | Tenant folder + frontend chart + backend chart | namespace named after each tenant folder |

Tenant frontend and backend versions are managed independently:

| File | Purpose |
|---|---|
| `tenants/versions-frontend.yaml` | Central frontend image tag for all tenants unless overridden by a tenant |
| `tenants/versions-backend.yaml` | Central backend image tag for all tenants unless overridden by a tenant |
| `tenants/<tenant>/frontend-values.yaml` | Tenant frontend host, resources, runtime config, optional image override |
| `tenants/<tenant>/backend-values.yaml` | Tenant backend resources, schema, secrets, optional image override |

## Frontend and Backend Communication

Tenant frontends use a relative backend URL:

```yaml
runtimeConfig:
  enabled: true
  backendApiUrl: /api
```

The frontend Ingress routes `/api` to the backend Service in the same tenant namespace:

```yaml
extraPaths:
  - path: /api
    pathType: Prefix
    serviceName: weather-app-backend
    servicePort: 8080
```

Traffic path:

```text
Browser -> https://<tenant>.inenp.werschlan.at/api -> frontend Ingress -> weather-app-backend:8080
```

## Access URLs

| Instance | URL | Backend routing |
|---|---|---|
| Shared frontend | https://weather.inenp.werschlan.at | frontend only in shared namespace |
| Tenant `tenant-a` | https://tenant-a.inenp.werschlan.at | `/api` -> backend in namespace `tenant-a` |
| Tenant `tenant-z` | https://tenant-z.inenp.werschlan.at | `/api` -> backend in namespace `tenant-z` |
| Staging | https://staging.inenp.werschlan.at | `/api` -> backend in namespace `staging` |

Management UIs:

| Service | URL |
|---|---|
| Argo CD | https://argocd.inenp.werschlan.at |
| Kargo | https://kargo.inenp.werschlan.at |
| Shared ingress | https://ingress.inenp.werschlan.at |

## Cloud Resources

Crossplane resources under `platform/crossplane/` manage cloud dependencies:

| Resource | File | Purpose |
|---|---|---|
| Backend GAR repository | `artifact-registry-backend.yaml` | Docker image repository for backend images |
| Frontend GAR repository | `artifact-registry-frontend.yaml` | Docker image repository for frontend images |
| Cloud SQL instance | `cloud-sql-instance.yaml` | PostgreSQL instance for backend persistence |
| Cloud SQL database/user | `cloud-sql-database.yaml`, `cloud-sql-user.yaml` | Database and application user |
| Cloud SQL Secret sync | `cloud-sql-external-secret.yaml` | Secret material for backend database credentials |
| Frontend GCS hosting | `frontend-hosting*.yaml` | GCS bucket for standalone SPA hosting |
| Provider configs | `provider-config*.yaml`, `provider-gcp-*.yaml` | GCP provider configuration and providers |
| Tenant schema convention | `schema-multitenancy.yaml` | Documents/defines schema-based tenant isolation convention |

## AI-assisted development

We used AI tools (**Cursor** and **ChatGPT**) as support throughout the project - not as a replacement for review and ownership. They helped draft:

- GitHub issues
- Pull request descriptions
- Argo CD manifests
- Helm values
- Crossplane resources
- Tenant configuration
- Documentation (including this repo's `docs/`)

All AI-generated content was reviewed, adapted, and validated by the team before merge.
