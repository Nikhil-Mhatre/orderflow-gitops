# GitOps starter: API service on Amazon EKS

This repository is a starter for bootstrapping an API workload with Argo CD. It intentionally contains Kubernetes/GitOps manifests only; AWS infrastructure is managed separately by Terraform.

## Assumptions

- **Amazon EKS Auto Mode**, not Azure AKS. VPC, RDS for PostgreSQL, Secrets Manager, EventBridge, SQS, and DLQ are AWS services.
- Terraform creates the AWS infrastructure and the IAM permissions/Pod Identity association needed by the External Secrets Operator (ESO) and/or application.
- Argo CD is installed during infrastructure provisioning. One-time cluster bootstrap (such as registering this repo in Argo CD) is performed manually.
- The API image is already built and pushed to ECR by CI.
- RDS uses `manage_master_user_password = true`, so AWS RDS manages the master credential in Secrets Manager.

If you actually mean Azure AKS, this repository's AWS-specific External Secrets configuration and identity assumptions need to be replaced.

## Repository layout

```text
.
├── README.md
├── bootstrap/
│   └── root-app.yaml
├── platform/
│   └── external-secrets/
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── helm-repository-notes.md
│       └── helm-release.yaml
└── apps/
    └── api/
        ├── kustomization.yaml
        ├── base/
        │   ├── namespace.yaml
        │   ├── serviceaccount.yaml
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   ├── external-secret.yaml
        │   └── kustomization.yaml
        └── overlays/
            └── dev/
                ├── kustomization.yaml
                └── patch-replicas.yaml
```

## Before deploying

Replace these placeholders:
- `REPLACE_WITH_AWS_REGION`
- `REPLACE_WITH_AWS_ACCOUNT_ID`
- `REPLACE_WITH_ECR_REPOSITORY`
- `REPLACE_WITH_IMAGE_TAG`
- `REPLACE_WITH_RDS_MANAGED_SECRET_ARN`
- `REPLACE_WITH_EKS_CLUSTER_NAME` (only if used in your identity setup)

The RDS-managed secret ARN can include a generated suffix. Avoid manually copying the database host, port, or password into Kubernetes manifests. This starter uses the **AWS Secrets Manager secret ARN** as the lookup key, while ESO writes selected JSON fields to a Kubernetes Secret.

## Suggested bootstrap sequence

1. Apply `bootstrap/root-app.yaml` once, after Argo CD is installed and can access this Git repository. Update `repoURL` and target branch first.
2. The root Application deploys ESO and the API application.
3. ESO installs first as a platform component. Its CRDs/controllers must be ready before the `SecretStore` and `ExternalSecret` resources can reconcile. For a production setup, use a staged bootstrap: install ESO, wait for it to become healthy, then enable the API app. See the note below.
4. Configure AWS identity exactly once through Terraform: EKS Pod Identity association (recommended where supported) or IRSA. The IAM role used by ESO needs `secretsmanager:GetSecretValue` and `secretsmanager:DescribeSecret` on the target secret ARN, plus `kms:Decrypt` if a customer-managed KMS key encrypts it.
5. Ensure the API application reads the Kubernetes Secret keys that ESO creates. The example assumes the secret JSON contains `username`, `password`, `host`, `port`, and `dbname`; adjust `remoteRef.property` and `secretKey` to match the actual RDS-managed secret JSON.
6. Set the correct API container port, health endpoints, image, and environment variables for your app.

## Important AWS Secrets Manager detail

With `manage_master_user_password = true`, RDS creates/manages the master password secret. The secret JSON commonly contains keys such as `username`, `password`, `engine`, `host`, `port`, `dbClusterIdentifier`/`dbname` depending on the RDS resource and configuration. Verify the actual JSON in your account. The starter maps common fields but does not assume the DB host/port are hard-coded.

## Identity design

Prefer a dedicated IAM role for ESO with least-privilege read access to only the RDS secret. Associate it to the ESO controller service account using EKS Pod Identity (or IRSA if that is your chosen pattern). Do not put AWS access keys in Kubernetes Secrets. The API pod generally should not need permission to read Secrets Manager directly if ESO materializes the credential as a Kubernetes Secret.

## Operational cautions

- The database **master** account is usually not the best credential for an application. For production, create a least-privilege application database user and store that credential in Secrets Manager. This starter follows the requested master-secret path for initial testing.
- Kubernetes Secrets are base64-encoded, not encrypted by that fact alone. Enable EKS envelope encryption / appropriate cluster controls and tightly restrict RBAC.
- `ExternalSecret` syncs secret values into the cluster. Secret rotation behavior depends on the ESO refresh interval and the application’s ability to reload credentials.
- Pin the ESO chart version before production use; do not rely on an unbounded latest chart.
