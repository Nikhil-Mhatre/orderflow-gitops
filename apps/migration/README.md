# Database migration Job

This directory is deliberately not referenced by `environments/dev/kustomization.yaml`.

Run it as an explicit, controlled release step after publishing the migration image and before promoting the API image that depends on the schema change. Confirm the migration command and container entrypoint match your application.

For repeatable executions, use a unique Job name per release/migration or have CI delete/recreate the Job under an explicit approval policy. Do not rely on changing an image tag on an already-completed Job to execute it again.
