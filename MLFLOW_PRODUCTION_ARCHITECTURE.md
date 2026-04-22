# MLflow Production Architecture: Stateless + AWS Backend

## Problem: Current Stateless Setup

**What we have now:**

```
K8s MLflow Pod → SQLite (local disk)
                → Artifacts (pod filesystem)
```

**Issues:**

- Pod restarts → SQLite data lost ❌
- Filesystem artifacts lost on pod restart ❌
- Single pod = single point of failure ❌
- No multi-replica scaling (can't run 3 MLflow pods in parallel) ❌
- Not cloud-native ❌

---

## Solution: Production Stateful Architecture

```
┌─────────────────────────────────────────┐
│  K8s Cluster (stateless MLflow pods)    │
│  ├── MLflow Pod 1                        │
│  ├── MLflow Pod 2 (scalable)             │
│  └── MLflow Pod 3 (for HA)               │
└────────┬────────────────────────────────┘
         │
         ├──────────────────────────┬──────────────────┐
         │                          │                  │
         ▼                          ▼                  ▼
   ┌──────────────┐         ┌─────────────┐   ┌─────────────┐
   │ AWS RDS      │         │   AWS S3    │   │ AWS Secrets │
   │ PostgreSQL   │         │  (artifacts)│   │ (credentials)
   │(run metadata)│         └─────────────┘   └─────────────┘
   └──────────────┘
```

**Why this works:**

- ✅ PERSISTENT: Data survives pod restarts
- ✅ SCALABLE: Run 3-10 MLflow replicas (load-balanced by K8s Service)
- ✅ HA: One pod dies → others keep serving requests
- ✅ BEST PRACTICE: Separate compute (K8s) from storage (AWS managed services)
- ✅ ZERO DATA LOSS: RDS auto-backups, S3 replication built-in

---

## High-Level Implementation Approach

### 1. AWS Setup (Prerequisites)

```
Step 1: Create RDS PostgreSQL instance
  - Engine: postgres 14+
  - DB name: mlflow
  - Username: mlflow_user
  - Password: (store in AWS Secrets Manager)
  - Publicly accessible OR in same VPC as K8s

Step 2: Create S3 bucket
  - Name: <your-org>-mlflow-artifacts
  - Versioning: enabled
  - Encryption: SSE-S3

Step 3: Create IAM user (or role if using IRSA)
  - Permissions: S3 full access + RDS access
  - Export credentials as AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
```

### 2. K8s Setup (Helm Values)

```yaml
# mlflow-production-values.yaml

mlflow:
  # Point to RDS PostgreSQL
  backendStore: postgresql://mlflow_user:password@rds-endpoint:5432/mlflow

  # Point to S3
  defaultArtifactRoot: s3://your-bucket/mlflow-artifacts

  # Replicas for HA
  replicaCount: 3

  # AWS credentials (via Secret)
  env:
    - name: AWS_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: aws-creds
          key: access-key
    - name: AWS_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: aws-creds
          key: secret-key
    - name: AWS_DEFAULT_REGION
      value: "us-east-1"

# Ingress or LoadBalancer Service for external access
```

### 3. Deployment Steps (High-Level)

```
1. Create AWS resources (RDS, S3, IAM)
2. Create K8s Secret with AWS credentials
3. Generate Helm values with RDS + S3 config
4. helm upgrade --install mlflow with new values
5. Verify: MLflow UI tracks to AWS, pods are replicated
6. Test: Kill a pod → service auto-routes to others
```

---

## Data Flow: Training → MLflow → AWS

```
┌──────────────────┐
│ train.py         │
│ (your machine)   │
└────────┬─────────┘
         │
         │ mlflow.log_param()
         │ mlflow.log_metric()
         │ mlflow.log_model()
         │
         ▼
   ┌──────────────────────────────┐
   │ K8s MLflow Service           │
   │ (LoadBalancer or ClusterIP)  │
   └────────┬─────────────────────┘
            │
            ├─────────────────────────────────┐
            │                                 │
            ▼                                 ▼
      ┌──────────────┐              ┌──────────────────┐
      │ RDS          │              │ S3               │
      │ ├─ run_id    │              │ ├─ models/       │
      │ ├─ params    │              │ ├─ metrics.csv   │
      │ ├─ metrics   │              │ └─ datasets/     │
      │ └─ tags      │              └──────────────────┘
      └──────────────┘
```

---

## Benefits Over Current Setup

| Aspect                | Stateless (Current) | Stateful (Production) |
| --------------------- | ------------------- | --------------------- |
| **Data Persistence**  | Lost on restart ❌  | Permanent ✅          |
| **Scalability**       | 1 pod               | 3+ pods (HA)          |
| **Disaster Recovery** | Manual RDS backup   | Auto RDS backups      |
| **Cost**              | K8s compute only    | K8s + AWS services    |
| **Complexity**        | Simple              | Medium                |
| **Production Ready**  | No ❌               | Yes ✅                |

---

## Next Steps (When Ready)

1. **AWS Setup Phase**
   - Create RDS PostgreSQL instance
   - Create S3 bucket
   - Create IAM credentials

2. **Helm Integration Phase**
   - Write `mlflow-production-values.yaml`
   - Create K8s Secret for AWS credentials
   - Deploy via: `helm upgrade --install mlflow bitnami/mlflow -f mlflow-production-values.yaml`

3. **Testing Phase**
   - Run train.py with MLFLOW_TRACKING_URI pointing to K8s service
   - Verify metrics appear in MLflow UI
   - Kill MLflow pod → verify it auto-restarts with data intact
   - Test multi-pod routing

4. **Documentation Phase**
   - Create `AWS_SETUP_COMMANDS.md` (RDS + S3 creation)
   - Create `HELM_PRODUCTION_VALUES.md` (Helm chart config)
   - Update CI workflow to log experiments to production MLflow

---

## Terminology

- **Stateless Pod**: Can be replaced anytime; no local state
- **Persistent Backend**: External database (RDS) holds all state
- **Artifact Repository**: S3 or object storage for model files
- **IRSA**: IAM Roles for Service Accounts (K8s native AWS auth, no secrets needed)
- **RTO/RPO**: Recovery Time Objective (how fast), Recovery Point Objective (how much data loss)

---

## References

- [MLflow Backend Store Documentation](https://mlflow.org/docs/latest/tracking/backend-stores/)
- [MLflow Artifact Stores Documentation](https://mlflow.org/docs/latest/tracking/artifacts-stores/)
- [AWS RDS Best Practices](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/BestPractices.General.html)
- [AWS S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)

---

**Status**: High-level design document. Implementation in progress.
**Created**: April 19, 2026
**Next Review**: After AWS resources provisioned
