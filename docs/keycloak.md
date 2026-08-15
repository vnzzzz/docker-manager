# Keycloak

このリポジトリではKeycloakをローカル開発用途で使用します。`start-dev` とH2 `dev-file` を使用するため、本番環境には流用しません。

設定の正本は `docker-compose.yml` です。この文書では初期設定と永続データの扱いだけを説明します。

## 初回起動

`.env.sample` から `.env` を作成し、bootstrap管理者の認証情報を設定します。

```bash
cp .env.sample .env
```

```dotenv
KC_BOOTSTRAP_ADMIN_USERNAME=admin
KC_BOOTSTRAP_ADMIN_PASSWORD=<password>
```

未設定または空の場合はComposeのvariable interpolationで起動前にエラーにします。

```bash
docker compose up -d
```

Admin Consoleは `http://keycloak.localhost` から利用します。

`KC_BOOTSTRAP_ADMIN_USERNAME` / `KC_BOOTSTRAP_ADMIN_PASSWORD` は初期管理者の作成に使用します。既にmaster realmが存在する環境ではbootstrap admin設定は無視されます。

## Health check

`KC_HEALTH_ENABLED=true` を有効化し、container内部からmanagement port `9000` の `/health/ready` を確認します。management portはhostへpublishしません。

```bash
docker compose ps
```

`keycloak` が `healthy` になったことを確認します。

## Memory

Keycloak containerには `1g` のmemory limitを設定します。Keycloakはcontainer memoryを基準にheap sizeを計算するため、memory limitを設定しない構成にはしません。

## Version upgrade

### H2 `dev-file` の扱い

Keycloak公式では、既定のH2 `dev-file` databaseのversion migrationはサポートされていません。このリポジトリではKeycloakのmajor / minor versionを更新するとき、既存の `data/keycloak` を新versionで直接起動しません。

upgrade前にKeycloakを停止し、必ず `data/keycloak` をbackupします。

```bash
docker compose down

backup_file="../docker-manager-keycloak-$(date +%Y%m%d-%H%M%S).tar.gz"
tar -C data -czf "$backup_file" keycloak
```

その後は、用途に応じてresetまたはexport/importを選択します。

### Reset

realmやuserを引き継ぐ必要がない場合は、backup取得後にGitでignoreされているruntime dataだけを削除してfresh databaseを作成します。

まず削除対象を確認します。

```bash
git clean -ndX -- data/keycloak
```

内容を確認した後に実行します。

> 次のコマンドは `data/keycloak` のignored runtime dataを削除します。backupを確認してから実行してください。

```bash
git clean -fdX -- data/keycloak
docker compose up -d
```

### Realm / userを引き継ぐ場合

H2 database自体は移行せず、旧versionでrealm dataをexportし、新versionの空databaseへimportします。export/importには制約があり、session、event、workflow state、revoked tokenなどは引き継がれません。

24.0.5から26.xへ移行する場合は、codeを更新する前に旧versionでexportします。export時はKeycloak serverを停止します。

```bash
docker compose down

export_dir="../docker-manager-keycloak-export-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$export_dir"

docker run --rm \
  --mount type=bind,src="$PWD/data/keycloak",dst=/opt/keycloak/data \
  --mount type=bind,src="$export_dir",dst=/tmp/export \
  quay.io/keycloak/keycloak:24.0.5 \
  export --db=dev-file --dir /tmp/export
```

export完了後、codeを更新し、`.env` の管理者変数を `KC_BOOTSTRAP_ADMIN_USERNAME` / `KC_BOOTSTRAP_ADMIN_PASSWORD` へ変更します。続いて既存H2をresetし、exportしたrealm dataを新versionへimportします。

削除前に対象を確認します。

```bash
git clean -ndX -- data/keycloak
```

> 次のコマンドは `data/keycloak` のignored runtime dataを削除します。backupとexport結果を確認してから実行してください。

```bash
git clean -fdX -- data/keycloak
```

importするrealmには `master` realmと既存管理者が含まれます。Compose serviceには通常起動用のbootstrap admin環境変数が設定されているため、importではCompose serviceを使わず、Keycloak imageを直接起動します。これにより `KC_BOOTSTRAP_ADMIN_USERNAME` / `KC_BOOTSTRAP_ADMIN_PASSWORD` をimport processへ渡しません。

```bash
docker run --rm \
  --mount type=bind,src="$PWD/data/keycloak",dst=/opt/keycloak/data \
  --mount type=bind,src="$export_dir",dst=/tmp/export,readonly \
  quay.io/keycloak/keycloak:26.7.1 \
  import --db=dev-file --dir /tmp/export
```

`KC-SERVICES0032: Import finished successfully` の後にtemporary bootstrap admin作成エラーで終了した場合は、realm import自体が完了している可能性があります。まず通常起動して状態を確認します。

```bash
docker compose up -d
docker compose ps keycloak
```

Keycloakが`healthy`になり、Admin Consoleで既存管理者によるログインと必要なrealmを確認できれば再importは不要です。master realmが既に存在する場合、bootstrap admin設定は通常起動では無視されます。

通常起動できない、またはimportしたrealmが不足している場合だけ、backupとexport結果を確認したうえでH2 runtime dataをresetし、上記の`docker run`によるimportをやり直します。

起動後にAdmin Consoleへログインし、必要なrealm、client、userが引き継がれていることを確認します。

## 参考資料

- Keycloak: Running Keycloak in a container
  - https://www.keycloak.org/server/containers
- Keycloak: Bootstrapping and recovering an admin account
  - https://www.keycloak.org/server/bootstrap-admin-recovery
- Keycloak: Tracking instance status with health checks
  - https://www.keycloak.org/observability/health
- Keycloak: Upgrading Guide
  - https://www.keycloak.org/docs/latest/upgrading/
- Keycloak: Importing and exporting realms
  - https://www.keycloak.org/server/importExport
