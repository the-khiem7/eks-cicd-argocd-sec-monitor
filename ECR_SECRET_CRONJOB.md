# ECR Image Pull Secret Refresh Cron

This guide documents how to automatically refresh the Kubernetes image pull secret for Amazon ECR.

ECR authorization tokens expire after a limited time, so clusters that use `imagePullSecrets` should refresh the secret periodically unless worker nodes already have IAM permission to pull from ECR directly.

## Example Values

```text
AWS account: <aws-account-id>
Region: <aws-region>
ECR registry: <aws-account-id>.dkr.ecr.<aws-region>.amazonaws.com
Kubernetes namespace: <namespace>
Secret name: <secret-name>
```

Do not commit real AWS account IDs, credentials, or private registry details if the repository is public. Keep real values on the EC2 host, in a private runbook, or in a secret manager.

## Prerequisites

Run these commands on the EC2 instance that has access to both AWS and the Kubernetes cluster.

Verify AWS access:

```bash
aws sts get-caller-identity
aws ecr get-login-password --region <aws-region> | head -c 20
echo
```

Verify Kubernetes access:

```bash
kubectl get nodes
kubectl auth can-i create secret -n <namespace>
kubectl auth can-i update secret -n <namespace>
kubectl auth can-i patch secret -n <namespace>
```

Check binary paths:

```bash
which aws
which kubectl
```

Example paths:

```text
/usr/local/bin/aws
/usr/bin/kubectl
```

## Create Refresh Script

Create the script:

```bash
cat > /home/ubuntu/refresh-ecr-secret.sh <<'EOF'
#!/bin/bash
set -e

AWS_REGION="<aws-region>"
AWS_ACCOUNT_ID="<aws-account-id>"
K8S_NAMESPACE="<namespace>"
SECRET_NAME="<secret-name>"
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

/usr/bin/kubectl create secret docker-registry "${SECRET_NAME}" \
  --docker-server="${ECR_REGISTRY}" \
  --docker-username=AWS \
  --docker-password="$(/usr/local/bin/aws ecr get-login-password --region "${AWS_REGION}")" \
  --namespace "${K8S_NAMESPACE}" \
  --dry-run=client -o yaml | /usr/bin/kubectl apply -f -
EOF

chmod +x /home/ubuntu/refresh-ecr-secret.sh
```

Before using the script, replace:

```text
<aws-region>
<aws-account-id>
<namespace>
<secret-name>
```

If `which aws` or `which kubectl` returns different paths, update the script to match the actual paths.

## Test The Script

Run:

```bash
/home/ubuntu/refresh-ecr-secret.sh >> /home/ubuntu/refresh-ecr-secret.log 2>&1
tail -n 50 /home/ubuntu/refresh-ecr-secret.log
```

Expected output:

```text
secret/<secret-name> configured
```

Verify the secret:

```bash
kubectl get secret <secret-name> -n <namespace>
```

## Add Crontab

Open crontab:

```bash
crontab -e
```

Add:

```cron
@reboot /sbin/swapoff -a
0 */6 * * * /home/ubuntu/refresh-ecr-secret.sh >> /home/ubuntu/refresh-ecr-secret.log 2>&1
```

This refreshes the ECR secret every 6 hours.

Verify:

```bash
crontab -l
```

Expected:

```cron
@reboot /sbin/swapoff -a
0 */6 * * * /home/ubuntu/refresh-ecr-secret.sh >> /home/ubuntu/refresh-ecr-secret.log 2>&1
```

## Check Cron Logs

View the refresh log:

```bash
tail -n 100 /home/ubuntu/refresh-ecr-secret.log
```

The cron runs at:

```text
00:00
06:00
12:00
18:00
```

## Deployment Requirement

Deployments that pull from private ECR must reference the secret:

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: <secret-name>
```

The same secret can pull both backend and frontend images if they are in the same ECR registry:

```text
<aws-account-id>.dkr.ecr.<aws-region>.amazonaws.com/<backend-repository>
<aws-account-id>.dkr.ecr.<aws-region>.amazonaws.com/<frontend-repository>
```

## Troubleshooting

If the script fails with `No such file or directory`, check the paths:

```bash
which aws
which kubectl
```

If AWS auth fails:

```bash
aws sts get-caller-identity
aws ecr get-login-password --region <aws-region>
```

If Kubernetes auth fails:

```bash
kubectl get nodes
kubectl auth can-i create secret -n <namespace>
kubectl auth can-i update secret -n <namespace>
```

If pods still cannot pull images:

```bash
kubectl describe pod -n <namespace> <pod-name>
kubectl get secret <secret-name> -n <namespace> -o yaml
```

Look for image pull errors such as:

```text
ImagePullBackOff
ErrImagePull
no basic auth credentials
```
