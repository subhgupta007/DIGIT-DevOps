# DIGIT Deployment Guide (Infra Pre-existing)

## Architecture Overview

```
AWS EKS (digit-tl-eks)
├── namespace: backbone
│   ├── kafka-kraft          (in-cluster Kafka)
│   ├── elasticsearch        (in-cluster ES master)
│   ├── elasticsearch-data   (in-cluster ES data)
│   ├── redis                (in-cluster Redis)
│   ├── minio                (in-cluster object store, optional)
│   └── ingress-nginx        (AWS NLB LoadBalancer)
│
└── namespace: egov
    ├── gateway              (Zuul API gateway)
    ├── digit-ui             (React frontend)
    ├── tl-services          (Trade License backend)
    ├── tl-calculator        (TL fee calculator)
    ├── egov-workflow-v2     (Workflow engine)
    ├── egov-persister       (Kafka → DB writer)
    ├── egov-idgen           (ID generation)
    ├── egov-mdms-service    (Master data)
    ├── egov-user            (User management)
    ├── billing-service      (Billing)
    └── ... (other core services)

AWS RDS PostgreSQL (external, provisioned by Terraform)
    └── database: tldb  (all services connect via SPRING_DATASOURCE_URL)
```

## Before Running the Deploy Workflow

### Step 1 — RDS endpoint (already configured)

The RDS endpoint is:
```
digit-tl-eks-db.cdoy2wumu54z.ap-south-1.rds.amazonaws.com:5432
```
DB name: `tldb` | DB user: `tldbuser`

These are already set in `deploy-as-code/charts/environments/env.yaml`:
```yaml
db-host: "digit-tl-eks-db.cdoy2wumu54z.ap-south-1.rds.amazonaws.com"
db-name: "tldb"
db-url: "jdbc:postgresql://digit-tl-eks-db.cdoy2wumu54z.ap-south-1.rds.amazonaws.com:5432/tldb"
```

### Step 2 — Verify MDMS data repo

In `env.yaml`, the `egov-mdms-service` section points to:
```yaml
egov-mdms-service:
  initContainers:
    gitSync:
      repo: "git@github.com:egovernments/egov-mdms-data"
      branch: "DEV"
```

This must contain MDMS data for the `pb` state and `pb.amritsar` tenant including:
- `data/pb/TradeLicense/` — TL trade categories, structure types
- `data/pb/BillingService/` — TL billing slabs
- `data/pb/workflow-v2/` — TL workflow config

If you have a fork with this data, update the `repo` and `branch` values.

### Step 3 — Verify GitHub Secrets are set

In your GitHub repository Settings → Secrets → Actions:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_DEFAULT_REGION` (e.g. `ap-south-1`)
- `AWS_REGION` (e.g. `ap-south-1`)
- `DOCKER_USERNAME` (for build.yml)
- `DOCKER_ACCESS_TOKEN` (for build.yml)

### Step 4 — Run the deploy workflow

Go to GitHub Actions → "DIGIT Deploy Only (Infra Pre-existing)" → Run workflow:
- `cluster_name`: `digit-tl-eks`
- `helmfile_target`: `digit-helmfile.yaml` (full stack)

## Deployment Order (handled automatically by helmfile --include-needs)

1. backbone services (kafka, elasticsearch, redis, ingress-nginx)
2. configmaps (egov-config, egov-service-host)
3. egov-mdms-service (needs git-sync to pull MDMS data)
4. egov-enc-service (needs mdms)
5. egov-user (needs enc-service)
6. egov-idgen, egov-workflow-v2, egov-persister
7. tl-services, tl-calculator, billing-service
8. gateway, digit-ui

## Verifying the TL Landmark Fix

After deployment, test the full flow:
1. Login at `https://tl.pgrdigit.in/digit-ui/citizen`
2. Trade License → New Application
3. Complete Step 1 (Trade Details)
4. Complete Step 2 (Location) → Provide Landmark → click Next
5. Should proceed to Step 3 (Document Details) — NOT crash to /error

If it still crashes, check tl-services logs:
```bash
kubectl logs -n egov deployment/tl-services --tail=100
```
