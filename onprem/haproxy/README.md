# HAProxy Edge Load Balancer

This folder runs HAProxy as the public edge load balancer for an on-prem Kubernetes cluster. HAProxy receives traffic on ports `80` and `443`, terminates TLS, and forwards HTTP to Traefik on worker NodePort `30080`.

## Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | Runs `haproxy:2.9` with host ports 80 and 443. |
| `haproxy.cfg.example` | Manual HAProxy config example. Copy it to `haproxy.cfg` and edit backend IPs. |
| `haproxy.cfg` | Ready-to-run HTTP config. Edit backend IPs when node IPs change. |
| `.env.example` | Runtime settings template. |
| `certs/` | Local HAProxy PEM certificates. Do not commit real keys. |

## Configure

```bash
cd onprem/haproxy
```

Edit `haproxy.cfg` only if your node IPs are different:

- Replace `server worker1` and `server worker2` with your Kubernetes worker IPs.
- Do not add the control-plane IP unless Traefik is explicitly scheduled on that node.
- Keep port `30080` unless you changed `onprem/traefik/04-traefik-nodeport-service.yaml`.

## TLS

The committed `haproxy.cfg` is HTTP-only so HAProxy can start before a TLS certificate exists.

When you want HTTPS, add a `frontend https_front` section and point it at a combined PEM file:

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
