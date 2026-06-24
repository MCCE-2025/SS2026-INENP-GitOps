# INENP GitOps Repository

This repository is the GitOps entry point for the INENP Weather App platform (WS2026). It contains Argo CD application definitions, platform component configuration, shared application values, tenant-specific configuration, and Crossplane-managed cloud resources.

## Documentation

- **Project overview, repository structure, access URLs:** [docs/overview.md](docs/overview.md)
- **Services and responsibilities:** [docs/services.md](docs/services.md)
- **Adding a new tenant:** [docs/adding-a-tenant.md](docs/adding-a-tenant.md)

Related repositories:

| Repository | Role |
|---|---|
| [SS2026-INENP-IaC](https://github.com/MCCE-2025/SS2026-INENP-IaC) | GKE, Argo CD bootstrap, IAM, Workload Identity, Secret Manager bootstrap |
| [SS2026-INENP-backend](https://github.com/MCCE-2025/SS2026-INENP-backend) | Spring Boot backend source and Helm chart |
| [SS2026-INENP-frontend](https://github.com/MCCE-2025/SS2026-INENP-frontend) | Quasar/Vue frontend source and Helm chart |

## Access URLs

Everything is served over HTTPS through the shared `ingress-nginx` entrypoint. DNS records are created by ExternalDNS and TLS certificates by cert-manager (`letsencrypt-prod`). All hosts live under `inenp.werschlan.at`.

| Service | URL | Notes |
|---|---|---|
| Argo CD | https://argocd.inenp.werschlan.at | user `admin`; password from `argocd-initial-admin-secret` |
| Kargo | https://kargo.inenp.werschlan.at | user `admin`; password from `terraform output -raw kargo_admin_password` in the IaC repo |
| Shared frontend | https://weather.inenp.werschlan.at | shared frontend deployment |
| Tenant `tenant-a` | https://tenant-a.inenp.werschlan.at | routes `/api` to the tenant backend |
| Tenant `tenant-z` | https://tenant-z.inenp.werschlan.at | routes `/api` to the tenant backend |
| Staging | https://staging.inenp.werschlan.at | can pin its own frontend/backend versions |
| Shared ingress | https://ingress.inenp.werschlan.at | DNS name for the ingress-nginx LoadBalancer |

## Quick Operations

Promote all tenant frontends:

```yaml
# tenants/versions-frontend.yaml
image:
  tag: "vX.Y.Z"
```

Promote all tenant backends:

```yaml
# tenants/versions-backend.yaml
image:
  tag: "vX.Y.Z"
```

Create a new tenant by adding a folder under `tenants/<tenant-name>/` with namespace, secrets, network policy, database schema, frontend values, and backend values. See [docs/adding-a-tenant.md](docs/adding-a-tenant.md).

## AI-assisted development

We used AI tools (**Cursor** and **ChatGPT**) as support throughout the project - not as a replacement for review and ownership. They helped draft:

- GitHub issues
- Pull request descriptions
- Configuration files (Argo CD, Helm values, Crossplane resources)
- Documentation (including this repo's `docs/`)

All AI-generated content was reviewed, adapted, and validated by the team before merge.
