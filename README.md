# docker 管理用セット

## 目的

ローカルの docker 開発環境の管理を簡単にするため、traefik と portainer を導入する。  
併せて、他プロジェクトから共通で利用する keycloak / coredns もここでホストする。

## ディレクトリ構成

### 概説

- traefik と portainer は 1 つの docker-compose.yml で準備する（イメージは公式を直接利用）
- 今後作成する docker プロジェクトは、それぞれ別のプロジェクトディレクトリの中で docker-compose.yml を作成し、その際に label で traefik の管理情報を付記する
- https 対応は未実施

### 構成

```bash
# docker-mgr：portainer+traefik
.
├── README.md
├── .env
├── data
│   ├── coredns          # Corefile 管理
│   ├── keycloak         # Keycloak data
│   ├── portainer        # Portainer data
│   └── traefik          # Traefik 設定 (traefik.yml)
├── docker-compose.yml
└── images               # README 用画像
```

## 手順

1. traefik と portainer を立ち上げる

   プロジェクトルートで下記を実行する

   ```bash
   docker-compose up -d
   ```

1. traefik の管理コンソールへのログインを確認する

   `http://traefik.localhost`にブラウザでアクセスする

   ![picture 1](images/traefik-console.png)

1. portainer の管理コンソールへのログインを確認する

   `http://portainer.localhost`にブラウザでアクセスする

   ![picture 2](images/portainer-console.png)

   > 初回は admin ユーザー作成を求められるので、適当な password を入力する

   Containers から、起動中のコンテナ情報が確認できる

   ![picture 3](images/portainer-container.png)

補足：起動しているコンテナの情報は下記で確認できる。

```bash
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"
```

補足：

- `web` ネットワークは traefik 用（名前は `${TRAEFIK_NETWORK}`）で、他プロジェクトは外部ネットワークとして参照する
- `dns` ネットワークは coredns 用（名前は `${DNS_NETWORK}`）で、`172.31.0.10` に固定された coredns を他プロジェクトが利用する
