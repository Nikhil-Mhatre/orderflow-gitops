# Order Flow GitOps

This repository is the source of truth for Kubernetes desired state. Argo CD watches `k8s/`; GitHub Actions only commits image tag changes here and never runs `kubectl`.

## Layout
- `argocd/orderflow.yaml`: single root Argo CD Application.
- `k8s/kustomization.yaml`: all Kubernetes resources.
- `k8s/api.yaml`, `worker.yaml`, `web.yaml`: Deployments and Services.
- `k8s/ingress.yaml`: ALB class and ingress routes.
- `k8s/configmap.yaml`: non-secret configuration.
- `k8s/external-secret.yaml`: copies values from AWS Secrets Manager via External Secrets Operator.

## Before syncing
1. Replace `repoURL` in `argocd/orderflow.yaml`.
2. Replace ECR account/region/image tags in `k8s/api.yaml`, `worker.yaml`, and `web.yaml`.
3. Set DNS hostnames in `k8s/ingress.yaml`.
4. Set the RDS endpoint and SQS URL in `k8s/configmap.yaml`.
5. Install Argo CD and External Secrets Operator, and configure ESO's IAM access to the two named secrets.
6. Apply `kubectl apply -f argocd/orderflow.yaml`.

Check: `kubectl get applications -n argocd` and `kubectl get pods,svc,ingress -n orderflow`.

The manifests assume API port 3001 and `/health`, Web port 3000, and that the app reads the listed environment variables. Verify against actual source before deploying.
