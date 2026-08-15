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

Docker daemonを操作できる主体はhostに大きな影響を与えられるため、Docker socketへのaccessは明確なtrust boundaryとして扱います。Traefik / Portainerは一般的なapplication containerより高いtrustを必要とするcontainerです。

`docker.sock` を `:ro` でbind mountしても、Docker APIのHTTP methodやendpointがread-onlyへ制限されるわけではありません。`ro`はmountされたfilesystem objectへのwriteを抑止する指定であり、Unix socketを経由したDocker API authorizationの代替にはなりません。

### Traefik

TraefikはDocker providerによるservice discoveryのためDocker APIへアクセスします。Traefik公式も、Docker APIへの無制限accessはsecurity concernであり、Traefikが侵害された場合にunderlying hostへ影響し得ると警告しています。

TraefikのDocker providerは `providers.docker.endpoint` でUnix socket以外のendpointも利用できます。公式ドキュメントではDocker socket proxy、TLS/SSHで保護したendpoint、authorization plugin等が選択肢として挙げられています。

#### Socket proxyの採否

**このrepositoryでは現時点でsocket proxyを導入せず、Traefikへのdirect socket mountを維持します。**

socket proxyを使うと、Traefikから許可するDocker API endpointやHTTP methodを絞れるため、Traefik侵害時の影響を小さくできる利点があります。一方、このlocal development stackでは次の理由から採用しません。

- PortainerはDocker管理用途のため引き続きDocker socketへ強い権限で接続し、stack全体としてDocker hostへのhigh-trust boundaryは残る
- socket proxy自体がDocker socketへ接続するhigh-trust componentになり、第三者image、設定、private network、version updateという運用対象が増える
- Traefik公式が例示するTecnativa Docker Socket Proxyも、proxy portをprivate networkだけへ限定し、必要APIだけを個別に許可する運用を要求する
- このrepositoryは信頼できる開発者のsingle-host local environmentを対象とし、Traefik entrypointもhost loopbackだけへpublishしている

したがって、現時点ではsecurity boundaryを隠蔽せず明示したうえで構成の単純さを優先します。これは「direct socketが安全」という判断ではなく、local-onlyという利用条件の下でresidual riskを受容する判断です。

#### Residual risk

Traefik processが侵害され、Docker socketへ到達する任意のAPI requestを実行できる状態になった場合、host上のDocker resourcesやhost自体へ影響が波及する可能性があります。

特に次の前提が崩れる場合はsocket proxyまたは別のDocker API access controlを再検討します。

- hostを複数の信頼境界の異なる利用者で共有する
- Traefik entrypointやdashboardをLAN / Internetへ公開する
- 信頼できないapplication containerをshared `web` networkへ参加させる
- productionまたはそれに近い長期稼働環境へ転用する
- Portainerを廃止し、TraefikだけがDocker API accessを必要とする構成になる

### Portainer

Portainerはlocal Docker environmentを管理するためDocker socketへ接続します。Portainer公式ではdirect Docker socket接続はlocal environment向けの方式として説明されています。

Portainerはcontainerの作成・更新・停止等を行う管理planeなので、Traefikより広いDocker API権限を必要とします。この用途ではdirect socket accessを維持します。

Portainerが侵害された場合、管理対象Docker environmentへ高い権限で操作される可能性があるため、Portainer UIを外部公開しません。

### Docker socketに対するsecurity controls

Direct socket accessを維持する代わりに、次を継続します。

- Traefikのhost公開はloopback `80` のみにする
- Portainerはhostへ直接port publishせずTraefik経由だけにする
- `providers.docker.exposedByDefault: false` を維持する
- Traefikのsocket mountは少なくとも `:ro` を維持する。ただしAPI authorizationとは扱わない
- Traefik / Portainer imageはpatch versionまで固定する
- Dependabotでpatch updateを継続検知し、CIのsmoke testを通す
- Traefikのminor / major updateはrelease notes / migration guideを確認する
- PortainerはLTS streamを維持し、同一LTS系列のpatchを追従する。新しいLTS系列への移行は手動確認する

image update運用の詳細は [operations.md](operations.md#image-update) を参照してください。

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
- Tecnativa: Docker Socket Proxy
  - https://github.com/Tecnativa/docker-socket-proxy
