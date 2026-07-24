# Coworking Space Analytics — EKS Deployment

Flask analytics API running on Amazon EKS, backed by PostgreSQL (Bitnami Helm chart) and delivered through an automated **CodeBuild → ECR → EKS** image pipeline.

---

## Architecture at a glance

Developer → S3 (source zip) → CodeBuild → ECR (versioned images) → kubectl apply → EKS

EKS runs:
- Deployment: `coworking` (Flask, port 5153)
- Service: `LoadBalancer` → AWS ELB
- StatefulSet: `postgresql` (Bitnami)
- Logs → CloudWatch Container Insights

| Component  | Choice                                                       |
|------------|--------------------------------------------------------------|
| Cluster    | EKS 1.34, 2× `t3.medium` managed nodes                       |
| Database   | Bitnami PostgreSQL Helm chart                                |
| App        | Python 3.10 Flask, image from ECR                            |
| Ingress    | Kubernetes `LoadBalancer` (AWS ELB) on port 5153             |
| CI         | CodeBuild (S3 source → build → push to ECR)                  |
| Logs       | CloudWatch Container Insights (`.../application` log group)  |

---

## Configuration

The app reads env vars — the image never contains config:

| Var                                            | Source                                             |
|------------------------------------------------|----------------------------------------------------|
| `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USERNAME` | `configmap/coworking-config`                       |
| `DB_PASSWORD`                                  | `secret/coworking-db-secret` (via `secretKeyRef`)  |

Change DB credentials → edit ConfigMap/Secret → `kubectl rollout restart deployment/coworking`. **No rebuild needed.**

---

## Releasing a new version

The pipeline is driven by an **image tag** in `buildspec.yaml` following SemVer (`MAJOR.MINOR.PATCH`).

**1. Bump `IMAGE_TAG`** in `buildspec.yaml` (e.g. `1.0.1` → `1.0.2`).

**2. Upload source to S3 and trigger CodeBuild:**

```bash
zip -r src.zip . -x "*.git*"
aws s3 cp src.zip s3://coworking-src-<account-id>/coworking-source.zip
aws codebuild start-build --project-name coworking-analytics-build
```

**3. Roll the cluster to the new tag:**

```bash
kubectl set image deployment/coworking \
  coworking=<account-id>.dkr.ecr.us-east-1.amazonaws.com/coworking-analytics:1.0.2
```

**4. Rollback if needed:**

```bash
kubectl rollout undo deployment/coworking
```

**Health gates during rollout:**
- **Liveness** — `/health_check` (restart on failure)
- **Readiness** — `/readiness_check` (runs `SELECT COUNT(*) FROM tokens` — pod stays out of the LB until Postgres is reachable)

---

## Standout notes

**Resource limits.** `requests: 100m CPU / 256Mi`, `limits: 500m CPU / 512Mi`. The API is I/O-bound (waits on Postgres), so a small baseline packs pods efficiently while the 5× CPU / 2× memory headroom absorbs report-generation spikes without letting a misbehaving pod starve its neighbors.

**Instance type.** `t3.medium` is the sweet spot — the bursty low-CPU workload is exactly what T-family credits are designed for, and it avoids the ENI-IP shortage seen on `t3.small`. For sustained production analytics traffic, step up to `m5.large` / `m6i.large` to remove the burst-credit ceiling.

**Cost savings:**
- Use **Spot** capacity for the stateless `coworking` workload (up to 90% off); keep Postgres on on-demand.
- Drop persistence + Bitnami exporter in non-prod (no EBS charges).
- Delete the ELB when idle — each one is ~$18/mo regardless of traffic. Use `port-forward` for occasional access.
- Set an **ECR lifecycle policy** to keep only the last 5 image tags.
- Add a **CloudWatch retention policy** (7–14 days) instead of "never expire".
< webhook test -->
