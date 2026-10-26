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

## Check username & password in secret manager

## Encode the password correctly

```bash
node -e 'console.log(encodeURIComponent(process.argv[1]))' 'YOUR_CURRENT_PASSWORD'
```

## Create the Kubernetes Secret

```bash
read -s -p "Database password: " ENCODED_PASSWORD
echo

kubectl create secret generic order-flow-secrets \
  -n order-flow \
  --from-literal=DATABASE_URL="postgresql://DB_USER:${ENCODED_PASSWORD}@DB_HOST:5432/DB_NAME"

unset ENCODED_PASSWORD
```

## Rollout deployments

```bash
kubectl rollout restart deployment/api -n order-flow
kubectl rollout restart deployment/worker -n order-flow
kubectl rollout restart deployment/frontend -n order-flow
```

## View Secret

```bash
kubectl get secret order-flow-secrets -n order-flow \
  -o jsonpath='{.data.DATABASE_URL}' | base64 -d
echo
```

## Download Certificate from Official AWS site for RDS (Select appropriate region)

```bash
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html?utm_source=chatgpt.com#UsingWithRDS.SSL.CertificatesDownload
```

## Useful commands

```bash
kubectl logs deployments/api -n order-flow
kubectl logs deployments/worker -n order-flow
kubectl logs deployments/frontend -n order-flow
```

## BUilding and tagging images

```bash
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
export IMAGE_TAG=$(git rev-parse --short=12 HEAD)

docker build --platform linux/amd64 \
  -t "$ECR_REGISTRY/order-flow-dev/api:$IMAGE_TAG" \
  -f backend/api/Dockerfile backend/api
docker build --platform linux/amd64 \
  -t "$ECR_REGISTRY/order-flow-dev/worker:$IMAGE_TAG" \
  -f backend/worker/Dockerfile backend/worker
docker build --platform linux/amd64 \
  -t "$ECR_REGISTRY/order-flow-dev/frontend:$IMAGE_TAG" \
  -f frontend/Dockerfile frontend

docker build \
  --platform linux/amd64 \
  -f Dockerfile.migrate \
  -t "$ECR_REGISTRY/order-flow-dev/api:$IMAGE_TAG" .

docker push "$ECR_REGISTRY/order-flow-dev/api:$IMAGE_TAG"
docker push "$ECR_REGISTRY/order-flow-dev/worker:$IMAGE_TAG"
docker push "$ECR_REGISTRY/order-flow-dev/frontend:$IMAGE_TAG"
docker push "$ECR_REGISTRY/order-flow-dev/api:$IMAGE_TAG"
```

## Show files/folders as tree

```bash
find . \
  -path '*/.*' -prune -o \
  -path '*/node_modules' -prune -o \
  -print | sed -e 's;[^/]*/;|   ;g;s;|   \([^|]\);+--- \1;'
``
```
