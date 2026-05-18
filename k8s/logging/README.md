# Kubernetes Logging Stack

![Kubernetes](https://img.shields.io/badge/Kubernetes-Runtime%20Config-326CE5?logo=kubernetes&logoColor=white)
![Loki](https://img.shields.io/badge/Loki-Log%20Store-F46800?logo=grafana&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-Log%20Explore-F46800?logo=grafana&logoColor=white)

This folder contains runtime logging configuration for a Kubernetes cluster after Loki and Promtail are installed by Argo CD from `argocd/logging`.

```text
argocd/logging = install/manage logging tools
k8s/logging    = configure logging inside the cluster
```

## Components

| Component | Purpose |
|---|---|
| `logging` namespace | Shared namespace for Loki and Promtail. |
| Loki datasource ConfigMap | Lets Grafana discover Loki as a datasource. |
| Loki | Stores and indexes Kubernetes logs. |
| Promtail | Collects pod stdout/stderr from every node and ships it to Loki. |

## Deploy To Kubernetes

If Argo CD is already installed in the cluster, apply the logging Applications from the repo root:

```bash
kubectl apply -f argocd/logging/10-loki-app.yaml
kubectl apply -f argocd/logging/20-promtail-app.yaml
kubectl apply -f argocd/logging/30-logging-config-app.yaml
```

This creates or syncs:

```text
logging namespace
Loki
Promtail DaemonSet
Grafana Loki datasource ConfigMap
```

The default values are tuned for a small Kubernetes cluster:

```text
Loki request: 50m CPU, 128Mi memory
Promtail request per node: 25m CPU, 64Mi memory
Loki persistence: disabled
Loki canary: disabled
Loki chunks/results cache: disabled
```

If pods stay `Pending`, check node capacity and events:

```bash
kubectl get pods -n logging
kubectl describe pod -n logging -l app.kubernetes.io/name=loki
kubectl describe pod -n logging -l app.kubernetes.io/name=promtail
kubectl top nodes
```

If you are not using Argo CD, install the same stack with Helm and then apply this folder:

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install loki grafana-community/loki \
  --namespace logging \
  --create-namespace \
  --values <values-from-argocd/logging/10-loki-app.yaml>

helm upgrade --install promtail grafana/promtail \
  --namespace logging \
  --values <values-from-argocd/logging/20-promtail-app.yaml>

kubectl apply -k k8s/logging
```

## Log Flow

```text
Pod stdout/stderr
-> node log files
-> Promtail DaemonSet
-> Loki
-> Grafana Explore
```

## Useful Checks

```bash
kubectl get pods -n logging
kubectl get svc -n logging
kubectl logs -n logging -l app.kubernetes.io/name=promtail --tail=100
kubectl logs -n logging -l app.kubernetes.io/name=loki --tail=100
```

## Useful LogQL

```logql
{namespace="hospital-prod"}
{namespace="hospital-prod"} |= "error"
{namespace="hospital-prod", pod=~".*be.*"}
{namespace="security"} |= "falco"
```

## Object Storage Plan

Use filesystem storage only for dev or demos. For persistent logs, configure Loki with object storage such as S3 or an S3-compatible service like MinIO.

Production target on Kubernetes:

```text
Loki
-> object storage bucket
-> Grafana queries Loki
```

For AWS EKS, prefer IRSA. For a generic Kubernetes cluster, use an access key stored in a Kubernetes Secret or use your platform's workload identity equivalent.

Example IAM permissions for Loki buckets:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::<loki-chunks-bucket>",
        "arn:aws:s3:::<loki-ruler-bucket>",
        "arn:aws:s3:::<loki-admin-bucket>"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::<loki-chunks-bucket>/*",
        "arn:aws:s3:::<loki-ruler-bucket>/*",
        "arn:aws:s3:::<loki-admin-bucket>/*"
      ]
    }
  ]
}
```

After the buckets and credentials exist, update the Loki Helm values in `argocd/logging/10-loki-app.yaml`:

```yaml
loki:
  storage:
    type: s3
    bucketNames:
      chunks: <loki-chunks-bucket>
      ruler: <loki-ruler-bucket>
      admin: <loki-admin-bucket>
    s3:
      region: <aws-region>
  storage_config:
    aws:
      region: <aws-region>
      bucketnames: <loki-chunks-bucket>
      s3forcepathstyle: false
  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/<loki-irsa-role-name>
```

For non-EKS clusters, remove the `serviceAccount.annotations` block and provide S3 credentials through a Secret mounted or injected into the Loki chart values.
