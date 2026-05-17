# Rancher Setup On Existing Kubernetes Cluster

This guide documents how to install Rancher on an existing Kubernetes cluster running on EC2 instances.

The example hostname used during setup was:

```text
3.211.47.196.sslip.io
```

Replace it with your own public IP or real DNS name when needed.

## Prerequisites

- A working Kubernetes cluster.
- `kubectl` configured to access the cluster.
- EC2 Security Group allows inbound traffic on:
  - `80/tcp`
  - `443/tcp`
- A public IP or DNS name that points to the node receiving ingress traffic.

Verify the cluster:

```bash
kubectl get nodes
kubectl get pods -A
```

## Install Helm

If `helm` is not installed:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 -o get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

Verify:

```bash
helm version
```

## Install cert-manager

Rancher uses cert-manager when Rancher-generated TLS certificates are used.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

Verify:

```bash
kubectl get pods -n cert-manager
```

Expected pods:

```text
cert-manager
cert-manager-cainjector
cert-manager-webhook
```

## Install Rancher

Add the Rancher Helm repository:

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

Create the Rancher namespace:

```bash
kubectl create namespace cattle-system
```

Install Rancher:

```bash
helm upgrade --install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=3.211.47.196.sslip.io \
  --set bootstrapPassword=Admin@123456 \
  --set replicas=1
```

Verify:

```bash
kubectl get pods -n cattle-system
kubectl get ingress -n cattle-system
```

## Install ingress-nginx

If the cluster does not already have an ingress controller, install ingress-nginx.

For an EC2-based cluster without a cloud LoadBalancer, use `hostNetwork=true` so nginx listens directly on node ports `80` and `443`.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx \
  --create-namespace \
  --set controller.kind=DaemonSet \
  --set controller.hostNetwork=true \
  --set controller.dnsPolicy=ClusterFirstWithHostNet \
  --set controller.service.type=ClusterIP \
  --set controller.ingressClassResource.default=true \
  --set controller.tolerations[0].key=node-role.kubernetes.io/control-plane \
  --set controller.tolerations[0].operator=Exists \
  --set controller.tolerations[0].effect=NoSchedule \
  --set controller.tolerations[1].key=node-role.kubernetes.io/master \
  --set controller.tolerations[1].operator=Exists \
  --set controller.tolerations[1].effect=NoSchedule
```

Annotate the Rancher ingress so nginx handles it:

```bash
kubectl annotate ingress rancher -n cattle-system kubernetes.io/ingress.class=nginx --overwrite
```

Verify that ingress-nginx is running on the node that owns the public IP:

```bash
kubectl get pods -n ingress-nginx -o wide
```

Verify the node is listening on `80` and `443`:

```bash
sudo ss -lntp | grep -E ':80|:443'
```

## Fix Rancher jail command crash

On some clusters, Rancher may start with this error:

```text
[FATAL] error running the jail command: exit status 2
```

If that happens, patch the Rancher deployment to run the Rancher container as privileged:

```bash
kubectl patch deployment rancher -n cattle-system --type='strategic' -p '{"spec":{"template":{"spec":{"containers":[{"name":"rancher","securityContext":{"privileged":true}}]}}}}'
```

Watch the rollout:

```bash
kubectl get pods -n cattle-system -w
```

Expected result:

```text
rancher-xxxxx   1/1   Running
```

## Access Rancher

Open:

```text
https://3.211.47.196.sslip.io
```

Login:

```text
Username: admin
Password: Admin@123456
```

Rancher will ask you to change the admin password after the first login.

If the browser shows a certificate warning, continue to the site. This is expected when using Rancher-generated self-signed certificates.

## Troubleshooting

Check Rancher pod status:

```bash
kubectl get pods -n cattle-system
kubectl logs -n cattle-system deploy/rancher --previous
```

Check ingress:

```bash
kubectl get ingress -n cattle-system -o wide
kubectl describe ingress rancher -n cattle-system
```

Check ingress-nginx:

```bash
kubectl get pods -n ingress-nginx -o wide
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=80
```

If the browser returns `ERR_CONNECTION_REFUSED`, check:

```bash
sudo ss -lntp | grep -E ':80|:443'
kubectl get pods -n ingress-nginx -o wide
```

If nginx is not running on the node with the public IP, make sure the DaemonSet has control-plane/master tolerations, or expose Rancher through a node that is actually running ingress-nginx.

## Useful Commands

Rollout status:

```bash
kubectl -n cattle-system rollout status deploy/rancher
```

Restart Rancher:

```bash
kubectl rollout restart deployment rancher -n cattle-system
```

List Rancher-related resources:

```bash
kubectl get all -n cattle-system
```

Uninstall Rancher:

```bash
helm uninstall rancher -n cattle-system
```

Uninstall ingress-nginx:

```bash
helm uninstall ingress-nginx -n ingress-nginx
```

