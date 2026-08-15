# Integration

他のDocker Compose projectから `docker-manager` のTraefik / CoreDNSを利用する方法を説明します。

## 前提

先に `docker-manager` を起動し、shared networkが存在することを確認します。

```bash
docker compose up -d
docker network inspect docker-manager_web >/dev/null
docker network inspect docker-manager_dns >/dev/null
```

`.env` で `TRAEFIK_NETWORK` / `DNS_NETWORK` を変更している場合は、そのnetwork名を利用側projectでも合わせます。

## Traefikだけを利用する

host browserから `http://app.localhost` のようなURLでserviceへアクセスしたいだけなら、backend serviceを `web` networkへ接続します。

```yaml
services:
  app:
    image: example/app:1.0.0
    networks:
      - default
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.example-app.rule=Host(`app.localhost`)"
      - "traefik.http.routers.example-app.entrypoints=web"
      - "traefik.http.services.example-app.loadbalancer.server.port=3000"
      - "traefik.docker.network=${TRAEFIK_NETWORK:-docker-manager_web}"

networks:
  web:
    external: true
    name: ${TRAEFIK_NETWORK:-docker-manager_web}
```

`3000` はcontainer内でapplicationがlistenするportに置き換えます。Traefikとのservice-to-service通信にはhostへpublishしたportではなくcontainer portを使います。

`default` networkはそのproject固有の依存serviceとの通信に残し、Traefikから到達させるserviceだけをshared `web` networkへ参加させます。

## CoreDNSも利用する

containerから `app.localhost` や `keycloak.localhost` をTraefik経由で利用したい場合は、serviceを `dns` networkにも接続し、CoreDNSをDNS serverに指定します。

```yaml
services:
  app:
    image: example/app:1.0.0
    networks:
      - default
      - web
      - dns
    dns:
      - 172.31.0.10
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.example-app.rule=Host(`app.localhost`)"
      - "traefik.http.routers.example-app.entrypoints=web"
      - "traefik.http.services.example-app.loadbalancer.server.port=3000"
      - "traefik.docker.network=${TRAEFIK_NETWORK:-docker-manager_web}"

networks:
  web:
    external: true
    name: ${TRAEFIK_NETWORK:-docker-manager_web}
  dns:
    external: true
    name: ${DNS_NETWORK:-docker-manager_dns}
```

Docker Composeの `dns` はcontainerにcustom DNS serverを設定するためのservice属性です。

## Project固有のlocal nameを追加する

`docker-manager` 側でlocal overrideを作成します。

```bash
cp local/coredns/projects.conf.example local/coredns/projects.conf
```

例えば `app.localhost` をTraefikへ向ける場合は次を追加します。

```corefile
rewrite name app.localhost host.docker.internal
```

設定反映を確実にする場合はCoreDNSだけ再起動します。

```bash
docker compose restart coredns
```

`local/coredns/*.conf` はGit管理されないため、project固有名をこのrepositoryの共通設定へcommitする必要はありません。

## 複数network接続時のTraefik設定

serviceが `default` と `web` など複数networkへ接続する場合、次のlabelを省略しません。

```yaml
labels:
  - "traefik.docker.network=${TRAEFIK_NETWORK:-docker-manager_web}"
```

Traefik公式では、containerが複数networkへ接続している状態でnetworkを明示しない場合、backend接続に使うnetworkの選択は保証されません。

## 起動順と停止時の注意

利用側projectの `external` networkは `docker-manager` が作成します。そのため、利用側projectを起動する前に `docker-manager` のnetworkが存在する必要があります。

また `docker compose down` は、そのCompose projectが作成したnetworkも削除対象にします。利用側containerが `docker-manager_web` / `docker-manager_dns` に接続中の場合、network削除は失敗します。通常の一時停止では `docker compose stop` を使い、network自体を削除する場合は依存projectを先に停止・切断します。

## Troubleshooting

### `network ... declared as external, but could not be found`

`docker-manager` が起動しているか、network名が双方で一致しているか確認します。

```bash
docker network ls
docker network inspect docker-manager_web
docker network inspect docker-manager_dns
```

### Traefikから404になる

次を確認します。

- `traefik.enable=true` がある
- routerの `Host(...)` がアクセスURLと一致している
- backend serviceがshared `web` networkへ接続している
- `traefik.docker.network` が実際のshared network名と一致している
- `loadbalancer.server.port` がcontainer内のlisten portと一致している

### containerからlocal nameを解決できない

次を確認します。

- 対象containerが `dns` networkへ接続している
- `dns: 172.31.0.10` を設定している
- `local/coredns/*.conf` に必要なrewriteがある
- CoreDNSが起動している

```bash
docker compose ps coredns
docker compose logs coredns
```

## 参考資料

- Docker Compose: Use an existing external network
  - https://docs.docker.com/compose/how-tos/networking/#use-an-existing-network
- Docker Compose: Networks
  - https://docs.docker.com/reference/compose-file/networks/
- Docker Compose: Services (`dns`)
  - https://docs.docker.com/reference/compose-file/services/#dns
- Traefik: Docker provider
  - https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/
- Traefik: Docker routing labels
  - https://doc.traefik.io/traefik/reference/routing-configuration/other-providers/docker/
