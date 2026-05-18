# Logging GitOps Applications

![Argo CD](https://img.shields.io/badge/Argo%20CD-GitOps-EF7B4D?logo=argo&logoColor=white)
![Loki](https://img.shields.io/badge/Loki-Logs-F46800?logo=grafana&logoColor=white)
![Promtail](https://img.shields.io/badge/Promtail-Log%20Agent-F46800?logo=grafana&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-Explore%20Logs-F46800?logo=grafana&logoColor=white)

This folder contains Argo CD `Application` manifests for installing and managing the EKS logging stack.

```text
argocd/logging = install/manage Loki, Promtail, and logging config
k8s/logging    = namespace and Grafana Loki datasource config
```

## Files

| File | Purpose |
|---|---|
| `10-loki-app.yaml` | Installs Loki from the Grafana Helm chart into the `logging` namespace. |
| `20-promtail-app.yaml` | Installs Promtail as a DaemonSet to ship pod logs to Loki. |
| `30-logging-config-app.yaml` | Syncs `k8s/logging`, including the Grafana datasource ConfigMap. |

## Apply Order

```bash
kubectl apply -f argocd/logging/10-loki-app.yaml
kubectl apply -f argocd/logging/20-promtail-app.yaml
kubectl apply -f argocd/logging/30-logging-config-app.yaml
```

## Verify

```bash
kubectl get applications -n argocd
kubectl get pods -n logging
kubectl get svc -n logging
kubectl get configmap -n monitoring loki-grafana-datasource
```

## Grafana Queries

Open Grafana Explore and choose the `Loki` datasource:

```logql
{namespace="hospital-prod"}
{namespace="hospital-prod", container=~".*be.*"} |= "error"
{namespace="security"} |= "falco"
{namespace="traefik"} |= "500"
```

## S3 Plan

The default Loki config uses local filesystem storage so the stack starts easily in a dev EKS cluster. For production or long-lived logs, switch Loki to S3-backed object storage.

Required AWS side:

```text
S3 buckets for chunks, ruler, and admin data
IAM policy allowing Loki to use those buckets
IRSA role for the Loki service account
Loki Helm values changed from filesystem to s3
```

Recommended bucket settings:

```text
Buckets: hospital-dev-loki-chunks-<account-id>, hospital-dev-loki-ruler-<account-id>, hospital-dev-loki-admin-<account-id>
Block public access: enabled
Versioning: optional
Default encryption: SSE-S3 or SSE-KMS
Lifecycle: expire old log objects after 7-30 days for dev, 30-90 days for production
```
