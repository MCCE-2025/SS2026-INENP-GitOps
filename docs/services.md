# Services

This document defines the services managed through the GitOps repository and explains what each one is responsible for.

## Application Services

| Service | Argo CD file | Namespace | Purpose |
|---|---|---|---|
| Frontend | `argocd/applications/frontend.yaml` | `frontend` | Shared Quasar/Vue SPA deployment using the frontend Helm chart and `apps/frontend/values.yaml`. |
| Backend | `argocd/applications/backend.yaml` | `backend` | Shared Spring Boot API deployment using the backend Helm chart and `apps/backend/values.yaml`. |
| Tenant frontend | `argocd/applications/tenants.yaml` | one namespace per tenant | Per-tenant SPA deployment, public hostname, TLS, and `/api` routing to the tenant backend. |
| Tenant backend | `argocd/applications/tenants.yaml` | one namespace per tenant | Per-tenant API deployment, AVWX Secret reference, Cloud SQL schema selection, and resources. |

## Platform Services

| Service | Argo CD file | Namespace | Purpose |
|---|---|---|---|
| Argo CD Applications | `argocd/applications/*.yaml` | `argocd` | Declarative control plane that tells Argo CD what to deploy. Argo CD itself is bootstrapped by the IaC repo. |
| ingress-nginx | `argocd/applications/ingress-nginx.yaml` | `ingress-nginx` | Shared Kubernetes ingress controller and LoadBalancer entrypoint. All public app and management hosts route through it. |
| cert-manager | `argocd/applications/cert-manager.yaml` | `cert-manager` | Installs cert-manager and CRDs. Uses Workload Identity for DNS-01 certificate validation. |
| cert-manager config | `argocd/applications/cert-manager-config.yaml` | `cert-manager` | Applies the `letsencrypt-prod` ClusterIssuer after cert-manager and ingress-nginx are available. |
| ExternalDNS | `argocd/applications/external-dns.yaml` | `external-dns` | Watches ingress/service annotations and creates DNS records under `inenp.werschlan.at`. |
| Crossplane | `argocd/applications/crossplane.yaml` | `crossplane-system` | Installs Crossplane. |
| Crossplane resources | `argocd/applications/crossplane-resources.yaml` | `crossplane-system` | Applies Crossplane providers, provider configs, GAR repositories, Cloud SQL resources, and frontend hosting resources. |
| Kyverno | `argocd/applications/kyverno.yaml` | `kyverno` | Policy engine for Kubernetes resource checks. |
| Kyverno policies | `argocd/applications/kyverno-policies.yaml` | `kyverno` | Cluster policies for image tags, privileged pods, probes, and resource requests/limits. Policies are configured in audit mode. |
| Kargo | `argocd/applications/kargo.yaml` | `kargo` | Promotion tool installed with a public UI/API at `kargo.inenp.werschlan.at`. Secrets come from GCP Secret Manager via ESO. |
| Dynatrace Operator | `argocd/applications/dynatrace-operator.yaml` | `dynatrace` | Installs the Dynatrace Operator. Tokens are not stored in Git. |
| Dynatrace config | `argocd/applications/dynatrace-config.yaml` | `dynatrace` | Applies the DynaKube custom resource after the operator and CRDs are ready. |
| OpenBao | `argocd/applications/openbao.yaml` | `openbao` | Open-source Vault fork for secret management experiments/additive platform capability. Uses GCP KMS auto-unseal. |
| Platform ingress | `argocd/applications/platform-ingress.yaml` | `argocd` | Applies stable HTTPS ingress resources for platform UIs such as Argo CD. |
| DNS test | `argocd/applications/dns-test.yaml` | `default` | Small ExternalDNS validation manifest for testing DNS automation. |

## Cloud Services Managed by Crossplane

| Service | File | Purpose |
|---|---|---|
| Google Artifact Registry - backend | `platform/crossplane/artifact-registry-backend.yaml` | Docker image repository `weather-app-backend`. Backend release workflow pushes images here. |
| Google Artifact Registry - frontend | `platform/crossplane/artifact-registry-frontend.yaml` | Docker image repository `weather-app-frontend`. Frontend release workflow pushes images here. |
| Cloud SQL PostgreSQL | `platform/crossplane/cloud-sql-instance.yaml` | PostgreSQL 15 instance `weather-app-db` in `europe-west3`, with HA, backups, PITR, and deletion protection. |
| Cloud SQL database | `platform/crossplane/cloud-sql-database.yaml` | Application database used by backend deployments. |
| Cloud SQL user | `platform/crossplane/cloud-sql-user.yaml` | Application database user used by backend deployments. |
| Cloud SQL external secret | `platform/crossplane/cloud-sql-external-secret.yaml` | Exposes database credentials through the secret flow used by workloads. |
| Frontend hosting bucket | `platform/crossplane/frontend-hosting.yaml` | GCS bucket for standalone frontend SPA hosting. |
| Crossplane providers | `platform/crossplane/provider-gcp-*.yaml` | GCP providers for Artifact Registry, SQL, and Storage. |
| Provider configs | `platform/crossplane/provider-config*.yaml` | Provider configuration for Crossplane GCP access. |

## Tenant Services

Each tenant folder under `tenants/` is discovered by `argocd/applications/tenants.yaml`. A tenant normally contains:

| File | Purpose |
|---|---|
| `namespace.yaml` | Namespace, labels, and ResourceQuota |
| `frontend-values.yaml` | Tenant frontend host, TLS, resources, `/api` routing, optional image override |
| `backend-values.yaml` | Tenant backend resources, AVWX Secret reference, Cloud SQL schema, optional image override |
| `avwx-secret.yaml` | ExternalSecret that syncs the AVWX token from GCP Secret Manager |
| `external-secret.yaml` | ExternalSecret that syncs Cloud SQL credentials from GCP Secret Manager |
| `database-schema.yaml` | Documents the tenant's PostgreSQL schema |
| `network-policy.yaml` | Default-deny ingress plus same-namespace and ingress-nginx allow rules |

Tenant frontend and backend are deployed into the same namespace. The frontend Ingress routes `/api` to `weather-app-backend:8080`, so each tenant uses its own backend.

## Image Promotion

| Service | Central version file | Image repository |
|---|---|---|
| Frontend tenants | `tenants/versions-frontend.yaml` | `europe-west3-docker.pkg.dev/sonorous-stone-498307-u2/weather-app-frontend/weather-app-frontend` |
| Backend tenants | `tenants/versions-backend.yaml` | `europe-west3-docker.pkg.dev/sonorous-stone-498307-u2/weather-app-backend/weather-app-backend` |

Staging can pin versions independently in:

```text
tenants/staging/frontend-values.yaml
tenants/staging/backend-values.yaml
```

This allows testing a release in staging before promoting the same image tag centrally for all tenants.
