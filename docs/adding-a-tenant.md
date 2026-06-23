# Neuen Tenant anlegen

Diese Anleitung beschreibt, wie ein neuer Tenant im GitOps-Repository `SS2026-INENP-GitOps` angelegt wird.

## Überblick

Tenants werden **deklarativ über Git** bereitgestellt. Es gibt kein Skript und keine manuelle Argo-CD-Application pro Tenant.

Stattdessen entdeckt das Argo-CD-**ApplicationSet** (`argocd/applications/tenants.yaml`) automatisch jeden Unterordner unter `tenants/` und erzeugt daraus eine eigene Argo-CD-Application `tenant-<ordnername>`.

Pro Tenant werden bereitgestellt:

- Kubernetes-Namespace und ResourceQuota
- ExternalSecrets (Datenbank-Passwort, AVWX-API-Key)
- Frontend (Helm Chart aus `SS2026-INENP-frontend`)
- Backend (Helm Chart aus `SS2026-INENP-backend`)

Alle Tenants teilen sich eine Cloud-SQL-Instanz. Die Isolation erfolgt über **eigenes PostgreSQL-Schema pro Tenant** (Soft Multitenancy).

```text
tenants/
├── versions-frontend.yaml      # zentrale Frontend-Version (alle Tenants)
├── versions-backend.yaml       # zentrale Backend-Version (alle Tenants)
├── tenant-a/                   # Beispiel-Tenant
├── tenant-z/                   # Beispiel-Tenant
└── staging/                    # Canary-Tenant (Sonderfall)
```

## Voraussetzungen

Bevor ein neuer Tenant angelegt wird, müssen folgende Plattform-Komponenten laufen:

| Komponente | Zweck |
|---|---|
| Argo CD (Root App) | Synchronisiert `argocd/applications/` |
| Crossplane + Cloud SQL | Gemeinsame PostgreSQL-Instanz `weather-app-db` |
| External Secrets Operator | Sync von GCP Secret Manager → Kubernetes |
| ingress-nginx | Ingress Controller |
| cert-manager + ClusterIssuer | TLS-Zertifikate (`letsencrypt-prod`) |
| ExternalDNS | DNS-Einträge unter `*.inenp.werschlan.at` |

In GCP Secret Manager müssen folgende Secrets existieren:

| GCP Secret | Verwendung |
|---|---|
| `cloud-sql-app-password` | PostgreSQL-Passwort (IaC-verwaltet) |
| `avwx-api-key` | AVWX-API-Token für das Backend |

> **Hinweis:** Tenants teilen sich dasselbe DB-Passwort und denselben AVWX-Key. Die Mandantentrennung erfolgt ausschließlich über das PostgreSQL-Schema.

## Namenskonventionen

| Konzept | Regel | Beispiel (`tenant-b`) |
|---|---|---|
| Ordnername | Kleinbuchstaben, Bindestriche | `tenant-b` |
| Kubernetes-Namespace | Identisch zum Ordnernamen | `tenant-b` |
| Argo-CD-Application | `tenant-<ordnername>` | `tenant-tenant-b` |
| DNS-Hostname | `<ordnername>.inenp.werschlan.at` | `tenant-b.inenp.werschlan.at` |
| TLS-Secret | `<ordnername>-frontend-tls` | `tenant-b-frontend-tls` |
| ResourceQuota | `<ordnername>-quota` | `tenant-b-quota` |
| PostgreSQL-Schema | `tenant_<ordnername>` (Bindestriche → Unterstriche) | `tenant_b` |

### Schema-Naming

Standard-Tenants mit Präfix `tenant-` erhalten das Schema `tenant_<name>`:

| Ordner | Schema |
|---|---|
| `tenant-a` | `tenant_a` |
| `tenant-z` | `tenant_z` |
| `tenant-b` | `tenant_b` |

Sonder-Tenants (z. B. `staging`) verwenden den Ordnernamen direkt als Schema:

| Ordner | Schema |
|---|---|
| `staging` | `staging` |

Das Schema wird **nicht** von Crossplane angelegt. Flyway erstellt es beim ersten Start des Backends automatisch (`spring.flyway.create-schemas=true`).

## Schritt-für-Schritt-Anleitung

### 1. Tenant-Namen festlegen

Wähle einen eindeutigen Ordnernamen, z. B. `tenant-b`.

Leite daraus ab:

- Namespace: `tenant-b`
- Hostname: `tenant-b.inenp.werschlan.at`
- DB-Schema: `tenant_b`

### 2. Tenant-Verzeichnis anlegen

Erstelle den Ordner `tenants/tenant-b/` mit **sechs Dateien**. Am einfachsten kopierst du `tenants/tenant-a/` als Vorlage:

```bash
cp -r tenants/tenant-a tenants/tenant-b
```

### 3. Dateien anpassen

Ersetze in allen Dateien den Namen `tenant-a` durch `tenant-b` und `tenant_a` durch `tenant_b`.

#### `namespace.yaml`

Enthält Namespace und ResourceQuota (Sync-Wave `-1`, damit Quota vor Workloads existiert).

Anzupassen:

- `metadata.name` (Namespace)
- Labels `tenant:` und `environment:`
- ResourceQuota `metadata.name` und `metadata.namespace`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-b
  labels:
    tenant: tenant-b
    environment: dev
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-b-quota
  namespace: tenant-b
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
```

#### `frontend-values.yaml`

Helm-Values für das Frontend. Image-Tag kommt standardmäßig aus `tenants/versions-frontend.yaml`.

Anzupassen:

- Ingress-Hostname (Annotations, `hosts`, `tls`)
- TLS-Secret-Name
- Optional: `replicaCount`, `resources`

```yaml
ingress:
  annotations:
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

Wichtige Felder, die **nicht** geändert werden müssen (Plattform-Standard):

- `runtimeConfig.backendApiUrl: /api` — Frontend spricht Backend über denselben Host an
- `ingress.extraPaths` — leitet `/api` an den Tenant-Backend-Service weiter
- Image-Repository (GAR)

> **ResourceQuota:** Jeder Pod muss `resources.requests` und `resources.limits` definieren, sonst lehnt Kubernetes die Erstellung ab.

#### `backend-values.yaml`

Helm-Values für das Backend. Image-Tag kommt standardmäßig aus `tenants/versions-backend.yaml`.

Anzupassen:

- `database.schema` → `tenant_b`
- Optional: `replicaCount`, `resources`

```yaml
database:
  enabled: true
  schema: tenant_b
  credentialsSecretName: cloud-sql-credentials
  instanceConnectionName: sonorous-stone-498307-u2:europe-west3:weather-app-db
```

Folgende Werte bleiben für alle Standard-Tenants gleich:

- `serviceAccount.annotations` (GCP Workload Identity)
- `apiKeys` (Verweis auf ESO-Secret `api-keys`)
- `ingress.enabled: false` (Backend wird über Frontend-Ingress unter `/api` erreicht)

#### `external-secret.yaml`

Synchronisiert das Cloud-SQL-Passwort aus GCP Secret Manager.

Anzupassen:

- `metadata.namespace`

```yaml
metadata:
  name: cloud-sql-credentials
  namespace: tenant-b
```

#### `avwx-secret.yaml`

Synchronisiert den AVWX-API-Key aus GCP Secret Manager.

Anzupassen:

- `metadata.namespace`

```yaml
metadata:
  name: api-keys
  namespace: tenant-b
```

#### `database-schema.yaml`

Dokumentations-ConfigMap für das PostgreSQL-Schema.

Anzupassen:

- `data.schema`

```yaml
data:
  schema: tenant_b
```

### 4. Optional: Schema-Konvention aktualisieren

In `platform/crossplane/schema-multitenancy.yaml` kann ein Eintrag ergänzt werden:

```yaml
tenant-b-schema: tenant_b
```

Das ist **reine Dokumentation** — das Schema wird trotzdem von Flyway beim Backend-Start angelegt.

### 5. Änderungen committen und pushen

Nach dem Merge in den von Argo CD getrackten Branch (`HEAD`) passiert automatisch:

1. ApplicationSet entdeckt `tenants/tenant-b/`
2. Argo CD erstellt Application `tenant-tenant-b`
3. Sync-Wave `-1`: Namespace, Quota, ExternalSecrets
4. Sync-Wave `20`: Frontend- und Backend-Helm-Releases
5. ExternalDNS legt DNS-Eintrag an
6. cert-manager stellt TLS-Zertifikat aus
7. Flyway legt PostgreSQL-Schema `tenant_b` beim ersten Backend-Start an

> **Es ist keine neue Argo-CD-Application nötig.** Das ApplicationSet übernimmt die Provisionierung.

## Checkliste vor dem Merge

- [ ] Ordnername ist eindeutig und folgt der Konvention (`tenant-<name>`)
- [ ] Alle sechs Dateien sind vorhanden
- [ ] Namespace, Quota, Secrets und Ingress verweisen auf denselben Tenant-Namen
- [ ] Hostname `<name>.inenp.werschlan.at` ist noch nicht vergeben
- [ ] `database.schema` entspricht der Schema-Konvention (`tenant_b` für Ordner `tenant-b`)
- [ ] Frontend und Backend definieren `resources.requests` und `resources.limits`
- [ ] Summe aller Pod-Resources liegt innerhalb der ResourceQuota (`2 CPU / 4Gi` requests, `4 CPU / 8Gi` limits)

## Image-Versionen

Zentrale Versionen für alle Tenants:

| Datei | Steuert |
|---|---|
| `tenants/versions-frontend.yaml` | Frontend-Image-Tag aller Tenants |
| `tenants/versions-backend.yaml` | Backend-Image-Tag aller Tenants |

Ein Tenant kann die zentrale Version überschreiben, indem er in `frontend-values.yaml` bzw. `backend-values.yaml` ein eigenes `image.tag` setzt. Das ApplicationSet merged die Values in dieser Reihenfolge (spätere Datei gewinnt):

1. `tenants/versions-*.yaml`
2. `tenants/<name>/*-values.yaml`

Der `staging`-Tenant nutzt dieses Muster als Canary-Umgebung zum Testen neuer Images vor dem zentralen Rollout.

## Verifikation nach dem Deployment

### Argo CD

- Neue Application `tenant-<name>` erscheint und ist `Synced` / `Healthy`
- Alle drei Quellen (Manifests, Frontend-Chart, Backend-Chart) sind synchron

### Kubernetes

```bash
kubectl get ns tenant-b
kubectl get resourcequota -n tenant-b
kubectl get externalsecret -n tenant-b
kubectl get pods -n tenant-b
```

### End-to-End

- Frontend erreichbar: `https://tenant-b.inenp.werschlan.at`
- API erreichbar: `https://tenant-b.inenp.werschlan.at/api/...`
- TLS-Zertifikat gültig (cert-manager)
- Backend-Pod läuft und Schema `tenant_b` existiert in Cloud SQL

## Was nicht geändert werden muss

| Ressource | Grund |
|---|---|
| `argocd/applications/tenants.yaml` | ApplicationSet entdeckt neue Ordner automatisch |
| Crossplane Cloud-SQL-Ressourcen | Eine Instanz für alle Tenants |
| Helm Charts | Kommen aus Frontend-/Backend-Repo |
| GCP Secrets | Werden pro Tenant wiederverwendet (Schema-Isolation) |

## Troubleshooting

### Pod wird nicht erstellt — ResourceQuota

**Symptom:** `exceeded quota` oder fehlende Requests/Limits.

**Lösung:** In `frontend-values.yaml` und `backend-values.yaml` müssen `resources.requests` und `resources.limits` gesetzt sein. Die Summe aller Pods im Namespace darf die Quota nicht überschreiten.

### ExternalSecret bleibt leer

**Symptom:** Secret `cloud-sql-credentials` oder `api-keys` existiert nicht.

**Prüfen:**

- ESO läuft und `ClusterSecretStore` `gcp-secret-manager` ist bereit
- GCP Secrets `cloud-sql-app-password` und `avwx-api-key` existieren
- Workload Identity / IAM-Berechtigungen für ESO sind korrekt

### TLS / DNS funktioniert nicht

**Symptom:** Zertifikat fehlt oder Hostname nicht erreichbar.

**Prüfen:**

- Ingress-Annotation `external-dns.alpha.kubernetes.io/hostname` stimmt mit `hosts` überein
- cert-manager ClusterIssuer `letsencrypt-prod` ist aktiv
- ExternalDNS filtert auf `inenp.werschlan.at`

### Backend startet nicht — Datenbank

**Symptom:** Backend-CrashLoop, Flyway-Fehler.

**Prüfen:**

- `database.schema` in `backend-values.yaml` ist korrekt
- Secret `cloud-sql-credentials` enthält Passwort
- `instanceConnectionName` stimmt mit Cloud-SQL-Instanz überein

## Referenz: ApplicationSet

Das ApplicationSet in `argocd/applications/tenants.yaml` definiert das Verhalten:

```yaml
generators:
  - git:
      directories:
        - path: tenants/*
```

Pro entdecktem Ordner werden deployed:

1. Raw Manifests aus `tenants/<name>/` (ohne `*-values.yaml`)
2. Frontend Helm Chart + zentrale + tenant-spezifische Values
3. Backend Helm Chart + zentrale + tenant-spezifische Values

## Beispiel-Tenants

| Tenant | Typ | Besonderheit |
|---|---|---|
| `tenant-a` | Standard | Vorlage für neue Tenants |
| `tenant-z` | Standard | Identische Struktur wie `tenant-a` |
| `staging` | Canary | Eigene `image.tag`, Schema = `staging`, `environment: staging` |

## Verwandte Repositories

| Repository | Rolle |
|---|---|
| `SS2026-INENP-GitOps` | Tenant-Konfiguration, ApplicationSet |
| `SS2026-INENP-IaC` | Cluster-Bootstrap, Argo-CD-Root-App, Cloud-SQL-Secrets |
| `SS2026-INENP-frontend` | Frontend Helm Chart |
| `SS2026-INENP-backend` | Backend Helm Chart, Flyway-Schema-Erstellung |
