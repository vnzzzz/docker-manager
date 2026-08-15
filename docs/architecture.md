# Architecture

`docker-manager` は、ローカルのDocker開発環境で複数projectから共有するinfrastructure stackです。
service構成の正本は `docker-compose.yml`、共通設定の正本は `config/` 配下です。

## 全体構成

```mermaid
flowchart LR
    host["Host browser / CLI"]
    client["Other project container"]
    embedded["Docker embedded DNS\n127.0.0.11"]
    coredns["CoreDNS\n172.31.0.10 / dns"]
    traefik["Traefik\nweb + dns"]
    portainer["Portainer\nweb"]
    keycloak["Keycloak\nweb + dns"]
    app["Other project service\nweb"]
    docker["Docker Engine API"]

    host -->|"127.0.0.1:80"| traefik
    client -->|"DNS query"| embedded
    embedded -->|"custom DNS"| coredns
    coredns -->|"configured local name"| traefik
    traefik --> portainer
    traefik --> keycloak
    traefik --> app
    traefik -->|"Docker socket"| docker
    portainer -->|"Docker socket"| docker
```

HostへpublishするHTTP portはTraefikの `127.0.0.1:80` / `[::1]:80` だけです。Portainer、Keycloak、CoreDNSはhostへ直接publishしません。

## 接続シーケンス

### Host browserからserviceへ（現行構成）

`http://keycloak.localhost` へのアクセス例です。

```mermaid
sequenceDiagram
    autonumber
    participant B as Browser
    participant R as Host resolver
    participant T as Traefik
    participant D as Docker Engine API
    participant K as Keycloak

    T->>D: labels / network / IP / portを取得・監視
    D-->>T: container metadata / events
    Note over T: Docker providerがrouting configurationを生成

    B->>R: keycloak.localhost を名前解決
    R-->>B: 127.0.0.1 / ::1
    B->>T: HTTP request<br/>Host: keycloak.localhost<br/>loopback :80
    T->>K: HTTP request<br/>web network / container port 8080
    K-->>T: HTTP response
    T-->>B: HTTP response
```

`.localhost` はloopback向けのspecial-use nameです。CoreDNSはhostへ53番portをpublishしていないため、この経路の名前解決には関与しません。

TraefikはDocker APIから得たcontainer metadataをもとにDocker provider内でrouting configurationを生成し、`Host` ruleに一致したrequestを `web` network経由でKeycloakへ転送します。

### Containerからlocal URLでserviceへ（現行構成）

`dns: 172.31.0.10` を設定したcontainerから `http://keycloak.localhost` へアクセスする例です。

```mermaid
sequenceDiagram
    autonumber
    participant A as Caller service
    participant E as Docker embedded DNS<br/>caller: 127.0.0.11
    participant C as CoreDNS<br/>172.31.0.10
    participant CE as Docker embedded DNS<br/>CoreDNS: 127.0.0.11
    participant T as Traefik<br/>172.31.0.11 / dns
    participant K as Keycloak

    A->>E: keycloak.localhost を名前解決
    E->>C: custom DNSへquery
    C->>C: rewrite name<br/>keycloak.localhost -> host.docker.internal
    C->>C: hosts /etc/hosts<br/>host.docker.internal -> 172.31.0.11
    C-->>E: keycloak.localhost -> 172.31.0.11
    E-->>A: 172.31.0.11

    A->>T: HTTP request<br/>Host: keycloak.localhost<br/>dns network
    T->>K: HTTP request<br/>web network / container port 8080
    K-->>T: HTTP response
    T-->>A: HTTP response

    opt rewrite / hostsに一致しないDNS query
        C->>CE: forward . 127.0.0.11
        CE-->>C: upstream DNS response
    end
```

user-defined network上のcontainerはDocker embedded DNS `127.0.0.11` を使います。Composeの `dns` で指定したCoreDNSは、そのupstream DNSとして利用されます。

CoreDNSのexact `rewrite name` はresponse nameも元へ書き戻します。図中の2つの `127.0.0.11` は、それぞれcaller containerとCoreDNS containerから見える別のembedded DNSです。

> `.localhost` はapplicationやresolver自身がloopbackとして処理し、DNS queryを送らない場合があります。この図はcallerがcontainerのDNS resolverを利用する場合の経路です。

### 同一Docker network内でservice名を使う場合（現行構成）

`http://service-name:port` のようにservice名を直接使う通信では、Docker embedded DNSが名前解決し、callerから対象containerへ直接接続します。CoreDNSとTraefikは経由しません。

### 別hostからCoreDNSを利用する場合（将来構成）

以下は**未実装の構成例**です。別hostのbrowserからCoreDNSを利用し、TraefikのDocker labelsに依存せずKeycloakへ直接接続します。

```mermaid
sequenceDiagram
    autonumber
    participant B as Browser<br/>別host
    participant R as Client resolver
    participant C as CoreDNS<br/>Docker host LAN :53
    participant H as Docker host<br/>LAN IP
    participant K as Keycloak

    B->>R: keycloak.dev.test を名前解決
    R->>C: DNS query<br/>TCP / UDP 53
    C-->>R: Docker host LAN IP
    R-->>B: Docker host LAN IP
    B->>H: HTTP(S) request<br/>published port
    H->>K: container application port
    K-->>H: HTTP(S) response
    H-->>B: HTTP(S) response
```

この構成では `.localhost` を使いません。`.localhost` はclient自身のloopback向けなので、例ではtesting用途に予約された `.test` を使います。

実装にはCoreDNSの53/TCP・53/UDPとKeycloakのapplication portのpublish、または別途direct routingが必要です。host外へ公開するため、firewall、TLS、Keycloak `hostname` などのsecurity設計も見直します。

Traefikを残してDocker labelsだけを使わない場合は、CoreDNSでTraefikの到達可能IPを返し、Traefikのfile providerでrouter / serviceを明示定義する方式があります。

## Service

| Service | Role | Network | Persistent data |
| --- | --- | --- | --- |
| Traefik | reverse proxy / Docker service discovery | `web`, `dns` | なし |
| Portainer | Docker管理UI | `web` | `data/portainer` |
| CoreDNS | container向けlocal name resolution | `dns` | なし |
| Keycloak | local authentication / identity provider | `web`, `dns` | `data/keycloak` |

image version、port、label、health checkなどの実値は `docker-compose.yml` を正本とします。

## `web` network

`web` はTraefikがbackend serviceへ接続するshared bridge networkです。既定名は `docker-manager_web` で、`.env` の `TRAEFIK_NETWORK` でnetwork名を変更できます。

他projectから利用するserviceは、このnetworkを `external: true` で参照します。routeを作成するserviceには `traefik.enable=true` が必要です。

serviceが複数networkへ接続する場合は、backend接続に使うnetworkを `traefik.docker.network` で明示します。

## `dns` network

`dns` はCoreDNSをcontainerから利用するshared bridge networkです。

| Item | Value |
| --- | --- |
| network name | `${DNS_NETWORK:-docker-manager_dns}` |
| subnet | `172.31.0.0/24` |
| CoreDNS | `172.31.0.10` |
| Traefik | `172.31.0.11` |

共通設定は `config/coredns/Corefile`、利用者固有のrewriteはGit管理しない `local/coredns/*.conf` に置きます。

```corefile
rewrite name example.localhost host.docker.internal
```

CoreDNS containerでは `host.docker.internal` を `172.31.0.11` に固定しています。rewrite / hostsに一致しないqueryはDocker embedded DNS `127.0.0.11` へforwardします。

## Configuration / data / local の境界

```text
config/   Git管理する共通設定
  coredns/Corefile
  traefik/traefik.yml

data/     Git管理しない永続runtime data
  keycloak/
  portainer/

local/    Git管理しない利用者・project固有設定
  coredns/*.conf
```

`data/` は再現可能な設定の正本として扱わず、必要に応じてbackupします。

## Network変更時の注意

`TRAEFIK_NETWORK` / `DNS_NETWORK` で変更できるのはnetwork名だけです。

`172.31.0.0/24` を変更する場合は、次を同じ変更単位で更新します。

- `docker-compose.yml` の `dns.ipam.config[].subnet`
- CoreDNSの `ipv4_address`
- Traefikの `dns` network側 `ipv4_address`
- CoreDNSの `extra_hosts` にあるTraefik IP
- `docs/integration.md` のDNS server例
- `.github/workflows/ci.yml` のCoreDNS検証値

## 参考資料

- RFC 2606: Reserved Top Level DNS Names (`.test`)
  - https://www.rfc-editor.org/rfc/rfc2606
- RFC 6761: Special-Use Domain Names (`localhost.`)
  - https://www.rfc-editor.org/rfc/rfc6761
- Docker Engine: Networking overview / DNS services
  - https://docs.docker.com/engine/network/
- Docker Engine: Port publishing and mapping
  - https://docs.docker.com/engine/network/port-publishing/
- Docker Compose: Networks
  - https://docs.docker.com/reference/compose-file/networks/
- Docker Compose: Services (`dns`, `networks`, `ports`)
  - https://docs.docker.com/reference/compose-file/services/
- CoreDNS: rewrite
  - https://coredns.io/plugins/rewrite/
- CoreDNS: hosts
  - https://coredns.io/plugins/hosts/
- CoreDNS: forward
  - https://coredns.io/plugins/forward/
- Traefik: Docker provider
  - https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/
- Traefik: File provider
  - https://doc.traefik.io/traefik/reference/dynamic-configuration/file/
- Keycloak: Configuring the hostname
  - https://www.keycloak.org/server/hostname
