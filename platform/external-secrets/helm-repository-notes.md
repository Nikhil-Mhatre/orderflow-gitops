# ESO Helm chart

Argo CD supports Helm charts directly, so the Application can point to the official chart repository without a separate HelmRepository CRD.

The `helm-release.yaml` in this folder is an Argo CD `Application` that installs the chart from:
`https://charts.external-secrets.io`

Update `targetRevision` to a tested, pinned chart version before using this beyond a disposable test environment.
