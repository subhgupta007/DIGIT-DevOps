# DIGIT TL Platform — Complete Architecture, Infrastructure & Workflow Reference

## 1. Repository Structure

```
DIGIT-DevOps/
├── .github/workflows/
│   ├── digit_install.yml        # Full install: Terraform infra + DIGIT deploy (push to release-githubactions)
│   ├── digit_deploy_only.yml    # Deploy only: skips Terraform, uses existing EKS (push to release-githubactions OR deploy branch)
│   └── build.yml                # Build Docker images for tl-services, tl-calculator, digit-ui, gateway
│
├── build/
│   └── build-config.yml         # Maps pipeline names to Docker image build configs (used by build.yml)
│
├── deploy-as-code/
│   ├── digit-helmfile.yaml      # ROOT helmfile — orchestrates all sub-helmfiles
│   └── charts/
│       ├── environments/
│       │   ├── env.yaml         # Environment-specific values (DB, Kafka, domain, service config)
│       │   └── env-secrets.yaml # Secrets (DB password, AWS keys, git SSH key, etc.)
│       ├── backbone-services/
│       │   ├── backboneservices-helmfile.yaml  # Deploys: kafka, elasticsearch, redis, minio, ingress-nginx
│       │   └── [kafka-kraft, elasticsearch, redis, minio, ingress-nginx, cert-manager, ...]
│       ├── core-services/
│       │   ├── coreservices-helmfile.yaml      # Deploys: all DIGIT core + TL services
│       │   └── [tl-services, tl-calculator, egov-workflow-v2, egov-persister, egov-idgen, ...]
│       └── auxiliary-services/
│           ├── auxiliary-helmfile.yaml         # Optional: oauth2-proxy, pgadmin, kafka-connect
│           └── [oauth2-proxy, pgadmin, kafka-connect, ...]
│
└── infra-as-code/terraform/sample-aws/
    ├── input.yaml               # Cluster name, DB name, domain — source of truth for infra params
    ├── main.tf                  # EKS cluster, RDS, VPC, node groups, Karpenter
    ├── variables.tf             # Terraform variables (patched by init.go from input.yaml)
    ├── outputs.tf               # Outputs: db_instance_endpoint, cluster_endpoint, etc.
    └── scripts/
        ├── init.go              # Patches variables.tf + env.yaml with values from input.yaml
        └── envYAMLUpdater.go    # Patches env.yaml with terraform outputs (RDS endpoint, etc.)
```

---

## 2. Infrastructure (AWS — Already Provisioned)

### Cluster
| Resource | Value |
|---|---|
| EKS Cluster Name | `digit-tl-eks` |
| Region | `ap-south-1` |
| Kubernetes Version | 1.34 |
| Node Type | SPOT `r5ad.large` |
| Node Count | min:1, desired:3, max:5 |
| Node Disk | 100GB gp3 |

### Database
| Resource | Value |
|---|---|
| Type | AWS RDS PostgreSQL 16.6 |
| Instance | `db.t4g.medium` |
| Endpoint | `digit-tl-eks-db.cdoy2wumu54z.ap-south-1.rds.amazonaws.com:5432` |
| DB Name | `tldb` |
| Username | `tldbuser` |
| Password | `tldbuser1234` (in env-secrets.yaml) |
| Storage | 20GB gp3 |

### Networking
| Resource | Value |
|---|---|
| VPC CIDR | `192.168.0.0/16` |
| Availability Zones | `ap-south-1b`, `ap-south-1a` |
| Terraform State Bucket | `tl-digit-bucket` (S3, `ap-south-1`) |
| Domain | `tl.pgrdigit.in` |

### In-Cluster Backbone Services (namespace: `backbone`)
| Service | Purpose | Status |
|---|---|---|
| kafka-kraft | Message broker (Kafka KRaft mode) | enabled |
| elasticsearch | Search/indexing master node | enabled |
| elasticsearch-data | Search/indexing data node | enabled |
| redis | Session cache (egov-user tokens) | enabled |
| minio | Object storage (filestore fallback) | enabled |
| ingress-nginx | AWS NLB LoadBalancer + ingress controller | enabled |
| cert-manager | TLS certificate management (Let's Encrypt) | enabled |
| postgresql | In-cluster PostgreSQL | **DISABLED** (using RDS) |

---

## 3. Application Architecture

### Kubernetes Namespaces
```
backbone  — infrastructure services (Kafka, ES, Redis, Minio, Nginx)
egov      — all DIGIT application services
```

### Service Call Chain for TL New Application (Landmark → Next)
```
Browser
  └─► https://tl.pgrdigit.in/digit-ui/citizen/tl/tradelicence/new application/landmark
        │
        ▼
  ingress-nginx (backbone namespace, AWS NLB)
        │
        ▼
  gateway (egov namespace, Zuul API Gateway)
  ├── authenticates token via egov-user
  ├── checks access via egov-accesscontrol
        │
        ▼
  tl-services (egov namespace)  POST /tl-services/v1/_create
  ├── egov-idgen          → generates PB-TL-[date]-[SEQ_EG_TL_APL] application number
  ├── egov-mdms-service   → fetches TL trade categories, billing config (git-sync from egov-mdms-data DEV branch)
  ├── egov-user           → enriches applicant details
  ├── egov-workflow-v2    → initiates TL workflow (APPLY action)
  │     └── publishes to Kafka: save-wf-transitions
  │           └── egov-persister consumes → writes to RDS (tldb)
  ├── publishes to Kafka: save-tl-tradelicense
  │     └── egov-persister consumes → writes to RDS (tldb)
  └── returns TL application object → digit-ui advances to Step 3
```

### Kafka Topics (in-cluster kafka-kraft, backbone namespace)
| Topic | Producer | Consumer |
|---|---|---|
| `save-tl-tradelicense` | tl-services | egov-persister |
| `update-tl-tradelicense` | tl-services | egov-persister |
| `update-tl-workflow` | tl-services | egov-persister |
| `save-wf-transitions` | egov-workflow-v2 | egov-persister |
| `egov.core.notification.sms` | tl-services | egov-notification-sms |
| `persist-user-events-async` | tl-services | egov-user-event |

### Kafka Consumer Groups (unique per service — fixed)
| Service | Consumer Group |
|---|---|
| tl-services | `egov-tl-services` |
| egov-workflow-v2 | `egov-workflow-v2` |
| egov-persister | `egov-persister` |
| gateway | `egov-api-gateway` |
| tl-calculator | `tl-calculator` |

### Service Host Resolution (egov-service-host ConfigMap)
All services in `egov` namespace use `.egov` suffix for cross-namespace DNS:
```
tl-services    → http://tl-services.egov:8080/
tl-calculator  → http://tl-calculator.egov:8080/
egov-workflow-v2 → http://egov-workflow-v2.egov:8080/
egov-mdms-service → http://egov-mdms-service.egov:8080/
```
Backbone services use `.backbone` suffix:
```
Kafka  → kafka-kraft-controller-headless.backbone:9092
ES     → elasticsearch-data.backbone:9200
Redis  → redis.backbone:6379
```

---

## 4. Helmfile Value Merge Order

Helmfile loads values in this order (last wins):
```
1. charts/environments/env-secrets.yaml   (secrets: DB password, AWS keys, git SSH)
2. charts/environments/env.yaml           (overrides: DB endpoint, domain, service config)
```

The `env.yaml` `configmaps.egov-config.data` block deep-merges into `configmaps/values.yaml`,
overriding individual keys. This is how the RDS endpoint overrides the default `postgresql.backbone`.

---

## 5. CI/CD Workflows

### Workflow 1: `digit_install.yml` — Full Install (Terraform + Deploy)
**Trigger:** push to `release-githubactions` branch
**Use when:** provisioning a brand new environment from scratch

```
push to release-githubactions
        │
        ├─ check-changed-files     (detects deploy-as-code changes)
        │
        ├─ Input_validation        (reads input.yaml, runs init.go to patch variables.tf + env.yaml)
        │
        ├─ Terraform_Infra_Creation
        │   ├── terraform apply remote-state (S3 bucket for tfstate)
        │   ├── terraform apply main (EKS + RDS + VPC + node groups)
        │   ├── aws eks update-kubeconfig
        │   ├── envYAMLUpdater.go  ← writes RDS endpoint into env.yaml
        │   └── re-archives deploy-as-code with updated env.yaml
        │
        └─ DIGIT-deployment
            ├── aws eks update-kubeconfig
            ├── kubectl create namespace egov
            ├── helmfile -f digit-helmfile.yaml apply --include-needs=true
            └── prints LoadBalancer ID
```

**NOTE:** This workflow is still active on `release-githubactions`. Since infra is already done,
pushing to this branch will attempt terraform apply (idempotent — no changes if infra exists)
then deploy. Safe to use but slower. Use `digit_deploy_only.yml` for deploy-only.

### Workflow 2: `digit_deploy_only.yml` — Deploy Only (No Terraform)
**Trigger:**
- push to `release-githubactions` branch when `deploy-as-code/**` changes (auto)
- push to `deploy` branch when `deploy-as-code/**` changes (auto)
- `workflow_dispatch` (manual, choose cluster name + helmfile target)

**Use when:** infra already exists, deploying/upgrading DIGIT services only

```
push to release-githubactions (or deploy branch) with deploy-as-code changes
        │
        ├─ resolve-cluster
        │   ├── reads cluster_name from input.yaml (push trigger)
        │   └── or uses workflow_dispatch input
        │
        └─ digit-deploy
            ├── install: aws-iam-authenticator, kubectl, helm, helmfile, sops
            ├── aws eks update-kubeconfig --name digit-tl-eks
            ├── kubectl get nodes  (verify connectivity)
            ├── kubectl create namespace egov (idempotent)
            ├── kubectl create namespace backbone (idempotent)
            ├── validate env.yaml (no placeholders, RDS endpoint present)
            ├── helmfile -f digit-helmfile.yaml apply --include-needs=true
            ├── kubectl get pods -n egov
            └── prints LoadBalancer endpoint
```

### Workflow 3: `build.yml` — Build Docker Images
**Trigger:** `workflow_dispatch` (manual, choose pipeline: digit-ui / tl-services / tl-calculator / gateway)

```
workflow_dispatch (pipeline_name = tl-services)
        │
        ├─ resolve-config
        │   ├── checkout this repo
        │   ├── checkout subhgupta007/URBAN → urban-src/
        │   ├── reads build/build-config.yml
        │   └── resolves: work-dir, image-name, dockerfile, tag
        │
        ├─ build-matrix (linux/amd64)
        │   ├── docker buildx build --platform linux/amd64
        │   ├── pushes: $DOCKER_USERNAME/tl-services:$TAG-amd64
        │   └── pushes: $DOCKER_USERNAME/tl-services-db:$TAG-amd64
        │
        └─ create-manifest
            ├── docker manifest create $DOCKER_USERNAME/tl-services:$TAG
            └── docker manifest push
```

**Image tag format:** `{branch}-{short-commit-hash}` (e.g. `master-76f77645ab`)

---

## 6. Bugs Fixed (TL Landmark Crash)

The "Something went wrong!" crash on Landmark → Next was caused by 9 compounding issues:

| # | File | Bug | Fix |
|---|---|---|---|
| 1 | `env.yaml` | `db-url` had literal `<db_host_name>` placeholder | Set real RDS endpoint |
| 2 | `egov-workflow-v2/values.yaml` | `SPRING_KAFKA_CONSUMER_GROUP_ID: egov-tl-services` (same as tl-services) | Changed to `egov-workflow-v2` |
| 3 | `tl-services/values.yaml` | `pdf-link` and `host-link` values undefined → empty env vars | Added defaults |
| 4 | `configmaps/values.yaml` | `tl-services` and `tl-calculator` missing `.egov` namespace in service host | Added `.egov` suffix |
| 5 | `gateway/values.yaml` | `GATEWAY_ROUTES_TL_CALCULATOR_URL` hardcoded without namespace | Fixed to `.egov:8080` |
| 6 | `digit-helmfile.yaml` | 3 non-existent helmfile paths causing deploy failure | Commented out |
| 7 | `egov-persister/values.yaml` | git-sync branch `UAT` mismatched with env.yaml `UNIFIED-DEV` | Fixed to `UNIFIED-DEV` |
| 8 | `egov-idgen/values.yaml` | `idformat-from-mdms: false` disabled MDMS ID generation | Fixed to `true` |
| 9 | `env.yaml` | `egov-location` had `gmaps: true` but `gmapskey` in secrets is a placeholder (`jbsdbvxmbsmnx`) → Google Maps API returns 403 on Landmark step | Added `egov-location.gmaps: false` to env.yaml |
| 10 | `egov-workflow-v2/values.yaml` | `heap: "-Xmx64m -Xms64m"` — only 64MB JVM heap causes OOMKill on startup → CrashLoopBackOff for 3 days → tl-services APPLY transition fails | Increased to `heap: "-Xmx192m -Xms192m"` and added `memory_limits: "384Mi"` |
| 11 | `env.yaml` | `egov-state-level-tenant-id` not set in env.yaml → ConfigMap in cluster had stale value `pg` instead of `pb` → `egov-workflow-v2` queries MDMS with `tenantId=pg` at startup → `Missing property in path $['MdmsRes']['Workflow']` → CrashLoopBackOff | Added `egov-state-level-tenant-id: "pb"` explicitly to `configmaps.egov-config.data` in env.yaml |

---

## 7. GitHub Secrets Required

| Secret | Used By | Value |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | digit_install.yml, digit_deploy_only.yml | AWS IAM key |
| `AWS_SECRET_ACCESS_KEY` | digit_install.yml, digit_deploy_only.yml | AWS IAM secret |
| `AWS_DEFAULT_REGION` | digit_install.yml, digit_deploy_only.yml | `ap-south-1` |
| `AWS_REGION` | digit_install.yml, digit_deploy_only.yml | `ap-south-1` |
| `DOCKER_USERNAME` | build.yml | Docker Hub username |
| `DOCKER_ACCESS_TOKEN` | build.yml | Docker Hub access token |

---

## 8. Key Configuration Files

### `deploy-as-code/charts/environments/env.yaml`
The single source of truth for environment-specific config. Overrides chart defaults.
Critical values:
```yaml
db-host: "digit-tl-eks-db.cdoy2wumu54z.ap-south-1.rds.amazonaws.com"
db-url:  "jdbc:postgresql://digit-tl-eks-db.cdoy2wumu54z.ap-south-1.rds.amazonaws.com:5432/tldb"
kafka-brokers: "kafka-kraft-controller-headless.backbone:9092"
egov-services-fqdn-name: "https://tl.pgrdigit.in/"
```

### `deploy-as-code/charts/environments/env-secrets.yaml`
Secrets injected as Kubernetes Secrets. Contains DB credentials, AWS S3 keys, git SSH key.

### `infra-as-code/terraform/sample-aws/input.yaml`
Source of truth for infra parameters. Read by `init.go` and `digit_deploy_only.yml` (push trigger).
```yaml
cluster_name: "digit-tl-eks"
db_name: "tldb"
db_username: "tldbuser"
domain_name: "tl.pgrdigit.in"
```

---

## 9. How to Deploy (Current State)

Since infra is already provisioned, use `digit_deploy_only.yml`:

**Auto-trigger (recommended):**
Push any change to `deploy-as-code/**` on branch `release-githubactions`.
The workflow reads `cluster_name` from `input.yaml` automatically.

**Manual trigger:**
GitHub Actions → "DIGIT Deploy Only (Infra Pre-existing)" → Run workflow
- cluster_name: `digit-tl-eks`
- helmfile_target: `digit-helmfile.yaml`

**Verify after deploy:**
```bash
# Check TL services are running
kubectl get pods -n egov | grep tl

# Test the TL flow
# Login → Trade License → New Application → complete all steps through Landmark → Next
# Should proceed to Document Details (Step 3), not crash to /error
```
