# Application integration contract

These manifests are infrastructure/deployment templates. The app images must implement the
same contract or the manifests must be adjusted to match the actual code.

## API
- Container port: 3001
- Readiness/liveness: `GET /health`
- Reads `DATABASE_URL` and `JWT_SECRET` from Kubernetes Secret `orderflow-secrets`.
- Reads `AWS_REGION`, `EVENTBRIDGE_BUS_NAME`, `SQS_QUEUE_URL`, `SQS_DLQ_URL` from ConfigMap.
- Publishes events to the EventBridge bus using AWS SDK `PutEvents` with source `orderflow`
  so the included EventBridge rule routes them to SQS.

## Worker
- Reads `DATABASE_URL` and `JWT_SECRET` from `orderflow-secrets` if required by the code.
- Reads `AWS_REGION`, `EVENTBRIDGE_BUS_NAME`, `SQS_QUEUE_URL`, `SQS_DLQ_URL` from ConfigMap.
- Polls SQS with long polling; acknowledges successfully processed messages with `DeleteMessage`.
- AWS SDK should use the workload identity credentials from the `orderflow-app` service account,
  not hard-coded AWS access keys.

## Frontend
- Container port 3000; Service port 80.
- Configure the frontend's API base URL/CORS to match the actual public API URL.
- Hostnames `api.example.com` and `frontend.example.com` are placeholders and must be replaced
  with real DNS names. TLS is not configured in this starter.

## Database
- Terraform creates the database but does not run migrations. Add a migration Job or run the
  application's documented migration command after RDS becomes available.
- `DATABASE_URL` is stored in Secrets Manager at `/orderflow/dev/database`.
- RDS password is Terraform-managed and is therefore present in Terraform state. Protect state.

## Important
This repository cannot guarantee app health until the real images, route names, environment
variables, and application event/message schemas are checked against the application source.
