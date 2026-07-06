---
name: ingress-traefik
description: |
  Traefik — reverse-proxy и единственная точка входа HTTP-трафика в сеть. LXC в DMZ, держит служебный WG-туннель к VPS, терминирует TLS, применяет middleware-цепочки и защиту CrowdSec. Документ описывает домены и сертификаты, конфигурацию, TLS, catch-all, middleware, доверие заголовкам, nftables, CrowdSec и метрики. Используй для вопросов по reverse-proxy, публикации сервисов, middleware, CrowdSec.
---

# Traefik

Traefik — reverse-proxy для всех сервисов и единственная точка, через которую HTTP-трафик попадает к бэкендам (и снаружи через VPS, и изнутри сети). Развёрнут LXC в сегменте DMZ, `192.168.40.11`. Держит служебный WireGuard-клиент `wg0` к VPS (второй конец туннеля публикации, см. `edge-vps.md`), терминирует TLS, применяет middleware и маршрутизирует запросы на бэкенды в SERVICES и других VLAN.

DMZ по замыслу содержит только Traefik — все чувствительные сервисы стоят в SERVICES, и Traefik ходит к ним по явным firewall-разрешениям. Базлайн LXC (unprivileged + `nesting=1`) — см. `conventions.md`.

## 1. Домены и сертификаты

Основной домен — `kvasok.xyz`, wildcard `*.kvasok.xyz` публикует self-hosted сервисы. Снаружи `*.kvasok.xyz` указывает на VPS (см. `edge-vps.md`); внутри сети Unbound через split-horizon резолвит те же имена в локальный адрес Traefik (`192.168.40.11`).

Сертификаты Let's Encrypt получает **Traefik через DNS-01 challenge** (wildcard `*.kvasok.xyz`). VPS сертификаты не хранит — он работает на L4. TLS-резолвер по умолчанию для `*.kvasok.xyz` — `timewebcloud`; для отдельных хостов есть `namecheap`.

## 2. Конфигурация

Главный конфиг — `/etc/traefik/traefik.yaml`. Динамические провайдеры — `/etc/traefik/dynamic/*.yaml` (по файлу на сервис плюс общий `config.yaml` со всеми middleware и TLS-опциями), директория читается с `watch: true`. Access-логи включены и используются парсером CrowdSec.

Entrypoint `websecure` (443) принимает PROXY protocol только от VPS внутри туннеля (`proxyProtocol.trustedIPs: ["10.0.0.1"]`). Метрики Prometheus отдаются на отдельном entrypoint `metrics` (`192.168.40.11:8081`) — секция `metrics.prometheus.entryPoint: metrics` в `traefik.yaml`. Изменения статической конфигурации (entryPoints) требуют полного рестарта сервиса — `watch: true` относится только к динамическому провайдеру.

## 3. TLS-опции

В `tls.options.default`: `minVersion: VersionTLS12`. Минимум именно 1.2, а не 1.3, потому что WebOS на LG TV не поддерживает TLS 1.3 — Jellyfin на нём при TLS 1.3 падает на handshake без записи в access-log. TLS 1.2 с современными cipher suites остаётся приемлемым уровнем. `curvePreferences`: X25519, P256, P384, P521. Современный набор cipher suites (ECDHE + GCM/ChaCha20). Default-сертификат-заглушка `/etc/traefik/certs/default.{crt,key}` отдаётся на запрос без подходящего SNI.

## 4. Catch-all роутер

В `config.yaml` определён роутер `catch-all` с `priority: 1` и правилом `HostRegexp(.+) || PathPrefix(/)`. Он ловит всё, что не подошло ни под один другой роутер, прогоняет через `crowdsec` и `block-all` (allow-list только `127.0.0.1/32`) и отдаёт `noop@internal`. Практический эффект: любой запрос на неизвестный хост получает 403 — нет утечки на default backend, нет шанса случайно открыть внутренний сервис из-за опечатки в конфиге.

## 5. Middleware

Конфигурация middleware разбита на строительные блоки и собранные из них цепочки.

### 5.1. Строительные блоки

| Middleware | Назначение |
| :--- | :--- |
| `crowdsec` | плагин CrowdSec (LAPI `127.0.0.1:8080`, mode `stream`, обновление раз в 60 сек, при недоступности LAPI работает с кэшем) |
| `block-all` | ipAllowList только `127.0.0.1/32` — фактически блок всего |
| `trusted` | ipAllowList доверенных внутренних сетей (см. 5.4) |
| `security-headers-base` | contentTypeNosniff, forceSTS, includeSubdomains, STS 180 дней, X-Forwarded-Proto=https — базовый набор для любых HTTP-клиентов, включая API |
| `security-headers-browser` | referrer policy, permissions policy (запрет camera/microphone/geolocation/USB/Bluetooth), CSP (`frame-ancestors 'self'`, `base-uri 'self'`, `form-action 'self'`) — заголовки только для браузерных клиентов |
| `rate-limit` | average 50, burst 20, period 1s |
| `rate-limit-strict` | average 10, burst 5, period 1s |
| `buffering` | отключение буферизации (лимиты в 0) — для стримов и больших аплоадов |

### 5.2. Цепочки

```yaml
headers-chain-html:
  chain:
    middlewares: [security-headers-base, security-headers-browser]

headers-chain-api:
  chain:
    middlewares: [security-headers-base]

internal-chain:
  chain:
    middlewares: [trusted, headers-chain-html]

internal-chain-api:
  chain:
    middlewares: [trusted, headers-chain-api]

external-chain:
  chain:
    middlewares: [crowdsec, rate-limit, headers-chain-html]

external-chain-api:
  chain:
    middlewares: [crowdsec, rate-limit, headers-chain-api]

external-chain-strict:
  chain:
    middlewares: [crowdsec, rate-limit-strict, headers-chain-html]

external-chain-no-rate-limit:
  chain:
    middlewares: [crowdsec, headers-chain-html]
```

### 5.3. Назначение цепочек

`internal-chain` — только доверенные внутренние сети. Применяется к внутренним сервисам: Traefik dashboard, Proxmox UI, Omada, Portainer, pgAdmin. `external-chain` — публичные сервисы с обычным rate-limit. `external-chain-strict` — где нужен жёсткий лимит (Vaultwarden). `external-chain-no-rate-limit` — где лимит мешает (Immich: листание галереи даёт множество параллельных запросов, при обычном лимите пользователя банит).

Многие сервисы дополнительно прикрыты Authelia через middleware `authelia` (forwardAuth), обычно как `internal-chain + authelia` или `external-chain + authelia` (см. `identity-authelia.md`). Одно осознанное исключение — Gotify, несовместимый с forward-auth (см. `services-core.md`).

### 5.4. Доверенные сети (`trusted`)

`trusted` ipAllowList покрывает внутренние сегменты, из которых разрешён доступ к internal-сервисам:

```
sourceRange:
  - 192.168.10.0/24   # MGMT network
  - 192.168.20.0/24   # INFRA network
  - 192.168.30.0/24   # TRUSTED network
  - 192.168.60.0/24   # IOT network
```

Важная особенность после перехода на маскарад AmneziaWG: VPN-клиенты приходят к Traefik под адресом AmneziaWG-LXC (`192.168.20.11`, INFRA), а не под своим `10.8.0.<N>`. Поэтому фактически VPN покрывается правилом `192.168.20.0/24` (INFRA). Побочный эффект: весь INFRA (включая Xray) попадает в `trusted` наравне с VPN — приемлемо, так как это доверенные инфраструктурные хосты. Подробнее про модель адресации VPN — в `vpn-amneziawg.md`.

## 6. Доверие к заголовкам

В CrowdSec-плагине `clientTrustedIPs` — доверенные внутренние сети (те же, что в `trusted`): запросы от них не банятся. `forwardedHeadersTrustedIPs: 10.0.0.1/32` — Traefik доверяет `X-Forwarded-*` заголовкам только от VPS (сосед по wg0). Это не даёт подделать реальный IP клиента через заголовок кому-либо, кроме доверенного VPS.

## 7. Файрвол (nftables)

Traefik — точка входа всего HTTP-трафика, поэтому фильтрация консервативна: whitelist с `policy drop`. В отличие от типовых сервисных LXC (см. `conventions.md`), у Traefik nftables специфичен — он терминирует туннель к VPS, принимает 443 из внутренних VLAN и отдаёт метрики Monitoring.

Что разрешено во входящих: loopback и conntrack established/related; базовые ICMP; SSH (22) из MGMT и AMNEZIAWG_IP; HTTPS (443) на `eth0` из внутренних доверенных VLAN (через split-horizon клиенты идут на `192.168.40.11`); HTTPS (443) на `wg0` от VPS (`10.0.0.1`, публичный трафик с PROXY protocol) и VPN-подсети; внутренний Traefik API (8079) только с DockerHost (виджет Homepage); метрики Traefik (8081) и CrowdSec (6060) только с Monitoring LXC (`192.168.50.21`).

```nft
#!/usr/sbin/nft -f

flush ruleset

define VPS_WG  = 10.0.0.1

define MGMT_NET = 192.168.10.0/24
define INFRA_NET = 192.168.20.0/24
define TRUSTED_NET = 192.168.30.0/24
define SERVICES_NET = 192.168.50.0/24
define IOT_NET = 192.168.60.0/24

define AMNEZIAWG_IP = 192.168.20.11
define MONITORING_IP = 192.168.50.21
define DOCKERHOST_IP = 192.168.50.30

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;

        # Loopback
        iifname "lo" accept

        # Conntrack
        ct state established,related accept
        ct state invalid drop

        # ICMPv4 - for ping and path MTU discovery
        ip protocol icmp icmp type {
            destination-unreachable,
            time-exceeded,
            parameter-problem,
            echo-request
        } accept

        # ICMPv6 - for NDP
        ip6 nexthdr icmpv6 icmpv6 type {
            destination-unreachable,
            packet-too-big,
            time-exceeded,
            parameter-problem,
            echo-request,
            echo-reply,
            nd-router-advert,
            nd-router-solicit,
            nd-neighbor-advert,
            nd-neighbor-solicit
        } accept

        # SSH from MGMT_NET and AMNEZIAWG_IP
        tcp dport 22 ip saddr { $MGMT_NET, $AMNEZIAWG_IP } accept

        # HTTPS from internal VLANs via split-horizon DNS (returns 192.168.40.11)
        iifname "eth0" tcp dport 443 ip saddr { $MGMT_NET, $INFRA_NET, $TRUSTED_NET, $SERVICES_NET, $IOT_NET } accept

        # HTTPS through wg0 from VPS (10.0.0.1) - public traffic with PROXY-protocol
        iifname "wg0" tcp dport 443 ip saddr $VPS_WG accept

        # Internal Traefik API - only with DockerHost (Homepage)
        iifname "eth0" tcp dport 8079 ip saddr $DOCKERHOST_IP accept

        # Prometheus metrics for Traefik - Monitoring (Prometheus)
        iifname "eth0" tcp dport 8081 ip saddr $MONITORING_IP accept

        # Prometheus metrics for CrowdSec - Monitoring (Prometheus)
        iifname "eth0" tcp dport 6060 ip saddr $MONITORING_IP accept

	# Everything else falls into policy drop
    }

    chain forward {
        type filter hook forward priority filter; policy drop;
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

SSH ужесточён общим drop-in `10-hardening.conf` (см. `conventions.md`), аутентификация по ключам, brute-force прикрыт коллекцией CrowdSec `crowdsecurity/sshd`.

## 8. CrowdSec

CrowdSec engine работает на Traefik LXC рядом с Traefik; bouncer интегрирован как плагин Traefik (Yaegi).

CrowdSec engine и плагин Traefik работают. Была проблема с загрузкой обновлений: плагин скачивается с `plugins.traefik.io`, а engine синхронизируется с `api.crowdsec.net` (CAPI), и оба хоста не поднимались через сеть провайдера напрямую. Решено заворотом исходящего трафика CrowdSec через Xray-прокси: в systemd-юните `crowdsec.service` прописаны переменные `HTTP_PROXY`/`HTTPS_PROXY` на HTTP-порт Xray (`http://192.168.20.12:10809`), после чего скачивание коллекций и CAPI-синхронизация проходят через гео-обход. Это добавляет зависимость engine от Xray-VM для обновлений и community-фида (при недоступности Xray новые коллекции и CAPI не обновятся, но локальные детекты и уже загруженные сценарии продолжают работать).

### 8.1. Коллекции

`crowdsecurity/appsec-generic-rules`, `crowdsecurity/appsec-virtual-patching`, `crowdsecurity/base-http-scenarios`, `crowdsecurity/http-cve`, `crowdsecurity/linux`, `crowdsecurity/sshd`, `crowdsecurity/traefik`, `crowdsecurity/whitelist-good-actors`.

### 8.2. Whitelist на уровне engine

Доверенные внутренние сети (MGMT, INFRA, TRUSTED, IOT) — их IP не банятся.

### 8.3. Нюансы

LAPI слушает только `127.0.0.1:8080`. Bouncer в режиме `stream` раз в 60 сек пуллит из LAPI полный список банов и держит кэш в памяти. При недоступности LAPI работает с последним кэшем: ранее забаненные остаются, новые не появляются до восстановления engine. Явного fail-closed (`defaultDecision: block`) у плагина нет — при длительном downtime новые атаки не фильтруются; частично страхует `Restart=always` в systemd (рестарт через 60 сек).

### 8.4. Метрики

CrowdSec engine отдаёт Prometheus-метрики на `192.168.40.11:6060` (`prometheus` в `/etc/crowdsec/config.yaml`: `enabled: true`, `level: full`, `listen_addr: 192.168.40.11`, `listen_port: 6060`). Доступ к порту на nftables открыт только Monitoring LXC. Метрики: активные баны по происхождению (свои детекты против community-фида CAPI) и причине, срабатывания сценариев, поток парсинга логов, запросы к LAPI. Скрейпит стек мониторинга (см. `monitoring.md`).

## 9. Метрики Traefik

Нативный Prometheus-эндпоинт на отдельном entrypoint `metrics` (`192.168.40.11:8081`). Даёт RPS, коды ответов по классам, латентность p95 по сервисам, трафик. Доступ к порту открыт только Monitoring LXC (nftables). Дашборд Traefik — см. `monitoring.md`.

## 10. Резервное копирование

Только PBS-снапшот всего LXC в составе общего ежедневного pve-задания. Отдельный restic не заводится — критичного point-in-time состояния у Traefik нет; конфиги (`/etc/traefik/`, `/etc/nftables.conf`, `/etc/wireguard/wg0.conf`) маленькие, статичные и восстанавливаются вместе с LXC из PBS. См. `backup.md`.

## 11. Зависимости

- **VPS (`edge-vps.md`)** — второй конец wg0-туннеля, источник публичного трафика с PROXY protocol.
- **Unbound на OPNsense** — split-horizon `*.kvasok.xyz → 192.168.40.11`, DNS для DNS-01 ACME.
- **Бэкенды в SERVICES** — Vaultwarden, Authelia, Gotify, Monitoring, DockerHost — цели проксирования (доступ по явным firewall-разрешениям).
- **Monitoring LXC (`192.168.50.21`)** — скрейпит метрики Traefik (8081) и CrowdSec (6060).
- - **Xray (`192.168.20.12`)** — HTTP-прокси для исходящего трафика CrowdSec (обновление коллекций, CAPI-синхронизация) через `HTTP_PROXY`/`HTTPS_PROXY` в `crowdsec.service`.
- **PBS (`192.168.10.15`)** — снапшоты LXC.
