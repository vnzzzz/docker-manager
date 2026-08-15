# docker-manager

Local Docker development infrastructure with Traefik, Portainer, CoreDNS and Keycloak.

ローカルのDocker開発環境で共通利用するインフラを管理します。
ローカル開発専用とし、Traefik / Portainer / Keycloak の管理UIを外部公開用途では使用しません。

## Setup

```bash
cp .env.sample .env
```

`.env` の `KC_BOOTSTRAP_ADMIN_USERNAME` / `KC_BOOTSTRAP_ADMIN_PASSWORD` を設定して起動します。

```bash
docker compose up -d
```

- Traefik: `http://traefik.localhost`
- Portainer: `http://portainer.localhost`
- Keycloak: `http://keycloak.localhost`

Keycloakの初期設定、health check、永続データのreset / version upgradeは [docs/keycloak.md](docs/keycloak.md) を参照してください。

## Screenshots

### Traefik

![Traefik dashboard](images/traefik-console.png)

### Portainer

![Portainer dashboard](images/portainer-console.png)

Containers から起動中のコンテナを確認できます。

![Portainer containers](images/portainer-container.png)

## Local DNS

利用プロジェクト固有の名前解決が必要な場合は、local overrideを作成します。

```bash
cp local/coredns/projects.conf.example local/coredns/projects.conf
```

```corefile
rewrite name example.localhost host.docker.internal
```

`local/coredns/*.conf` はGit管理されません。共通設定は `config/coredns/Corefile` を正本とします。

## Image versions

`docker-compose.yml` では image を patch version まで固定します。Portainer は LTS stream を使用します。
major / minor version を更新する場合は、各製品の release notes / migration guide を確認してから更新します。

## Dependency updates

Dependabotは毎週月曜日09:00（Asia/Tokyo）にDocker Compose imageとGitHub Actionsの更新を確認します。

- Docker Compose imageのpatch updateは1つのPRにまとめる
- Docker Compose imageのminor / major updateは個別PRとし、release notes / migration guideとCIを確認してからmergeする
- Portainerは現在のLTS minor系列内のpatch updateだけを自動提案対象とし、新しいLTS系列への移行はlifecycleを確認して手動で行う
- GitHub Actionsのpatch / minor updateはまとめ、major updateは個別PRにする

Dependabot PRも通常のPull Requestと同じCIを実行し、自動mergeは行いません。

## CI

Pull Requestと`main`へのpushでは `.github/workflows/ci.yml` を実行します。

- `docker compose config -q` によるCompose model検証
- Traefik / CoreDNSの必須設定ファイル確認
- stackの起動とTraefik containerのrunning確認
- Traefik経由のPortainer HTTP応答確認
- Keycloak health check確認
- CoreDNSへのDNS query確認

失敗時は `docker compose ps` と各serviceのlogsを出力してからcleanupします。

## Layout

```text
.
├── .github
│   ├── dependabot.yml
│   └── workflows
│       └── ci.yml
├── config
│   ├── coredns
│   │   └── Corefile
│   └── traefik
│       └── traefik.yml
├── data
│   ├── keycloak
│   └── portainer
├── docs
│   └── keycloak.md
├── local
│   └── coredns
│       └── projects.conf.example
├── docker-compose.yml
└── README.md
```

- `config/`: Git管理する共通設定
- `data/`: Git管理しない永続データ
- `local/`: Git管理しない利用者・プロジェクト固有設定
