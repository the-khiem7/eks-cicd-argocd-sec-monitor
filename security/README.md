# DevSecOps Security Tooling

![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![SonarQube](https://img.shields.io/badge/SonarQube-Code%20Quality-4E9BCD?logo=sonarqube&logoColor=white)
![Nexus](https://img.shields.io/badge/Nexus-Artifact%20Repository-1B1C30?logo=sonatype&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-Vulnerability%20Scanner-1904DA?logo=aqua&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-SonarQube%20DB-4169E1?logo=postgresql&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-Security%20Gate-2088FF?logo=githubactions&logoColor=white)
![DevSecOps](https://img.shields.io/badge/DevSecOps-Learning%20Lab-111827)

This folder contains the security servers and scan tooling used by the Hospital EKS GitOps platform.

![DevSecOps security tooling overview](./image.png)

## Learning Map

| Topic | Tool |
|---|---|
| Code quality and hotspots | SonarQube |
| Dependency and image vulnerability scanning | Trivy |
| IaC and Kubernetes manifest scanning | Trivy |
| Artifact repository and dependency proxy | Nexus |
| CI/CD security gate | GitHub Actions |

## Tooling

| Tool | Path | Purpose |
|---|---|---|
| SonarQube | `sonarqube/` | Static analysis, code quality, security hotspots, quality gates. |
| Nexus Repository | `nexus/` | Private artifact repository and dependency proxy. |
| Trivy | `trivy/` | Vulnerability, secret, IaC, and container image scanning. |

## Architecture

```mermaid
flowchart TB
  subgraph SecurityServer[Security Server]
    compose[Docker Compose]
    sq[SonarQube]
    pg[(PostgreSQL)]
    nx[Nexus]
    compose --> sq
    compose --> pg
    compose --> nx
    sq --> pg
  end

  subgraph CI[GitHub Actions]
    build[Build FE/BE]
    sonar[Sonar analysis]
    trivy[Trivy scan]
    artifact[Publish artifact/image]
  end

  build --> sonar
  sonar --> sq
  build --> trivy
  artifact --> nx
```

## Workflow

```mermaid
sequenceDiagram
  participant Dev as Developer
  participant GH as GitHub Actions
  participant SQ as SonarQube
  participant TV as Trivy
  participant NX as Nexus

  Dev->>GH: push or pull request
  GH->>GH: build backend and frontend
  GH->>SQ: send code analysis
  SQ-->>GH: quality gate
  GH->>TV: scan source, IaC, images
  TV-->>GH: vulnerability result
  GH->>NX: publish artifact when enabled
```

## Quick Start

```bash
cd security
docker compose up -d
docker compose ps
```

Open only from the server itself by default:

| Service | URL |
|---|---|
| SonarQube | `http://<host-ip-address>:9000` |
| Nexus | `http://<host-ip-address>:8081` |

To expose these services to another host, put Nginx, Caddy, Traefik, or an AWS ALB in front of them and enable HTTPS.

## Environment Files

`.env` files are included as editable placeholders:

| File | Purpose |
|---|---|
| `security/.env` | Docker Compose ports and SonarQube database settings. |
| `security/sonarqube/.env` | SonarQube URL/token placeholders. |
| `security/nexus/.env` | Nexus URL and credential placeholders. |
| `security/trivy/.env` | Trivy scan policy placeholders. |

Use real secrets in GitHub Actions secrets or a secret manager for production.

## Server Hardening

```bash
sudo apt update
sudo apt install -y curl uidmap dbus-user-session ufw fail2ban
sudo ufw allow OpenSSH
sudo ufw enable
```

Install Docker and enable rootless mode:

```bash
curl -fsSL https://get.docker.com | sudo sh
dockerd-rootless-setuptool.sh install
systemctl --user enable docker
systemctl --user start docker
sudo loginctl enable-linger "$USER"
docker --version
```

## CI/CD Gate

Typical order:

```text
Build -> Unit Test -> SonarQube Analysis -> Trivy Scan -> Publish artifact/image -> Argo CD Deploy
```

Minimum policy:

| Gate | Expected result |
|---|---|
| Build | FE/BE compile successfully. |
| SonarQube | Quality gate passes. |
| Trivy | No unapproved HIGH/CRITICAL findings. |
| Nexus | Artifacts publish with CI credentials only. |

## Documentation

- `sonarqube/README.md`
- `nexus/README.md`
- `trivy/README.md`
- `HARDENING.md`
- `SETUP.md`
