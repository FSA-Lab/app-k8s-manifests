# app-k8s-manifests

Kubernetes manifests for the microservice app, managed with Kustomize and deployed via ArgoCD.

This repo is the **CD (Continuous Delivery)** side of a two-repo GitOps workflow:
- **app-k8s** — source code, Dockerfiles, Jenkins CI pipeline
- **this repo** — K8s manifests, Kustomize overlays, ArgoCD applications

---

## Repo Structure

```
app-k8s-manifests/
├── k8s/
│   ├── base/                          # Kustomize base layer
│   │   ├── kustomization.yaml         # Root kustomization
│   │   ├── auth-service/              # App services
│   │   ├── inventory-service/
│   │   ├── order-service/
│   │   ├── payment-service/
│   │   ├── auth-db/                   # Databases (Postgres)
│   │   ├── inventory-db/
│   │   ├── order-db/
│   │   ├── payment-db/
│   │   ├── rabbitmq/                  # Message broker
│   │   ├── kong/                      # API gateway
│   │   ├── otel-collector/            # Tracing collector
│   │   ├── jaeger/                    # Tracing UI
│   │   ├── prometheus/                # Metrics
│   │   ├── grafana/                   # Dashboards
│   │   └── migrations/               # DB migration jobs
│   │
│   └── overlays/
│       ├── staging/
│       │   ├── kustomization.yaml     # staging namespace + image tags
│       │   ├── kong-patch.yaml        # Kong URLs → staging
│       │   ├── prometheus-patch.yaml  # Prometheus scrape targets → staging
│       │   └── payment-service-patch.yaml  # AUTH_SERVICE_URL → staging
│       └── prod/
│           ├── kustomization.yaml     # prod namespace + image tags
│           ├── kong-patch.yaml        # Kong URLs → prod
│           ├── prometheus-patch.yaml  # Prometheus scrape targets → prod
│           └── payment-service-patch.yaml  # AUTH_SERVICE_URL → prod
│
└── argocd/
    ├── staging-app.yaml               # ArgoCD app → auto-sync to staging
    └── prod-app.yaml                  # ArgoCD app → manual sync to prod
```

---

## Prerequisites

- Azure AKS cluster (or any K8s cluster)
- `kubectl` configured to access the cluster
- `kustomize` CLI (or `kubectl kustomize`)
- ArgoCD installed in the cluster
- DockerHub account for container images

---

## Configuration

### 1. Replace Placeholders

Before deploying, replace these placeholders in the overlay files:

**`k8s/overlays/staging/kustomization.yaml`** and **`k8s/overlays/prod/kustomization.yaml`**:

| Placeholder | Replace With | Example |
|---|---|---|
| `yourdockerhub` | Your DockerHub username | `johndoe` |

The ArgoCD Application manifests (`argocd/*.yaml`) already point to `https://github.com/FSA-Lab/app-k8s-manifests.git`. Update if the repo moves.

### 2. Create Kubernetes Secrets

Secrets are created manually with `kubectl` — they are NOT committed to Git.

**Infrastructure secrets (in `infra` namespace):**

```bash
kubectl create namespace infra

# Postgres credentials (shared across all DBs)
kubectl create secret generic postgres-secrets \
  --from-literal=POSTGRES_USER='root' \
  --from-literal=POSTGRES_PASSWORD='your-secure-password' \
  -n infra

# Grafana admin password
kubectl create secret generic grafana-secrets \
  --from-literal=GF_SECURITY_ADMIN_PASSWORD='admin' \
  -n infra
```

**App service secrets (in `staging` namespace):**

```bash
kubectl create namespace staging

# Auth service
kubectl create secret generic auth-service-secrets \
  --from-literal=DATABASE_URL='postgres://root:your-secure-password@auth-db.infra.svc.cluster.local:5432/auth' \
  --from-literal=RABBITMQ_URL='amqp://rabbitmq.infra.svc.cluster.local:5672' \
  --from-literal=JWT_SECRET='your-jwt-secret' \
  -n staging

# Inventory service
kubectl create secret generic inventory-service-secrets \
  --from-literal=DATABASE_URL='postgres://root:your-secure-password@inventory-db.infra.svc.cluster.local:5432/inventory' \
  --from-literal=RABBITMQ_URL='amqp://rabbitmq.infra.svc.cluster.local:5672' \
  --from-literal=JWT_SECRET='your-jwt-secret' \
  -n staging

# Order service
kubectl create secret generic order-service-secrets \
  --from-literal=DATABASE_URL='postgres://root:your-secure-password@order-db.infra.svc.cluster.local:5432/order' \
  --from-literal=RABBITMQ_URL='amqp://rabbitmq.infra.svc.cluster.local:5672' \
  --from-literal=JWT_SECRET='your-jwt-secret' \
  -n staging

# Payment service
kubectl create secret generic payment-service-secrets \
  --from-literal=DATABASE_URL='postgres://root:your-secure-password@payment-db.infra.svc.cluster.local:5432/payment' \
  --from-literal=RABBITMQ_URL='amqp://rabbitmq.infra.svc.cluster.local:5672' \
  --from-literal=JWT_SECRET='your-jwt-secret' \
  -n staging
```

**Repeat for `prod` namespace** with different passwords.

### 3. Configure ArgoCD Access

```bash
# Add this repo to ArgoCD
argocd repo add https://github.com/FSA-Lab/app-k8s-manifests.git \
  --username <github-username> \
  --password <github-pat>
```

Or apply the ArgoCD Application manifests — ArgoCD will prompt for credentials on first sync.

---

## How It Works

### Jenkins Updates Image Tags

When code is pushed to the app repo, Jenkins:
1. Builds Docker images tagged with the git short SHA (e.g., `johndoe/auth-service:abc1234`)
2. Pushes images to DockerHub
3. Clones this repo, runs `kustomize edit set image` to update the overlay
4. Commits and pushes the updated image tags

```bash
# What Jenkins runs (in the overlay directory):
kustomize edit set image auth-service=johndoe/auth-service:abc1234
kustomize edit set image inventory-service=johndoe/inventory-service:abc1234
kustomize edit set image order-service=johndoe/order-service:abc1234
kustomize edit set image payment-service=johndoe/payment-service:abc1234
```

### ArgoCD Syncs to K8s

- **Staging (`develop` branch):** ArgoCD auto-syncs when it detects changes — new image tags trigger a rolling update
- **Prod (`main` branch):** ArgoCD requires manual sync approval from the ArgoCD UI or CLI

### Manual Image Tag Update

To manually update an image tag (without Jenkins):

```bash
cd k8s/overlays/staging
kustomize edit set image auth-service=yourdockerhub/auth-service:v1.2.3
git add . && git commit -m "update auth-service to v1.2.3" && git push
```

---

## Validation

```bash
# Verify Kustomize renders valid YAML
kustomize build k8s/base
kustomize build k8s/overlays/staging
kustomize build k8s/overlays/prod

# Check cluster resources
kubectl get pods -n infra
kubectl get pods -n staging
kubectl get pods -n prod
kubectl get svc -n infra    # Kong external IP

# Check ArgoCD sync status
argocd app get app-staging
argocd app get app-prod
```

---

## Cluster Layout

```
┌─────────────────────────────────────────────────────────┐
│  Azure AKS Cluster                                       │
│                                                          │
│  infra namespace (shared)                                │
│  ┌─────────────────────────────────────────────────┐    │
│  │ auth-db, inventory-db, order-db, payment-db      │    │
│  │ rabbitmq, kong (LoadBalancer)                    │    │
│  │ otel-collector, jaeger, prometheus, grafana      │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  staging namespace              prod namespace            │
│  ┌───────────────────┐      ┌───────────────────┐      │
│  │ auth-service       │      │ auth-service      │      │
│  │ inventory-service  │      │ inventory-service │      │
│  │ order-service      │      │ order-service     │      │
│  │ payment-service    │      │ payment-service   │      │
│  │ migration jobs (4) │      │ migration jobs (4)│      │
│  └───────────────────┘      └───────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

### Networking

| Component | Service Type | Port | Notes |
|---|---|---|---|
| Kong | LoadBalancer | 8000 | External access via Azure LB |
| App services | ClusterIP | 3000 | Internal, Kong routes to them |
| Databases | ClusterIP | 5432 | Internal only |
| RabbitMQ | ClusterIP | 5672 | Internal only |
| OTel Collector | ClusterIP | 4318 | Internal only |
| Jaeger | NodePort | 16686 | Debug UI |
| Prometheus | NodePort | 9090 | Debug UI |
| Grafana | NodePort | 3004 | Dashboard UI |
