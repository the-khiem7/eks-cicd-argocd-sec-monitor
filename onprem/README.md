# On-Prem HAProxy ALB

This folder provides the on-premise ingress path for the hospital platform. Use it when the Kubernetes cluster is not running on EKS or when you do not want to use the AWS Load Balancer Controller and AWS ALB.

```text
Internet
  -> HAProxy host, ports 80/443
  -> Kubernetes worker NodePort 30080
  -> Traefik DaemonSet
  -> Gateway API HTTPRoute
  -> hospital frontend/backend services
```

## Folder Layout

| Path | Purpose |
|---|---|
| `traefik/` | Kubernetes manifests that install Traefik as a NodePort Gateway API controller. |
| `haproxy/` | Docker Compose HAProxy edge load balancer with a manual config example. |

## When To Use This

Use this path for Rancher, kubeadm, bare metal, virtual machines, or any Kubernetes cluster where an external HAProxy server should replace AWS ALB.

Keep using the EKS/Terraform path when the cluster is on AWS and you want cloud-native ALB integration.

## Deploy

1. Install Gateway API CRDs and Traefik CRDs if your cluster does not already have them.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v3.1/docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml
```

2. Install Traefik as a NodePort ingress controller.

```bash
kubectl apply -k onprem/traefik
kubectl -n traefik get pods,svc
```

3. Deploy the hospital application.

```bash
kubectl apply -k k8s/overlays/prod
```

4. Apply the example Gateway and HTTPRoute after checking the namespace and domain.

```bash
kubectl apply -f onprem/traefik/10-app-gateway-routes.example.yaml
```

5. Run HAProxy on the edge host.

```bash
cd onprem/haproxy
cp .env.example .env
cp haproxy.cfg.example haproxy.cfg
# Edit haproxy.cfg and replace the worker backend IPs.
docker compose up -d
```

## Requirements

| Requirement | Notes |
|---|---|
| DNS | Point your domain to the HAProxy host public IP. |
| HAProxy inbound | Open TCP `80` and `443` to users. |
| Worker inbound | Open TCP `30080` from the HAProxy host to Kubernetes workers. |
| Worker IPs | Add Kubernetes worker IPs manually to `onprem/haproxy/haproxy.cfg`. |
| TLS | Put HAProxy PEM certificates in `onprem/haproxy/certs/`. |

## Verification

```bash
kubectl -n traefik get ds,svc,pods -o wide
kubectl get gateway,httproute -A
curl -I http://<your-domain>
curl -IL https://<your-domain>
docker exec haproxy-alb haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
```

Only HAProxy should terminate TLS and redirect HTTP to HTTPS. Traefik stays HTTP-only behind HAProxy.
