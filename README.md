# docker-manager

Traefik / Portainer / CoreDNS / Keycloakを、ローカルのDocker開発環境で複数projectから共有するためのinfrastructure stackです。

**local development専用**です。管理UIやKeycloak `start-dev`、Docker socket accessを含むため、本番環境や外部公開用途には使用しません。

## Services

| Service | Role | Access | Persistent data |
| --- | --- | --- | --- |
| Traefik | reverse proxy / Docker service discovery | `http://traefik.localhost` | なし |
| Portainer | Docker管理UI | `http://portainer.localhost` | `data/portainer` |
| CoreDNS | container向けlocal DNS | `172.31.0.10:53` on `dns` network | なし |
| Keycloak | local authentication / identity provider | `http://keycloak.localhost` | `data/keycloak` |

service構成、image version、port、networkの実値は `docker-compose.yml` を正本とします。

## Prerequisites

- Docker Engine または Docker Desktop
- Docker Compose plugin (`docker compose`)

利用可能か確認します。

```bash
docker version
docker compose version
```

## Quick Start

```bash
git clone https://github.com/vnzzzz/docker-manager.git
cd docker-manager
cp .env.sample .env
```

`.env` にKeycloakのbootstrap管理者を設定します。

```dotenv
KC_BOOTSTRAP_ADMIN_USERNAME=admin
KC_BOOTSTRAP_ADMIN_PASSWORD=<password>
```

設定を検証して起動します。

```bash
docker compose config -q
docker compose up -d
docker compose ps
```

`keycloak` が `healthy` になったことを確認します。

主要URL:

- Traefik: `http://traefik.localhost`
- Portainer: `http://portainer.localhost`
- Keycloak: `http://keycloak.localhost`

## Documentation

- [Architecture](docs/architecture.md) — service / network / configuration / persistent dataの構成
- [Integration](docs/integration.md) — 他projectからTraefik / CoreDNSを利用するCompose例
- [Operations](docs/operations.md) — start / stop / backup / reset / update / troubleshooting
- [Security](docs/security.md) — local-only前提、Keycloak development mode、Docker socketのtrust boundary
- [Keycloak](docs/keycloak.md) — 初期設定、health check、H2 reset、version upgrade / realm export-import

## Local DNS

project固有のlocal nameが必要な場合はoverride fileを作成します。

```bash
cp local/coredns/projects.conf.example local/coredns/projects.conf
```

例:

```corefile
rewrite name example.localhost host.docker.internal
```

`local/coredns/*.conf` はGit管理されません。詳細は [Integration](docs/integration.md) を参照してください。

## Version policy

`docker-compose.yml` ではimageをpatch versionまで固定します。PortainerはLTS streamを使用します。

Dependabotは毎週Docker Compose imageとGitHub Actionsの更新を確認します。Docker imageのminor / major updateは個別にrelease notes / migration guideとCIを確認し、自動mergeは行いません。

## CI

Pull Requestと`main`へのpushでは `.github/workflows/ci.yml` を実行し、次を検証します。

- `docker compose config -q`
- stack起動
- Traefik / Portainer route
- Keycloak health
- CoreDNS name resolution
- failure時のdiagnosticsとcleanup

## Repository layout

```text
.
├── .github
│   ├── dependabot.yml
│   └── workflows/ci.yml
├── config
│   ├── coredns/Corefile
│   └── traefik/traefik.yml
├── data
│   ├── keycloak/
│   └── portainer/
├── docs
│   ├── architecture.md
│   ├── integration.md
│   ├── keycloak.md
│   ├── operations.md
│   └── security.md
├── local
│   └── coredns/
├── .env.sample
└── docker-compose.yml
```

- `config/`: Git管理する共通設定
- `data/`: Git管理しない永続runtime data
- `local/`: Git管理しない利用者・project固有設定
