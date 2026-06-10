# GitOps Repository Structure

This repository contains GitOps configuration for platform components, application workloads, tenant-specific settings, and Argo CD application definitions.

The structure separates Argo CD control manifests from the actual configuration that is deployed.

## Repository layout

```text
.
├── argocd/
│   └── applications/
│       └── crossplane.yaml
│
├── platform/
│   ├── crossplane/
│   │   └── values.yaml
│   ├── external-dns/
│   │   └── values.yaml
│   └── cert-manager/
│       └── values.yaml
│
├── apps/
│   ├── backend/
│   │   └── values.yaml
│   └── frontend/
│       └── values.yaml
│
└── tenants/
    ├── tenant-a.yaml
    └── tenant-b.yaml
```

## Directory responsibilities

### `argocd/`

The `argocd/` directory contains Argo CD control manifests.

These files tell Argo CD what should be deployed, from where, and into which Kubernetes namespace or cluster.

Current structure:

```text
argocd/
└── applications/
    └── crossplane.yaml
```

### `argocd/applications/`

This directory contains Argo CD `Application` manifests.

Each file defines one deployable unit that Argo CD should manage.

Example:

```text
argocd/applications/crossplane.yaml
```

The `crossplane.yaml` file defines the Argo CD application responsible for deploying Crossplane.

It does not contain the Crossplane runtime logic itself. It only tells Argo CD where the Crossplane Helm chart or configuration is located and where it should be deployed.

Example responsibility:

```text
argocd/applications/crossplane.yaml
= Argo CD Application definition for Crossplane
```

### `platform/`

The `platform/` directory contains configuration for platform-level components.

Platform components are tools and services required to run or operate the Kubernetes platform.

Examples:

- Crossplane
- ExternalDNS
- cert-manager
- ingress-nginx
- Kyverno
- monitoring
- logging

Example:

```text
platform/crossplane/
```

This directory can contain Helm values, Kustomize configuration, wrapper charts, or Kubernetes manifests related to Crossplane.

Example responsibility:

```text
platform/crossplane/values.yaml
= Crossplane-specific Helm values or configuration
```

### `apps/`

The `apps/` directory contains configuration for business applications.

Examples:

```text
apps/backend/
apps/frontend/
```

Each application directory can contain Helm values, Kustomize configuration, or Kubernetes manifests for that application.

Example:

```text
apps/backend/
└── values.yaml
```

The backend and frontend application definitions should stay separate from platform components. This keeps platform tooling and product workloads clearly separated.

### `tenants/`

The `tenants/` directory contains tenant-specific configuration.

A tenant file describes one soft tenant and can include values such as namespace, environment, enabled applications, image tags, replica counts, ingress hostnames, and resource limits.

Example:

```yaml
tenant: tenant-a
namespace: tenant-a
environment: dev

apps:
  backend:
    enabled: true
    imageTag: "1.0.0"
    replicas: 1
  frontend:
    enabled: true
    imageTag: "1.0.0"
    replicas: 1

ingress:
  host: tenant-a.example.com

resources:
  quota:
    cpuRequests: "2"
    memoryRequests: "4Gi"
    cpuLimits: "4"
    memoryLimits: "8Gi"
```

This allows the same application configuration to be reused across multiple tenants with different values.

## Bootstrap approach

Terraform is responsible for bootstrapping the initial platform setup.

Terraform may create:

- the Kubernetes cluster
- the Argo CD namespace
- the Argo CD Helm release
- repository credentials
- initial Argo CD application manifests
- required cloud resources and IAM bindings

After the bootstrap phase, Argo CD can reconcile the desired state from this Git repository.

This means:

```text
Terraform = infrastructure and bootstrap layer
Argo CD   = GitOps reconciliation layer
```

## Crossplane deployment flow

The Crossplane deployment is split into two parts:

```text
argocd/applications/crossplane.yaml
```

This file defines the Argo CD `Application` for Crossplane.

```text
platform/crossplane/
```

This directory contains Crossplane-specific configuration, such as Helm values or additional manifests.

This separation keeps Argo CD wiring separate from platform component configuration.

## Design principles

- Keep Argo CD application definitions in `argocd/applications/`.
- Keep platform components in `platform/`.
- Keep business applications in `apps/`.
- Keep tenant-specific configuration in `tenants/`.
- Avoid hardcoded project IDs, credentials, or environment-specific values.
- Use variables, values files, or tenant configuration files for environment-specific settings.
- Let Terraform handle infrastructure and bootstrap.
- Let Argo CD handle continuous reconciliation after bootstrap.

## Notes

This repository structure is intentionally simple and can be extended later.

For now, only the Crossplane Argo CD application is included under `argocd/applications/`.

Additional Argo CD application manifests can be added later, for example:

```text
argocd/applications/external-dns.yaml
argocd/applications/cert-manager.yaml
argocd/applications/backend.yaml
argocd/applications/frontend.yaml
```
