# order-flow-gitops

GitOps repository for the Order Flow application. Argo CD reconciles this repository; Terraform owns AWS infrastructure.

## Repository layout

```text
order-flow-gitops/
├── argocd/
│   ├── root-application.yaml
│   └── applications/
│       └── order-flow-dev.yaml
├── clusters/
│   └── dev/
│       └── kustomization.yaml
├── apps/
│   └── order-flow/
│       ├── base/
│       │   ├── namespace.yaml
│       │   ├── frontend/
│       │   ├── api/
│       │   ├── worker/
│       │   ├── ingress/
│       │   └── kustomization.yaml
│       └── overlays/
│           └── dev/
│               ├── kustomization.yaml
│               └── configmap.yaml
└── docs/
```

## Before syncing

1. Update `argocd/applications/order-flow-dev.yaml` with this GitOps repository URL and branch.
2. Replace `ACCOUNT_ID` in `apps/order-flow/overlays/dev/kustomization.yaml` with your AWS account ID. The image names assume the ECR repositories created by Terraform:
   - `order-flow-dev/frontend`
   - `order-flow-dev/api`
   - `order-flow-dev/worker`
3. Confirm the service ports and environment variable names match the actual application code. The examples assume frontend and API listen on port `3000`.
4. Ensure the EKS worker Pod Identity association matches namespace `order-flow` and ServiceAccount `order-flow-worker`.
5. Create the `order-flow-secrets` Kubernetes Secret in the `order-flow` namespace before syncing workloads. Do not commit real credentials.

## Secret setup for the dev environment

This repository intentionally does not commit database credentials. Create `secrets.env` locally with the variable names expected by the app (adjust to your code):

```dotenv
DATABASE_URL=postgresql://APP_USER:APP_PASSWORD@DB_ENDPOINT:5432/orderflow
```

Then create the secret:

```bash
kubectl create namespace order-flow --dry-run=client -o yaml | kubectl apply -f -
kubectl -n order-flow create secret generic order-flow-secrets \
  --from-env-file=secrets.env \
  --dry-run=client -o yaml | kubectl apply -f -
rm secrets.env
```

Use a least-privilege application DB user. Do not put the RDS master password in Git. For a longer-lived setup, wire Secrets Manager to Kubernetes using External Secrets Operator or the Secrets Store CSI Driver and a dedicated IAM role. Terraform should own that integration and its IAM permissions.

## Install / bootstrap Argo CD

Argo CD is installed by Terraform. After it is installed and `kubectl` is configured for the Terraform-managed cluster:

```bash
kubectl apply -f argocd/root-application.yaml
```

The root Application tracks `clusters/dev`, which creates the `order-flow-dev` Application. That application deploys `apps/order-flow/overlays/dev`.

Alternatively, bootstrap the child Application directly:

```bash
kubectl apply -f argocd/applications/order-flow-dev.yaml
```

Use one bootstrap path, not both. The root-app pattern is recommended.

## Image publishing

CI should build and push immutable images to ECR, then update the image tags in `apps/order-flow/overlays/dev/kustomization.yaml` via a pull request. Avoid `latest` for production. The initial `dev` tags are placeholders and will not pull successfully until images exist in ECR.

## Routing and ALB

The Ingress uses EKS Auto Mode's ALB controller:
- `/api/*` routes to the `api` Service.
- `/*` routes to the `frontend` Service.
- Worker has no public Service or Ingress.

The frontend should call the API using the same-origin path `/api`. Confirm that the API accepts this prefix or configure an appropriate rewrite in the app/proxy layer. For production, add ACM HTTPS listener configuration and DNS, and verify security groups and health checks.

## Validate locally

Install `kubectl` and `kustomize` (or use `kubectl kustomize`), then run:

```bash
kubectl kustomize apps/order-flow/overlays/dev
kubectl kustomize clusters/dev
```

These commands render YAML; they do not contact the cluster. Then check the live cluster:

```bash
kubectl get applications -n argocd
kubectl get ingress -n order-flow
kubectl get pods -n order-flow
```

## Important

- The manifests are a starter template. Application-specific health endpoints, environment variables, port numbers, resource sizing, and API routing need to match the actual source code.
- Do not commit credentials, Kubernetes Secret manifests containing real values, kubeconfigs, or Terraform state.
- EKS Auto Mode provisions the ALB from the Ingress; do not install NGINX Ingress Controller or the AWS Load Balancer Controller for this standard Auto Mode setup.
