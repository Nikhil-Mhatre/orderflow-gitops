# Deployment checklist

- [ ] Terraform infrastructure deployed successfully in `us-east-1`.
- [ ] ECR images built and pushed for `frontend`, `api`, and `worker`.
- [ ] Replace `ACCOUNT_ID` in dev Kustomize image names.
- [ ] Replace `REPLACE_WITH_TERRAFORM_PROCESSING_QUEUE_URL` with the Terraform output `processing_queue_url`.
- [ ] Confirm app ports, health endpoints, and environment variable names match application code.
- [ ] Create `order-flow-secrets` Secret in namespace `order-flow` using a least-privilege DB user.
- [ ] Confirm worker Pod Identity association is for namespace `order-flow` and ServiceAccount `order-flow-worker`.
- [ ] Update Argo CD repo URL in both Application manifests.
- [ ] Apply `argocd/root-application.yaml`.
- [ ] Wait for ALB provisioning and verify target health.
- [ ] Add HTTPS certificate and DNS before production use.
