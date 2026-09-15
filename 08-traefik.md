---
name: traefik
description: |
  Traefik — reverse-proxy и единственная точка входа HTTP-трафика в сеть. LXC в DMZ, держит служебный WG-туннель к VPS, терминирует TLS, применяет middleware-цепочки и защиту CrowdSec. Документ описывает домены и сертификаты, конфигурацию, TLS, catch-all, middleware, доверие заголовкам, nftables, CrowdSec и метрики. Используй для вопросов по reverse-proxy, публикации сервисов, middleware, CrowdSec.
---

# Traefik

Traefik — reverse-proxy для всех сервисов и единственная точка, через которую HTTP-трафик попадает к бэкендам (и снаружи через VPS, и изнутри сети). Развёрнут LXC в сегменте DMZ, `192.168.40.11`. Держит служебный WireGuard-клиент `wg0` к VPS (второй конец туннеля публикации, см. `07-edge-vps.md`), терминирует TLS, применяет middleware и маршрутизирует запросы на бэкенды в SERVICES и других VLAN.

DMZ по замыслу содержит только Traefik — все чувствительные сервисы стоят в SERVICES, и Traefik ходит к ним по явным firewall-разрешениям. Базлайн LXC (unprivileged + `nesting=1`) — см. `02-conventions.md`.

## 1. Домены и сертификаты

Основной домен — `kvasok.xyz`, wildcard `*.kvasok.xyz` публикует self-hosted сервисы. Снаружи `*.kvasok.xyz` указывает на VPS (см. `07-edge-vps.md`); внутри сети Unbound через split-horizon резолвит те же имена в локальный адрес Traefik (`192.168.40.11`).

Сертификаты Let's Encrypt получает **Traefik через DNS-01 challenge** (wildcard `*.kvasok.xyz`). VPS сертификаты не хранит — он работает на L4. TLS-резолвер по умолчанию для `*.kvasok.xyz` — `timewebcloud`; для отдельных хостов есть `namecheap`. DNS-01 через `namecheap` требует обращения к API Namecheap, недоступному напрямую через провайдера, поэтому исходящий трафик Traefik для этого заворачивается через Xray-прокси (см. раздел 8).

## 2. Конфигурация

Главный конфиг — `/etc/traefik/traefik.yaml`. Секреты и окружение (токен Timewebcloud, ключ Namecheap API, `HTTP_PROXY`/`HTTPS_PROXY` на Xray) — в `/etc/traefik/.env`, подключаемом как `EnvironmentFile` в systemd-юните, в конфиг не попадают. Динамические провайдеры — `/etc/traefik/dynamic/*.yaml` (по файлу на сервис плюс общий `config.yaml` со всеми middleware и TLS-опциями), директория читается с `watch: true`. Access-логи (`/var/log/traefik/access.log`) включены и используются парсером CrowdSec; основной лог (`/var/log/traefik/traefik.log`) пишется с уровнем `WARN`.

Весь `/etc/traefik` — самостоятельный git-репозиторий (`github.com/roman-kvasnikov/homelab-traefik`), изменения коммитятся и пушатся вручную скриптом `git-push.sh`. Секреты и артефакты (`.env`, `acme.json`, `default.crt/key`, `plugins-storage`) в `.gitignore`, в репозиторий не попадают. Это отдельный от Ansible путь версионирования — в отличие от `nftables.conf` (раздел 7), который генерируется и раскатывается Ansible (`16-ansible.md`).

Четыре entrypoint'а: `web` (`:80`) — редирект на `websecure` (301, permanent); реально недостижим извне, так как nftables не открывает 80 порт — весь трафик и так приходит на 443 (напрямую из LAN или через VPS). `websecure` (`:443`, `asDefault: true`) — основной, принимает PROXY protocol только от VPS внутри туннеля (`proxyProtocol.trustedIPs: ["10.0.0.1"]`). `internal` (`192.168.40.11:8079`) — API/дашборд Traefik и виджет Homepage. `metrics` (`192.168.40.11:8081`) — Prometheus (`metrics.prometheus.entryPoint: metrics`). Изменения статической конфигурации (entryPoints) требуют полного рестарта сервиса — `watch: true` относится только к динамическому провайдеру.

**Дашборд.** `https://traefik.kvasok.xyz` (роутер `traefik-dashboard`, entrypoint `websecure`) отдаёт `api@internal` через `chain-admin` + `authelia` — доступ только из MGMT/VPN и только после входа через Authelia. Тот же `api@internal` отдельно висит на entrypoint `internal` (роутер `traefik-api`, `chain-internal`) — этим путём до него достаёт Homepage; правда, физически на 8079 порт по nftables (раздел 7) пускают только сам Homepage, так что запас `chain-internal` на MGMT/TRUSTED/VPN здесь не реализуется — более узкий внешний слой режет раньше. `traefik-ping` (`traefik.kvasok.xyz/ping` → `ping@internal`) — healthcheck-эндпоинт (`ping.manualRouting: true` в статическом конфиге).

## 3. TLS-опции

В `tls.options.default`: `minVersion: VersionTLS12`. Минимум именно 1.2, а не 1.3, потому что WebOS на LG TV не поддерживает TLS 1.3 — Jellyfin на нём при TLS 1.3 падает на handshake без записи в access-log. TLS 1.2 остаётся приемлемым уровнем. `curvePreferences`: X25519, CurveP256, CurveP384, CurveP521. `sniStrict: false` — запрос без совпадающего SNI не отбрасывается, а получает default-сертификат-заглушку `/etc/traefik/certs/default.{crt,key}` (в `tls.stores.default.defaultCertificate`).

## 4. Catch-all роутер

В `config.yaml` определён роутер `catch-all` с `priority: 1` и правилом `HostRegexp(.+) || PathPrefix(/)`. Он ловит всё, что не подошло ни под один другой роутер, прогоняет через `crowdsec` и `allow-deny-all` (allow-list только `127.0.0.1/32`) и отдаёт `noop@internal`. Практический эффект: любой запрос на неизвестный хост получает 403 — нет утечки на default backend, нет шанса случайно открыть внутренний сервис из-за опечатки в конфиге.

## 5. Middleware

Конфигурация middleware разбита на строительные блоки и собранные из них цепочки.

### 5.1. Строительные блоки

| Middleware          | Назначение                                                                                                                                                                                                   |
| :------------------ | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `crowdsec`          | плагин CrowdSec (LAPI `127.0.0.1:8080`, mode `stream`, обновление раз в 60 сек, при недоступности LAPI работает с кэшем)                                                                                     |
| `allow-deny-all`    | ipAllowList только `127.0.0.1/32` — фактически блок всего                                                                                                                                                    |
| `allow-mgmt-ips`    | ipAllowList: MGMT + VPN — только управляющая сеть (см. 5.4)                                                                                                                                                  |
| `allow-trusted-ips` | ipAllowList: MGMT + TRUSTED + VPN — доверенные пользовательские сети (см. 5.4)                                                                                                                               |
| `allow-media-ips`   | ipAllowList: MGMT + TRUSTED + IOT + VPN — доверенные плюс телевизоры (см. 5.4)                                                                                                                               |
| `headers-common`    | contentTypeNosniff, forceSTS, includeSubdomains, STS 180 дней, X-Forwarded-Proto=https — базовый набор для любых HTTP-клиентов, включая API                                                                  |
| `headers-browser`   | referrer policy, permissions policy (запрет camera/microphone/geolocation/USB/Bluetooth), CSP (`frame-ancestors 'self'`, `base-uri 'self'`, `form-action 'self'`) — заголовки только для браузерных клиентов |
| `rate-default`      | average 50, burst 20, period 1s                                                                                                                                                                              |
| `rate-strict`       | average 10, burst 5, period 1s                                                                                                                                                                               |
| `buffering`         | отключение буферизации (лимиты в 0) — для стримов и больших аплоадов                                                                                                                                         |

### 5.2. Цепочки

```yaml
headers-html:
  chain:
    middlewares: [headers-common, headers-browser]

headers-api:
  chain:
    middlewares: [headers-common]

chain-admin:
  chain:
    middlewares: [allow-mgmt-ips, headers-html]

chain-internal:
  chain:
    middlewares: [allow-trusted-ips, headers-html]

chain-internal-api:
  chain:
    middlewares: [allow-trusted-ips, headers-api]

chain-external:
  chain:
    middlewares: [crowdsec, rate-default, headers-html]

chain-external-api:
  chain:
    middlewares: [crowdsec, rate-default, headers-api]

chain-external-strict:
  chain:
    middlewares: [crowdsec, rate-strict, headers-html]

chain-external-unlimited:
  chain:
    middlewares: [crowdsec, headers-html]
```

### 5.3. Назначение цепочек

`chain-admin` — только управляющая сеть (MGMT + VPN). Применяется к админ-интерфейсам инфраструктуры: Traefik dashboard, Proxmox UI, Omada, OPNsense, pgAdmin — доступ к ним только из management, даже TRUSTED не пускается. `chain-internal` — доверенные пользовательские сети (MGMT + TRUSTED + VPN). Для внутренних сервисов, к которым ходят обычные устройства пользователя. `chain-external` — публичные сервисы с обычным rate-limit. `chain-external-strict` — где нужен жёсткий лимит (Vaultwarden). `chain-external-unlimited` — где лимит мешает (Immich: листание галереи даёт множество параллельных запросов, при обычном лимите пользователя банит; Authelia: React SPA грузит десятки JS-чанков параллельно).

Отдельно стоит **media-доступ**: `allow-media-ips` (MGMT + TRUSTED + IOT + VPN) добавляет к доверенным сетям IOT — это нужно, чтобы телевизоры (в IOT) дотягивались до Jellyfin, оставаясь при этом отрезанными от прочих internal-сервисов. То есть IOT пускается только к медиа, а не ко всему internal (`chain-internal` его не включает). Это изолирует телевизоры: медиасервер им доступен, админки и остальные внутренние сервисы — нет.

Многие сервисы дополнительно прикрыты Authelia через middleware `authelia` (forwardAuth), обычно как `chain-internal + authelia` или `chain-external + authelia` (см. `11-authelia.md`). Одно осознанное исключение — Gotify, несовместимый с forward-auth (см. `13-gotify.md`).

### 5.4. Уровни доступа (ipAllowList)

Вместо одного списка доверенных сетей — три уровня, по возрастанию охвата. Сервис получает нужный уровень через соответствующую цепочку.

```yaml
allow-mgmt-ips: # chain-admin
  - 192.168.10.0/24 # MGMT
  - 10.8.0.0/24 # VPN

allow-trusted-ips: # chain-internal
  - 192.168.10.0/24 # MGMT
  - 192.168.30.0/24 # TRUSTED
  - 10.8.0.0/24 # VPN

allow-media-ips: # media (Jellyfin)
  - 192.168.10.0/24 # MGMT
  - 192.168.30.0/24 # TRUSTED
  - 192.168.60.0/24 # IOT (телевизоры)
  - 10.8.0.0/24 # VPN
```

Градация: `allow-mgmt-ips` — самый узкий (только management + VPN), для админок; `allow-trusted-ips` добавляет TRUSTED (пользовательские устройства), для обычных internal-сервисов; `allow-media-ips` добавляет ещё и IOT, только для медиа. IOT (телевизоры) намеренно есть **только** в media-списке — телевизор дотягивается до Jellyfin, но не до админок и прочих internal-сервисов.

Адресация VPN: во всех трёх списках VPN указан как `10.8.0.0/24`. AmneziaWG работает в routed-модели — NAT на LXC снят, на OPNsense добавлен маршрут `10.8.0.0/24` через `192.168.20.11` (INFRA), поэтому VPN-клиент доходит до Traefik под своим адресом `10.8.0.<N>`, а не под INFRA-адресом. За счёт этого списки на `10.8.0.0/24` работают как задумано, и каждому пиру можно назначать доступ пофайрвольно. Модель адресации VPN — в `09-amneziawg.md`.

## 6. Доверие к заголовкам

В CrowdSec-плагине `clientTrustedIPs` — доверенные внутренние сети (MGMT, TRUSTED, IOT, VPN): запросы от них не банятся. `forwardedHeadersTrustedIPs: 10.0.0.1/32` — Traefik доверяет `X-Forwarded-*` заголовкам только от VPS (сосед по wg0). Это не даёт подделать реальный IP клиента через заголовок кому-либо, кроме доверенного VPS.

## 7. Файрвол (nftables)

Traefik — точка входа всего HTTP-трафика, поэтому фильтрация консервативна: whitelist с `policy drop`. В отличие от типовых сервисных LXC (см. `02-conventions.md`), у Traefik nftables специфичен — он терминирует туннель к VPS, принимает 443 из внутренних VLAN и отдаёт метрики Monitoring. Используется одна таблица `inet filter`; NAT на Traefik нет — в routed-модели VPN-трафик доходит под своим адресом `10.8.0.0/24`. Конфиг генерируется и раскатывается Ansible (`16-ansible.md`) — файл на хосте начинается с пометки «Managed by Ansible, do not edit by hand»; define-блок несёт общий для всех хостов инвентарь адресов, ниже показаны только реально используемые в правилах Traefik.

Что разрешено во входящих: loopback и conntrack established/related; базовые ICMP; SSH (22) из MGMT и VPN; HTTPS (443) на `eth0` из внутренних доверенных VLAN — **MGMT, INFRA, TRUSTED, SERVICES, IOT** (через split-horizon клиенты идут на `192.168.40.11`; INFRA добавлен, чтобы AmneziaWG и Xray сами могли достучаться до сервисов через Traefik); HTTPS (443) на `wg0` от VPS (`10.0.0.1`, публичный трафик с PROXY protocol) и VPN-подсети; внутренний Traefik API (8079) только с Homepage (`192.168.20.20`, виджет дашборда); метрики Traefik (8081) и CrowdSec (6060) только с Monitoring LXC (`192.168.50.21`).

```nft
#!/usr/sbin/nft -f

# Managed by Ansible - do not edit by hand.
# Source of truth: inventory vars for traefik + group_vars/all/network.yml

flush ruleset

define VPS_WG_IP     = 10.0.0.1
define VPN_NET       = 10.8.0.0/24

define MGMT_NET      = 192.168.10.0/24
define INFRA_NET     = 192.168.20.0/24
define TRUSTED_NET   = 192.168.30.0/24
define SERVICES_NET  = 192.168.50.0/24
define IOT_NET       = 192.168.60.0/24

define HOMEPAGE_IP   = 192.168.20.20
define MONITORING_IP = 192.168.50.21

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

        # SSH from MGMT_NET and VPN_NET
        tcp dport 22 ip saddr { $MGMT_NET, $VPN_NET } accept

        # HTTPS from internal VLANs via split-horizon DNS (returns 192.168.40.11)
        iifname "eth0" tcp dport 443 ip saddr { $MGMT_NET, $INFRA_NET, $TRUSTED_NET, $SERVICES_NET, $IOT_NET, $VPN_NET } accept

        # HTTPS through wg0 from VPS (10.0.0.1) - public traffic with PROXY-protocol
        iifname "wg0" tcp dport 443 ip saddr $VPS_WG_IP accept

        # Internal Traefik API - only from Homepage
        iifname "eth0" tcp dport 8079 ip saddr $HOMEPAGE_IP accept

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

SSH ужесточён общим drop-in `10-hardening.conf` (см. `02-conventions.md`), аутентификация по ключам, brute-force прикрыт коллекцией CrowdSec `crowdsecurity/sshd`.

## 8. CrowdSec

CrowdSec engine работает на Traefik LXC рядом с Traefik; bouncer подключён как **локальный** плагин Traefik (Yaegi, `localPlugins`).

**Локальный плагин, а не удалённый.** Удалённые плагины Traefik скачиваются с `plugins.traefik.io` при каждом старте, что создавало гонку с готовностью сети на буте: пока исходящая связность не поднята, скачивание падало, и все роутеры с middleware `crowdsec` не поднимались. Перевод bouncer'а в `localPlugins` (плагин лежит на диске LXC) убирает сетевую зависимость при старте целиком — CrowdSec-защита поднимается независимо от внешней сети.

**Заворот исходящего через Xray.** Часть исходящего трафика CrowdSec и Traefik всё же уходит наружу: engine синхронизируется с CAPI (`api.crowdsec.net`), а Traefik для DNS-01 через `namecheap` обращается к API Namecheap — эти хосты не поднимаются через сеть провайдера напрямую. Поэтому `HTTP_PROXY`/`HTTPS_PROXY` на HTTP-порт Xray (`http://192.168.20.12:10809`) прописаны в `crowdsec.service` (CAPI) и `traefik.service` (namecheap ACME). При недоступности Xray не пройдут только CAPI-синхронизация и выпуск namecheap-сертификатов; уже выданные сертификаты, локальный плагин, сценарии и детекты продолжают работать.

### 8.1. Коллекции

`crowdsecurity/appsec-generic-rules`, `crowdsecurity/appsec-virtual-patching`, `crowdsecurity/base-http-scenarios`, `crowdsecurity/http-cve`, `crowdsecurity/linux`, `crowdsecurity/sshd`, `crowdsecurity/traefik`, `crowdsecurity/whitelist-good-actors`.

### 8.2. Whitelist на уровне engine

Доверенные внутренние сети (MGMT, INFRA, TRUSTED, IOT) и VPN-подсеть — их IP не банятся.

### 8.3. Нюансы

LAPI слушает только `127.0.0.1:8080`. Bouncer в режиме `stream` раз в 60 сек (`updateIntervalSeconds: 60`) пуллит из LAPI полный список банов и держит кэш в памяти. `defaultDecisionSeconds: 14400` — длительность решения (4 часа), которую плагин применяет, если сам ответ LAPI её не передаёт. При недоступности LAPI работает с последним кэшем: ранее забаненные остаются, новые не появляются до восстановления engine. Явного fail-closed (`defaultDecision: block`) у плагина нет — при длительном downtime новые атаки не фильтруются; частично страхует `Restart=always` в systemd (рестарт через 60 сек).

### 8.4. Метрики

CrowdSec engine отдаёт Prometheus-метрики на `192.168.40.11:6060` (`prometheus` в `/etc/crowdsec/config.yaml`: `enabled: true`, `level: full`, `listen_addr: 192.168.40.11`, `listen_port: 6060`). Доступ к порту на nftables открыт только Monitoring LXC. Метрики: активные баны по происхождению (свои детекты против community-фида CAPI) и причине, срабатывания сценариев, поток парсинга логов, запросы к LAPI. Скрейпит стек мониторинга (см. `15-monitoring.md`).

## 9. Метрики Traefik

Нативный Prometheus-эндпоинт на отдельном entrypoint `metrics` (`192.168.40.11:8081`). Даёт RPS, коды ответов по классам, латентность p95 по сервисам, трафик. Доступ к порту открыт только Monitoring LXC (nftables). Дашборд Traefik — см. `15-monitoring.md`.

## 10. Резервное копирование

Только PBS-снапшот всего LXC в составе общего ежедневного pve-задания. Критичного point-in-time состояния у Traefik нет; конфиги (`/etc/traefik/`, `/etc/nftables.conf`, `/etc/wireguard/wg0.conf`) маленькие, статичные и восстанавливаются вместе с LXC из PBS. См. `06-backup.md`.

Отдельно от PBS: `/etc/traefik` — собственный git-репозиторий (`github.com/roman-kvasnikov/homelab-traefik`, раздел 2), даёт независимую от PBS историю изменений динамических конфигов и роутеров. Секреты (`.env`, `acme.json`, сертификаты) туда не попадают, при восстановлении из голого git-клона их пришлось бы завести заново.

## 11. Зависимости

- **VPS (`07-edge-vps.md`)** — второй конец wg0-туннеля, источник публичного трафика с PROXY protocol.
- **Unbound на OPNsense** — split-horizon `*.kvasok.xyz → 192.168.40.11`, DNS для DNS-01 ACME.
- **Бэкенды в SERVICES** — Vaultwarden, Authelia, Gotify, Monitoring, медиастек, Immich, Frigate, Shares/FileBrowser, OnlyOffice и прочие сервисы — цели проксирования (доступ по явным firewall-разрешениям).
- **Monitoring LXC (`192.168.50.21`)** — скрейпит метрики Traefik (8081) и CrowdSec (6060).
- **Xray (`192.168.20.12`)** — HTTP-прокси для исходящего трафика Traefik (выпуск сертификатов через `namecheap` DNS-01) и CrowdSec engine (CAPI-синхронизация), через `HTTP_PROXY`/`HTTPS_PROXY` в `traefik.service` и `crowdsec.service`. Bouncer-плагин локальный (`localPlugins`), по сети не качается.
- **PBS (`192.168.10.15`)** — снапшоты LXC.
