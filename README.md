# GitOps repository for EKS + Argo CD

This repository is a starter GitOps layout for deploying an API, worker, database migration Job, and AWS EKS Auto Mode ALB ingress with Argo CD and Kustomize.

## Repository flow

1. Terraform creates AWS infrastructure and installs Argo CD with the Helm provider.
2. Argo CD's root Application tracks `environments/dev`.
3. Kustomize composes the environment resources.
4. GitHub Actions builds/pushes images to ECR and updates image references in this repository.
5. Argo CD reconciles committed changes into EKS.

## Before applying

Search for `REPLACE_ME` and replace the placeholders. In particular:
- Set the GitOps repository URL in `argocd/root-app.yaml` if you apply that file directly.
- Set the API/worker/migration image repository values in the overlays to your ECR repositories.
- Set the ingress hostname and confirm your EKS Auto Mode ingress settings and subnet discovery requirements.
- Set application configuration and Secret references to match your app.

Do not commit plaintext credentials. The Secret manifests here are examples with placeholder values only; preferably manage real secrets through AWS Secrets Manager and an integration such as External Secrets Operator.

## Deployment notes

- The root Application in `argocd/root-app.yaml` is an example. If Terraform already creates the root Application through the Helm chart's `extraObjects`, avoid creating a duplicate root Application with the same name. Either use the Terraform-managed root Application or apply this file, not both.
- `environments/dev/kustomization.yaml` composes the dev workload resources and shared ingress-class configuration.
- The migration Job is kept separate from the main environment root composition. Run it deliberately through a dedicated Argo CD Application or controlled CI/CD workflow. See `argocd/migration-app.yaml`.
- Job pod templates are immutable after creation. This example uses a versioned Job name (`api-migration-v1`) as a safe pattern. For each migration release, create a new uniquely named Job or use a controlled pipeline that deletes/recreates the Job after checking status.
- EKS Auto Mode ALB support depends on a valid Ingress and cluster/network configuration. Review AWS documentation for the current required IngressClass/IngressClassParams schema and subnet tagging for your cluster before applying.

## Validate locally

Requires `kubectl` and `kustomize` (or `kubectl kustomize`).

```bash
kubectl kustomize environments/dev
kubectl kustomize apps/migration/overlays/dev
```

Optionally validate rendered resources against your cluster:

```bash
kubectl apply --dry-run=server -k environments/dev
```

Server-side validation requires cluster access and the relevant CRDs to be installed.

## Image update convention

Keep images pinned to immutable tags or digests, for example a Git commit SHA. Avoid `latest` in shared environments. CI should build/push the image first, then commit the new image tag into the appropriate overlay.
