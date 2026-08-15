# Security

このstackは**信頼できる開発者が使うlocal-only環境**を前提とします。本番環境、共有server、Internet公開環境へそのまま流用しません。

## Hostへの公開範囲

TraefikのHTTP entrypointだけをhostのloopbackへbindします。

```yaml
ports:
  - "127.0.0.1:80:80"
  - "[::1]:80:80"
```

Portainer / Keycloakはhost portを直接publishせず、Traefik経由でのみアクセスします。CoreDNSも53番portをhostへpublishしません。

この構成はlocal accessの範囲を狭めるためのものであり、authenticationやTLSの代替ではありません。管理UIは外部networkへ公開しないでください。

## Keycloak

Keycloakは `start-dev` とH2 `dev-file` を使用します。

Keycloak公式ではdevelopment modeはinsecure defaultsを持つためproductionで使用しないよう明記されています。production modeではsecure-by-defaultの前提としてhostnameやHTTPS/TLS等が必要です。

このrepositoryでは次を前提とします。

- `start-dev` はlocal development専用
- `http://keycloak.localhost` はlocal HTTPのみ
- H2 `dev-file` はproduction databaseとして扱わない
- `.env` のbootstrap admin passwordにproduction credentialを使い回さない
- Keycloakのmanagement port `9000` はhostへpublishしない

Keycloak固有の運用は [keycloak.md](keycloak.md) を参照してください。

## Docker socket

TraefikとPortainerはDocker Engineへアクセスするため `/var/run/docker.sock` をmountします。

```text
Traefik   /var/run/docker.sock -> /var/run/docker.sock:ro
Portainer /var/run/docker.sock -> /var/run/docker.sock
```

Docker daemonを操作できる主体はhostに大きな影響を与えられるため、Docker socketへのaccessは明確なtrust boundaryとして扱います。

### Traefik

TraefikはDocker providerによるservice discoveryのためDocker APIへアクセスします。Traefik公式も、Docker APIへの無制限accessはsecurity concernであり、Traefikが侵害された場合にunderlying hostへ影響し得ると警告しています。

`docker.sock` を `:ro` でbind mountしていることを、Docker API権限がread-onlyに制限されているという意味には扱いません。Unix socket経由のAPI access自体がtrust boundaryです。

より強い分離が必要な場合は、Traefik公式が例示するsocket proxy、TLS/SSHで保護したDocker API endpoint、authorization plugin等を別途設計します。現行構成はlocal developmentの簡潔さを優先しています。

### Portainer

Portainerはlocal Docker environmentを管理するためDocker socketへ接続します。Portainer公式ではdirect Docker socket接続はlocal environment向けの方式として説明されています。

Portainerが侵害された場合、管理対象Docker environmentへ高い権限で操作される可能性があるため、Portainer UIを外部公開しません。

## Traefik service discovery

`config/traefik/traefik.yml` では次を設定しています。

```yaml
providers:
  docker:
    exposedByDefault: false
```

このため、Traefikへ公開するcontainerは明示的に次のlabelを持つ必要があります。

```yaml
labels:
  - "traefik.enable=true"
```

意図しないcontainerの自動公開を避けるため、`exposedByDefault: false` を維持します。

ただし、`traefik.enable=true` を付けたserviceはshared `web` network上でTraefikから到達可能になるため、label追加は公開境界の変更として扱います。

## Credential / local configuration

`.env` と `local/coredns/*.conf` はGit管理しません。

- `.env` にproduction secretを保存しない
- `.env` をcommitしない
- shared machineで使用する場合はOS file permissionも確認する
- project固有のDNS nameは `local/coredns/*.conf` に置き、共通設定へ不用意にcommitしない

Unix系OSで `.env` のowner以外からのreadを避けたい場合は、必要に応じてpermissionを制限します。

```bash
chmod 600 .env
```

## Productionへ持ち込まないもの

少なくとも次はproduction向け設計ではありません。

- Keycloak `start-dev`
- H2 `dev-file`
- loopback-only / plain HTTP前提のTraefik entrypoint
- 管理UIとapplication routeを同一local proxyで扱う構成
- Docker socketをcontainerへ直接mountする現在のtrust model
- developer-localなCoreDNS rewrite

production用途ではTLS、authentication、secret management、Docker API access制御、永続database、backup/restore、availabilityを要件から再設計します。

## 参考資料

- Keycloak: Running Keycloak in a container
  - https://www.keycloak.org/server/containers
- Keycloak: Configuring Keycloak
  - https://www.keycloak.org/server/configuration
- Keycloak: Configuring Keycloak for production
  - https://www.keycloak.org/server/configuration-production
- Traefik: Docker provider / Docker API Access
  - https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/
- Traefik: Docker routing labels
  - https://doc.traefik.io/traefik/reference/routing-configuration/other-providers/docker/
- Docker: Protect the Docker daemon socket
  - https://docs.docker.com/engine/security/protect-access/
- Portainer: Connect to the Docker Socket
  - https://docs.portainer.io/admin/environments/add/docker/socket
