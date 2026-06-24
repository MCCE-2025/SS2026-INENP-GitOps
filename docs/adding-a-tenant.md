# Adding a Tenant

Tenants are created through GitOps. Do not create tenant workloads manually with `kubectl` or direct Helm commands. Add a tenant folder under `tenants/`, commit it through the normal PR flow, and let Argo CD reconcile it.

## How tenant discovery works

`argocd/applications/tenants.yaml` is an Argo CD `ApplicationSet`. It scans this repository for directories matching:

```text
tenants/*
```

Every matching directory becomes one Argo CD application named:

```text
tenant-<folder-name>
```

The tenant application combines:

```text
tenants/<tenant>/*.yaml              # raw tenant manifests, excluding *-values.yaml
tenants/versions-frontend.yaml       # central frontend version
tenants/<tenant>/frontend-values.yaml
tenants/versions-backend.yaml        # central backend version
tenants/<tenant>/backend-values.yaml
```

## Tenant folder structure

Create a folder like this:

```text
tenants/<tenant-name>/
|-- namespace.yaml
|-- frontend-values.yaml
|-- backend-values.yaml
|-- avwx-secret.yaml
|-- external-secret.yaml
|-- database-schema.yaml
`-- network-policy.yaml
```

Use lowercase DNS-safe tenant names, for example:

```text
tenant-b
team-blue
staging
```

## Recommended process

1. Copy an existing tenant folder, usually `tenants/tenant-a/`.
2. Rename the folder to the new tenant name, for example `tenants/tenant-b/`.
3. Update `namespace.yaml`.
4. Update `frontend-values.yaml`.
5. Update `backend-values.yaml`.
6. Update `database-schema.yaml`.
7. Update namespaces in `avwx-secret.yaml`, `external-secret.yaml`, and `network-policy.yaml`.
8. Open a PR and let Argo CD reconcile after merge.

## Required changes

### 1. Namespace and quota

Update names, labels, and ResourceQuota object names:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-b
  labels:
    tenant: tenant-b
    environment: dev
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-b-quota
  namespace: tenant-b
```

Keep the sync-wave annotation so the namespace and quota exist before workloads are applied.

### 2. Frontend values

Set the public hostname and TLS secret:

```yaml
runtimeConfig:
  enabled: true
  backendApiUrl: /api

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    external-dns.alpha.kubernetes.io/hostname: tenant-b.inenp.werschlan.at
  hosts:
    - host: tenant-b.inenp.werschlan.at
      paths:
        - path: /
          pathType: Prefix
  extraPaths:
    - path: /api
      pathType: Prefix
      serviceName: weather-app-backend
      servicePort: 8080
  tls:
    - secretName: tenant-b-frontend-tls
      hosts:
        - tenant-b.inenp.werschlan.at
```

`backendApiUrl: /api` is important: the browser calls the same host, and ingress-nginx routes `/api` to the backend Service in the same namespace.

### 3. Backend values

Set the tenant database schema:

```yaml
database:
  enabled: true
  schema: tenant_b
  credentialsSecretName: cloud-sql-credentials
  instanceConnectionName: sonorous-stone-498307-u2:europe-west3:weather-app-db
```

Use underscores for PostgreSQL schema names when the tenant name contains dashes.

The backend image tag normally comes from `tenants/versions-backend.yaml`. Only set `image.tag` in the tenant file when that tenant intentionally needs an override.

### 4. Database schema marker

Update `database-schema.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-schema
data:
  schema: tenant_b
```

This documents the schema assigned to the tenant.

### 5. External secrets

Update the namespace in both ExternalSecret files:

```yaml
metadata:
  namespace: tenant-b
```

Required secrets:

| File | Target Secret | Remote Secret Manager key |
|---|---|---|
| `avwx-secret.yaml` | `api-keys` | `avwx-api-key` |
| `external-secret.yaml` | `cloud-sql-credentials` | `cloud-sql-app-password` |

Secrets are synced by External Secrets Operator from GCP Secret Manager. Secret values must never be committed to Git.

### 6. Network policy

Update every namespace reference in `network-policy.yaml` to the new tenant namespace:

```yaml
metadata:
  namespace: tenant-b
```

The standard tenant policy model is:

| Policy | Purpose |
|---|---|
| `default-deny-ingress` | Denies all ingress into the tenant namespace by default |
| `allow-same-namespace` | Allows frontend and backend pods in the same namespace to communicate |
| `allow-ingress-nginx` | Allows the shared ingress controller to reach tenant workloads |

## Versioning and promotion

Tenant applications inherit central versions:

```text
tenants/versions-frontend.yaml
tenants/versions-backend.yaml
```

To promote a new frontend release to all tenants:

```yaml
image:
  tag: "v1.8.0"
```

To promote a new backend release to all tenants:

```yaml
image:
  tag: "v1.10.0"
```

To test a release in one tenant first, set `image.tag` in that tenant's values file. `staging` already follows this pattern.

## Validation checklist

- Tenant folder name is lowercase and DNS-safe
- `namespace.yaml` namespace and ResourceQuota names match the tenant
- `frontend-values.yaml` host, TLS secret, and ExternalDNS hostname match the tenant
- `frontend-values.yaml` keeps `runtimeConfig.backendApiUrl: /api`
- `frontend-values.yaml` routes `/api` to `weather-app-backend` on port `8080`
- `backend-values.yaml` uses the correct database schema
- `database-schema.yaml` matches the backend schema
- ExternalSecret namespaces match the tenant
- NetworkPolicy namespaces match the tenant
- No secret values are committed
- PR is reviewed before merge

After merge, Argo CD discovers the new folder and creates the tenant application automatically.
