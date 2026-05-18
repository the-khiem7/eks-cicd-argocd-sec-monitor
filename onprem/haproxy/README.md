# HAProxy Edge Load Balancer

This folder runs HAProxy as the public edge load balancer for an on-prem Kubernetes cluster. HAProxy receives traffic on ports `80` and `443`, terminates TLS, and forwards HTTP to Traefik on worker NodePort `30080`.

## Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | Runs `haproxy:2.9` with host ports 80 and 443. |
| `haproxy.cfg.example` | Manual HAProxy config example. Copy it to `haproxy.cfg` and edit backend IPs. |
| `haproxy.cfg` | Local runtime config, ignored by Git. |
| `.env.example` | Runtime settings template. |
| `certs/` | Local HAProxy PEM certificates. Do not commit real keys. |

## Configure

```bash
cd onprem/haproxy
cp .env.example .env
cp haproxy.cfg.example haproxy.cfg
```

Edit `haproxy.cfg`:

- Change the certificate path if your domain is not `benhvien.teamdevops.shop`.
- Replace `server worker1`, `server worker2`, and `server worker3` with your Kubernetes worker IPs.
- Keep port `30080` unless you changed `onprem/traefik/04-traefik-nodeport-service.yaml`.

## TLS

HAProxy expects a combined PEM file:

```text
onprem/haproxy/certs/<domain>.pem
```

Create it from a Let's Encrypt certificate:

```bash
cat /etc/letsencrypt/live/<domain>/fullchain.pem \
    /etc/letsencrypt/live/<domain>/privkey.pem \
  > certs/<domain>.pem
chmod 644 certs/<domain>.pem
```

## Start

```bash
docker compose up -d
docker compose logs -f haproxy
```

## Reload After Manual Changes

```bash
docker exec haproxy-alb haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
docker kill -s USR2 haproxy-alb
```

## Verify

```bash
curl -I http://<domain>
curl -IL https://<domain>
docker exec haproxy-alb haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
docker exec haproxy-alb grep 'server worker' /usr/local/etc/haproxy/haproxy.cfg
```

## Troubleshooting

| Symptom | Check |
|---|---|
| Browser cannot connect | DNS, HAProxy host firewall, ports 80 and 443. |
| TLS error | PEM filename/path in `haproxy.cfg` does not match the certificate file. |
| `503 Service Unavailable` | Traefik pods, NodePort `30080`, worker firewall. |
| Reload fails | HAProxy syntax, container name, and edited backend entries. |
