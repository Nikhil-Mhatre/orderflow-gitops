## update kubectl with eks cluster to access

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name YOUR_EKS_CLUSTER_NAME
```

## Create the root application

```bash
kubectl apply -f argocd/root-application.yaml
```

## Port Forward Argocd

```bash
kubectl port-forward svc/argocd-server 8080:443 -n argocd
```

## Open the UI

```bash
http://localhost:8080
```

## Get the initial admin password

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
echo
```

## Get the database connection details

```bash
terraform output
```

## Create the Kubernetes Secret

```bash
read -s -p "Database password: " DB_PASSWORD
echo

kubectl create secret generic order-flow-secrets \
  -n order-flow \
  --from-literal=DATABASE_URL="postgresql://DB_USER:${DB_PASSWORD}@DB_HOST:5432/DB_NAME"

unset DB_PASSWORD
```

## Rollout deployments

```bash
kubectl rollout restart deployment/api -n order-flow
kubectl rollout restart deployment/worker -n order-flow
kubectl rollout restart deployment/frontend -n order-flow
```
