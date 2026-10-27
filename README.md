# GitOps starter — API service (EKS + Argo CD)

This repository is a starter for deploying **the API service only**. Terraform remains responsible for AWS infrastructure: VPC, EKS Auto Mode, RDS for PostgreSQL, Secrets Manager, IAM/EKS Pod Identity associations, EventBridge, SQS and the dead-letter queue. Argo CD manages Kubernetes add-ons and application manifests from this repository.

> Terminology note: this template assumes **AWS EKS**, not Azure AKS, because the infrastructure described uses VPC, RDS, Secrets Manager and EKS Auto Mode.

## What this repo manages

- External Secrets Operator (ESO), installed by Argo CD from its upstream Helm chart.
- A `ClusterSecretStore` that reads AWS Secrets Manager using the AWS credentials available to the ESO controller pod.
- An `ExternalSecret` that synchronizes the database credentials into a Kubernetes Secret.
- The API Deployment and Service, with the database Secret exposed as environment variables.

It does **not** create AWS infrastructure or IAM roles/policies. Those are Terraform responsibilities.

## Repository layout

```text
.
├── bootstrap/
│   ├── root-application.yaml
│   ├── orderflow-project.yaml
├── platform/
│   └── external-secrets/
│       ├── application.yaml
│       └── resources/
│           ├── kustomization.yaml
│           ├── cluster-secret-store.yaml
│           └── external-secret.yaml
└── apps/
    └── api/
        ├── base/
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   └── kustomization.yaml
        └── overlays/
            └── dev/
                ├── kustomization.yaml
                └── api-patch.yaml
```

## Before applying

Replace the example values in `platform/external-secrets/resources/external-secret.yaml` and the API overlay with your actual values.

Important integration assumptions:

1. The EKS cluster and Argo CD already exist.
2. You bootstrap Argo CD's root Application manually once the cluster is running.
3. The ESO controller's Kubernetes service account has an **EKS Pod Identity association** to an IAM role created by Terraform.
4. That IAM role can call `secretsmanager:GetSecretValue` and `secretsmanager:DescribeSecret` for the intended secret. Add `kms:Decrypt` if the secret uses a customer-managed KMS key.
5. The secret in Secrets Manager is JSON and contains the keys `username` and `password`. The example uses a fixed **secret name**, not a hard-coded ARN.
6. The API image understands the environment variables `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, and `DB_PASSWORD`. Adjust these names to match your application.

### Database endpoint and database name

An RDS-managed master secret generally stores credentials, not the RDS endpoint. Avoid manually copying the endpoint into Git. A clean pattern is for Terraform to create a separate application connection secret (or a JSON secret that includes `host`, `port`, and `dbname`) using RDS outputs and the managed master secret's value, with appropriate lifecycle and access controls. Alternatively, inject non-secret connection settings through a generated ConfigMap or another deployment-time configuration mechanism.

This template expects the referenced Secrets Manager secret to contain:
```json
{
  "username": "example",
  "password": "example",
  "host": "example.cluster-xxxx.region.rds.amazonaws.com",
  "port": "5432",
  "dbname": "appdb"
}
```
Do not commit real credentials to this repository. If you instead use the RDS-managed master secret directly, remove the `host`, `port`, and `dbname` property mappings from the `ExternalSecret` and supply those non-secret values separately.

## Bootstrap

After Argo CD is installed and you have manually configured access to this Git repository, apply the root Application:

```bash
kubectl apply -f bootstrap/root-application.yaml
```

Update the `repoURL` in that file to the URL of your GitOps repository. The root Application points Argo CD at the ESO Application and API application definitions.

## Recommended rollout order

1. Confirm the EKS Pod Identity association for the ESO controller service account exists.
2. Apply the root Application.
3. Wait for ESO to become healthy.
4. Confirm `ClusterSecretStore/aws-secrets-manager` is ready.
5. Confirm the `api-db-credentials` ExternalSecret has synced.
6. Deploy the API and check its logs and readiness.

Useful checks:

```bash
kubectl get applications -n argocd
kubectl get pods -n external-secrets
kubectl get clustersecretstore
kubectl describe clustersecretstore aws-secrets-manager
kubectl get externalsecret -n api
kubectl describe externalsecret api-db-credentials -n api
kubectl get secret api-db-credentials -n api
kubectl rollout status deployment/api -n api
```

Kubernetes Secret values are base64-encoded, not encrypted by base64. Restrict RBAC access to Secrets and enable EKS secret encryption / appropriate cluster security controls.

## Customization checklist

- Set `repoURL` and `targetRevision` in Argo CD Application manifests.
- Set the Secrets Manager secret name in `external-secret.yaml`.
- Set the API image in `apps/api/overlays/dev/api-patch.yaml`.
- Set the container port, health endpoints, and environment variable names to match the API.
- Ensure the ESO Pod Identity association uses the exact service account name and namespace installed by the Helm chart (`external-secrets` in namespace `external-secrets` by default).
- Configure the API's own AWS permissions separately if it later needs to read AWS services directly. The API should not need Secrets Manager permissions when ESO materializes the secret into Kubernetes.


## Note about Argo CD sync ordering

The API `ExternalSecret` lives in the API namespace. The API overlay creates that namespace, but Argo CD does not guarantee ordering between separate Applications. If ESO reports that namespace `api` does not exist, create it once during bootstrap or move the `Namespace` manifest into a small platform namespace app that syncs before the API. The root Application deploys both child Applications; it does not enforce a dependency between them.

For a first test, you can manually create the namespace with `kubectl create namespace api`, then let Argo CD manage the API workload.


## Argo CD project: `orderflow`

The child Applications (`external-secrets` and `api-dev`) use the dedicated Argo CD AppProject named `orderflow`, rather than `default`. The `bootstrap/orderflow-project.yaml` file defines allowed Git/chart sources, destination namespaces, and the `ClusterSecretStore` cluster-scoped resource required by External Secrets Operator. The root bootstrap Application intentionally remains in `default` so it can create the AppProject that the child Applications depend on.

The project allows only the configured GitOps repository and External Secrets chart repository. Replace `YOUR_ORG/YOUR_GITOPS_REPO` consistently before applying. Review and tighten resource permissions further if your cluster policy requires it.
