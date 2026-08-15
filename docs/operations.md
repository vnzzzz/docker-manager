# Operations

日常運用、backup、reset、version update、troubleshootingをまとめます。Keycloak固有のdatabase移行は [keycloak.md](keycloak.md) を参照してください。

## 状態確認

```bash
docker compose ps
docker compose logs --tail=100
```

特定serviceだけ確認する場合:

```bash
docker compose logs --tail=100 keycloak
docker compose logs -f traefik
```

Compose modelだけを検証する場合:

```bash
docker compose config -q
```

## Start / stop / restart

stack全体を起動します。

```bash
docker compose up -d
```

containerとnetworkを残したまま一時停止する場合:

```bash
docker compose stop
```

再開:

```bash
docker compose start
```

特定serviceの再起動:

```bash
docker compose restart coredns
```

containerと、このCompose projectが作成したnetworkまで削除する場合:

```bash
docker compose down
```

`data/keycloak` / `data/portainer` はbind mountなので、`docker compose down` では削除されません。

他projectが `docker-manager_web` / `docker-manager_dns` に接続している状態ではnetworkを削除できません。共有networkを維持したい一時停止では `stop` を使います。

## Backup

KeycloakとPortainerの永続データを整合した状態で取得するため、backup前はstackを停止します。

```bash
docker compose down

backup_dir="../docker-manager-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$backup_dir"

tar -C data -czf "$backup_dir/runtime-data.tar.gz" keycloak portainer
```

backup後に再起動します。

```bash
docker compose up -d
```

Git管理されている `docker-compose.yml` / `config/` は通常のrepository履歴から復元します。`.env` はsecretを含むためGitへcommitしません。別途backupする場合もaccess権限を限定します。

Keycloakのmajor / minor upgradeでは、単純なdirectory backupだけでなくrealm exportが必要になる場合があります。詳細は [keycloak.md](keycloak.md) を参照してください。

## Reset

### Keycloak

削除対象をdry-runで確認します。

```bash
git clean -ndX -- data/keycloak
```

backupを確認してから実行します。

```bash
git clean -fdX -- data/keycloak
```

realm / userを保持するversion upgradeではこの手順だけを使わず、[keycloak.md](keycloak.md) のexport/import手順に従います。

### Portainer

Portainerの設定、local environment登録、user等を初期化する場合だけresetします。

```bash
git clean -ndX -- data/portainer
```

表示内容とbackupを確認してから実行します。

```bash
git clean -fdX -- data/portainer
```

次回起動時はfreshなPortainer data directoryとして初期設定が必要です。

## Image update

image tagの正本は `docker-compose.yml` です。`latest` は使用せず、patch versionまで固定します。

通常はDependabotが毎週version updateを確認します。

- Docker imageのpatch updateはgroup化する
- minor / major updateは個別にrelease notes / migration guideを確認する
- PortainerはLTS streamを維持し、同一LTS系列のpatchを追従する
- GitHub Actionsのpatch / minor updateはgroup化し、majorは個別確認する

変更後はPull RequestのCIでCompose modelとruntime smoke testを確認します。

手動でimageをpullして現在のtagを再取得する場合:

```bash
docker compose pull
docker compose up -d
docker compose ps
```

`docker compose pull` はCompose fileに記載されたtagを取得するだけなので、versionを上げる場合は先に `docker-compose.yml` のtagを更新します。

## Network名を変更する

既定network名が他projectと衝突する場合、`.env` で変更できます。

```dotenv
TRAEFIK_NETWORK=my-shared-web
DNS_NETWORK=my-shared-dns
```

変更後は利用側projectのexternal network名と `traefik.docker.network` も同じ値へ変更します。

`172.31.0.0/24` のsubnet自体が衝突する場合は、network名の変更だけでは解決しません。関連する固定IPとCI検証値も含めて変更が必要です。対象は [architecture.md](architecture.md#network変更時の注意) を参照してください。

## Troubleshooting

### required variable ... is missing a value

`KC_BOOTSTRAP_ADMIN_USERNAME` / `KC_BOOTSTRAP_ADMIN_PASSWORD` が `.env` に設定されているか確認します。

```bash
cp .env.sample .env
```

値を設定した後、Compose modelを確認します。

```bash
docker compose config -q
```

### `80` portをbindできない

別containerがport 80を使用していないか確認します。

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

Traefikはhostのloopback `80` を使用するため、既存serviceとの競合を解消してから起動します。

### Traefik routeが404になる

Traefikと対象serviceの状態を確認します。

```bash
docker compose ps traefik
docker compose logs --tail=100 traefik
```

他projectのrouteの場合は [integration.md](integration.md#traefikから404になる) のnetwork / labelも確認します。

### Keycloakがunhealthyになる

```bash
docker compose ps keycloak
docker compose logs --tail=200 keycloak
```

H2 file access、bootstrap設定、upgrade直後のdatabase状態を確認します。version upgrade後の場合は、既存H2 `dev-file` を直接migrationしない方針のため [keycloak.md](keycloak.md) を参照してください。

### CoreDNSでname resolutionできない

```bash
docker compose ps coredns
docker compose logs --tail=100 coredns
docker network inspect docker-manager_dns
```

network名を変更している場合は `docker-manager_dns` を実際の `DNS_NETWORK` 値へ置き換えます。

### Portainerへアクセスできない

```bash
docker compose ps portainer traefik
docker compose logs --tail=100 portainer traefik
```

Portainerはhostへ直接port publishしていないため、`http://portainer.localhost` はTraefikが動作していることも前提です。

## 参考資料

- Docker Compose CLI
  - https://docs.docker.com/reference/cli/docker/compose/
- `docker compose config`
  - https://docs.docker.com/reference/cli/docker/compose/config/
- `docker compose down`
  - https://docs.docker.com/reference/cli/docker/compose/down/
- Docker network
  - https://docs.docker.com/reference/cli/docker/network/
- Portainer lifecycle policy
  - https://docs.portainer.io/start/lifecycle
- Keycloak import / export
  - https://www.keycloak.org/server/importExport
