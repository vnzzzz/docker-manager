# docker 管理用セット

## 目的

ローカルの Docker 開発環境の管理を簡単にするため、Traefik と Portainer を導入する。  
併せて、他プロジェクトから共通で利用する Keycloak / CoreDNS もここでホストする。

## ディレクトリ構成

- `config/`: Git 管理する共通設定
- `data/`: Keycloak / Portainer の永続データ。Git 管理しない
- `local/`: 利用者・プロジェクト固有のローカル設定

```text
.
├── README.md
├── .env
├── config
│   ├── coredns
│   │   └── Corefile
│   └── traefik
│       └── traefik.yml
├── data
│   ├── keycloak
│   └── portainer
├── local
│   └── coredns
│       └── projects.conf.example
├── docker-compose.yml
└── images
```

## 起動

1. `.env.sample` をコピーする。

   ```bash
   cp .env.sample .env
   ```

1. `.env` の `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD` を設定する。

1. 必要に応じて CoreDNS のローカル設定を作成する。

   ```bash
   cp local/coredns/projects.conf.example local/coredns/projects.conf
   ```

   `local/coredns/*.conf` は Git 管理されない。利用プロジェクト固有の名前解決だけをここへ記載する。

   ```corefile
   rewrite name example.localhost host.docker.internal
   ```

1. プロジェクトルートで起動する。

   ```bash
   docker compose up -d
   ```

## 管理コンソール

### Traefik

`http://traefik.localhost` にアクセスする。

![Traefik console](images/traefik-console.png)

### Portainer

`http://portainer.localhost` にアクセスする。

初回は admin ユーザーの作成が必要。

![Portainer console](images/portainer-console.png)

Containers から起動中のコンテナ情報を確認できる。

![Portainer containers](images/portainer-container.png)

### Keycloak

`http://keycloak.localhost` にアクセスする。

## CoreDNS 設定

`config/coredns/Corefile` は docker-manager 自体に必要な共通設定として Git 管理する。

利用プロジェクト固有の rewrite は `local/coredns/*.conf` に分離し、Corefile から次の glob import で読み込む。

```corefile
import local/*.conf
```

CoreDNS の `import` は空の glob を許容するため、ローカル設定を作成しなくても CoreDNS を起動できる。

## ネットワーク

- `web` は Traefik 用。実際の network 名は `${TRAEFIK_NETWORK:-docker-manager_web}`
- `dns` は CoreDNS 用。実際の network 名は `${DNS_NETWORK:-docker-manager_dns}`
- CoreDNS は `dns` network 上で `172.31.0.10` を使用する
- 他プロジェクトから利用する場合は、必要な network を external network として参照する

## 確認

起動しているコンテナは次のコマンドで確認できる。

```bash
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"
```
