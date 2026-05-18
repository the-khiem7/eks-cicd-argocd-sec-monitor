# Fresh EKS Cluster Setup

![AWS EKS](https://img.shields.io/badge/AWS%20EKS-Fresh%20Cluster-FF9900?logo=amazoneks&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Platform%20Runtime-326CE5?logo=kubernetes&logoColor=white)
![Argo CD](https://img.shields.io/badge/Argo%20CD-GitOps-EF7B4D?logo=argo&logoColor=white)
![Kyverno](https://img.shields.io/badge/Kyverno-Policy%20as%20Code-326CE5?logo=kubernetes&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy%20Operator-Cluster%20Reports-1904DA?logo=aqua&logoColor=white)
![Falco](https://img.shields.io/badge/Falco-Runtime%20Detection-00AEC7)
![Prometheus](https://img.shields.io/badge/Prometheus-Metrics-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-Dashboards-F46800?logo=grafana&logoColor=white)
![Loki](https://img.shields.io/badge/Loki-Logs-F46800?logo=grafana&logoColor=white)

This guide starts from a newly created EKS cluster where only the worker nodes are ready.

The setup order is:

```text
EKS access
-> runtime secrets
-> Argo CD
-> hospital app
-> security stack
-> monitoring stack
-> logging stack
-> UI access and verification
```

## 0. Start From Repo Root

```powershell
cd "<repo-root>"
```

`<repo-root>` is the project folder that contains `argocd/`, `k8s/`, `terraform/`, and this file.

## 1. Verify Cluster Access

```powershell
aws eks update-kubeconfig --region us-east-1 --name hospital-dev-eks
kubectl get nodes
kubectl get pods -A
```

Expected node result:

```text
STATUS
Ready
Ready
```

If `kubectl` times out against a `10.x.x.x:443` endpoint, the EKS API is still private from your workstation. Enable public endpoint access for your public IP `/32`, then retry.

## 2. Confirm Production GitOps Target

The current Argo CD app is configured to deploy the production overlay from the `main` branch:

```text
targetRevision: main
path: k8s/overlays/prod
namespace: hospital-prod
```

Render the production overlay locally:

```powershell
kubectl kustomize k8s/overlays/prod
```

## 3. Create Runtime Secrets

The backend needs a database connection string secret.

Replace the placeholder values before running:

```powershell
kubectl create namespace hospital-prod --dry-run=client -o yaml | kubectl apply -f -

kubectl -n hospital-prod create secret generic be-db-secret `
  --from-literal=default-connection='Server=<db-host>;Database=<db-name>;User Id=<db-user>;Password=<db-password>;TrustServerCertificate=True'
```

The workloads also reference an ECR pull secret named `ecr-registry-secret`.

Create it if the nodes cannot pull from ECR through IAM alone:

```powershell
aws ecr get-login-password --region us-east-1 | `
kubectl -n hospital-prod create secret docker-registry ecr-registry-secret `
  --docker-server=606030503959.dkr.ecr.us-east-1.amazonaws.com `
  --docker-username=AWS `
  --docker-password-stdin
```

Verify:

```powershell
kubectl get secret -n hospital-prod
```

## 4. Install Argo CD

```powershell
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -

kubectl apply -n argocd `
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for Argo CD:

```powershell
kubectl get pods -n argocd -w
```

All pods should become `Running` or `Completed`.

## 5. Apply Argo CD Applications

Apply the hospital app:

```powershell
kubectl apply -f argocd/hospital-traefik-app.yaml
```

Apply the security stack:

```powershell
kubectl apply -f argocd/security/10-kyverno-app.yaml
kubectl apply -f argocd/security/00-security-namespace-policies-app.yaml
kubectl apply -f argocd/security/20-trivy-operator-app.yaml
kubectl apply -f argocd/security/30-falco-app.yaml
```

Apply the monitoring stack:

```powershell
kubectl apply -f argocd/monitoring/10-kube-prometheus-stack-app.yaml
kubectl apply -f argocd/monitoring/20-monitoring-rules-app.yaml
```

Apply the logging stack:

```powershell
kubectl apply -f argocd/logging/10-loki-app.yaml
kubectl apply -f argocd/logging/20-promtail-app.yaml
kubectl apply -f argocd/logging/30-logging-config-app.yaml
```

## 6. Verify Argo CD Sync

```powershell
kubectl get applications -n argocd
kubectl describe application hospital-traefik-app -n argocd
```

Check runtime namespaces:

```powershell
kubectl get pods -n hospital-prod
kubectl get pods -n security
kubectl get pods -n monitoring
kubectl get pods -n logging
```

Check security resources:

```powershell
kubectl get clusterpolicy
kubectl get vulnerabilityreports -A
kubectl get configauditreports -A
```

Check monitoring resources:

```powershell
kubectl get prometheus,alertmanager -n monitoring
kubectl get prometheusrule -n monitoring
kubectl get servicemonitor -A
```

Check logging resources:

```powershell
kubectl get pods -n logging
kubectl get svc -n logging
kubectl get configmap -n monitoring loki-grafana-datasource
kubectl logs -n logging -l app.kubernetes.io/name=promtail --tail=100
```

## 7. Access Argo CD UI

Run port-forward and keep the terminal open:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

Get the initial admin password:

```powershell
kubectl get secret argocd-initial-admin-secret -n argocd `
  -o jsonpath="{.data.password}" | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

Login:

```text
username: admin
password: <password-from-command>
```

## 8. Access Grafana UI

Run port-forward and keep the terminal open:

```powershell
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Open:

```text
http://localhost:3000
```

Get the Grafana admin password:

```powershell
kubectl get secret -n monitoring kube-prometheus-stack-grafana `
  -o jsonpath="{.data.admin-password}" | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

Login:

```text
username: admin
password: <password-from-command>
```

Open Grafana Explore and choose the `Loki` datasource.

Useful LogQL:

```logql
{namespace="hospital-prod"}
{namespace="hospital-prod"} |= "error"
{namespace="security"} |= "falco"
```

## 9. Useful Debug Commands

Application pods:

```powershell
kubectl get all -n hospital-prod
kubectl describe pod -n hospital-prod -l app=be-v1
kubectl describe pod -n hospital-prod -l app=fe-v1
kubectl logs -n hospital-prod -l app=be-v1 --tail=100
kubectl logs -n hospital-prod -l app=fe-v1 --tail=100
```

Argo CD:

```powershell
kubectl get applications -n argocd
kubectl describe application hospital-traefik-app -n argocd
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=100
```

Security:

```powershell
kubectl get pods -n security
kubectl get clusterpolicy
kubectl get policyreport -A
kubectl logs -n security -l app.kubernetes.io/name=falco --tail=100
```

Monitoring:

```powershell
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl get prometheusrule -n monitoring
```

Logging:

```powershell
kubectl get pods -n logging
kubectl get svc -n logging
kubectl logs -n logging -l app.kubernetes.io/name=loki --tail=100
kubectl logs -n logging -l app.kubernetes.io/name=promtail --tail=100
```

## 10. Loki S3 Storage Plan

The default Loki setup stores logs on local filesystem storage so the dev cluster can start quickly. For long-lived logs, move Loki storage to S3.

Target design:

```text
Promtail
-> Loki
-> S3 buckets for chunks, ruler, and admin data
-> Grafana queries Loki
```

AWS resources to create before switching Loki to S3:

```text
S3 buckets: hospital-dev-loki-chunks-<account-id>, hospital-dev-loki-ruler-<account-id>, hospital-dev-loki-admin-<account-id>
Bucket encryption: SSE-S3 or SSE-KMS
Block public access: enabled
Lifecycle expiration: 7-30 days for dev, 30-90 days for production
IAM policy: ListBucket/GetObject/PutObject/DeleteObject scoped to those buckets
IRSA role: attached to Loki service account
```

After those resources exist, update `argocd/logging/10-loki-app.yaml`:

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

Keep this change in Git and let Argo CD resync Loki.

## 11. Common Issues

| Symptom | Likely cause | Fix |
|---|---|---|
| `kubectl get nodes` timeout to `10.x.x.x:443` | EKS API endpoint is private from your workstation. | Enable public endpoint access for your public IP `/32`, or run `kubectl` from inside the VPC. |
| Backend pod `CreateContainerConfigError` | `be-db-secret` is missing. | Create `be-db-secret` in `hospital-prod`. |
| Pod `ImagePullBackOff` | ECR auth or image tag issue. | Create `ecr-registry-secret`, verify ECR image exists, or confirm node IAM can pull ECR. |
| Argo app `OutOfSync` | Resources differ from Git. | Sync in UI or wait for auto-sync. |
| Argo app `Degraded` | Pod, CRD, or Helm chart issue. | Describe the Application and check namespace pod events. |
| Monitoring rules fail to apply | Prometheus Operator CRDs not ready yet. | Wait for `kube-prometheus-stack`, then re-apply `20-monitoring-rules-app.yaml`. |
| Kyverno policies fail to apply | Kyverno CRDs not ready yet. | Wait for Kyverno pods, then re-apply `00-security-namespace-policies-app.yaml`. |
| Loki datasource missing in Grafana | Grafana sidecar has not reloaded datasource ConfigMaps yet. | Restart Grafana or wait for the sidecar to pick up `loki-grafana-datasource`. |
| Promtail running but no logs in Loki | Loki gateway service is not ready or Promtail cannot push. | Check `kubectl get svc -n logging` and Promtail logs. |

## 12. Cleanup Commands

Delete Argo CD Applications only:

```powershell
kubectl delete -f argocd/logging/30-logging-config-app.yaml
kubectl delete -f argocd/logging/20-promtail-app.yaml
kubectl delete -f argocd/logging/10-loki-app.yaml
kubectl delete -f argocd/monitoring/20-monitoring-rules-app.yaml
kubectl delete -f argocd/monitoring/10-kube-prometheus-stack-app.yaml
kubectl delete -f argocd/security/30-falco-app.yaml
kubectl delete -f argocd/security/20-trivy-operator-app.yaml
kubectl delete -f argocd/security/00-security-namespace-policies-app.yaml
kubectl delete -f argocd/security/10-kyverno-app.yaml
kubectl delete -f argocd/hospital-traefik-app.yaml
```

Delete Argo CD itself:

```powershell
kubectl delete -n argocd `
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl delete namespace argocd
```

Destroy AWS infrastructure only when you are done:

```powershell
cd "<repo-root>\terraform\environments\dev"
terraform destroy
```
