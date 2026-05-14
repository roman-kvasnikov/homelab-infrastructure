---
name: Homelab-Infrastructure
description: |
  Полное описание домашней инфраструктуры Романа — Proxmox-based Homelab с внешним
  VPS как entry point, AmneziaWG для клиентского доступа, WireGuard служебный туннель,
  Traefik + CrowdSec в LXC, DockerHost VM с self-hosted сервисами. Используй этот
  контекст для любых вопросов по архитектуре, диагностике или изменениям инфраструктуры.
---

# Домашняя инфраструктура Романа

Этот документ описывает текущее состояние homelab. Используется как контекст для ИИ при обсуждении любых вопросов по сети, сервисам, безопасности и архитектуре.

## 1. Общая архитектура

Инфраструктура построена по схеме **внешний VPS как entry point + домашний homelab за NAT**. Никаких портов на домашнем роутере наружу не проброшено — весь входящий трафик идёт через VPS, а домашний Traefik LXC сам инициирует исходящее WireGuard-соединение к VPS.

```
                       Интернет
                          │
                          ▼
                ┌─────────────────────┐
                │  VPS  77.232.136.6  │
                │  - NGINX (stream)   │
                │  - WireGuard wg0    │
                │  - AmneziaWG awg0   │
                │  - nftables         │
                └──────────┬──────────┘
                           │
                  служебный wg0 туннель
                  (10.0.0.1 ↔ 10.0.0.2)
                           │
                           ▼
                  ┌─────────────────┐
                  │  Traefik LXC    │
                  │  192.168.1.15   │
                  │  wg0 = 10.0.0.2 │
                  └────────┬────────┘
                           │
                       MASQUERADE
                           │
                           ▼
                ┌─────────────────────┐
                │ LAN  192.168.1.0/24 │
                │ Proxmox + LXC + VM  │
                └─────────────────────┘
```

Внешние пиры (телефон, ноут) подключаются по AmneziaWG к VPS, дальше их трафик форвардится в `wg0` к Traefik LXC и через MASQUERADE попадает в LAN. То есть и публичный HTTPS-трафик, и трафик VPN-клиентов входят в дом по одному и тому же служебному `wg0`-туннелю.

## 2. Сетевая топология

Роутер — Keenetic Giga (`192.168.1.1`). Основная LAN — `192.168.1.0/24` (SSID `Izolda-Rally`), для камер выделен отдельный сегмент `192.168.30.0/24`. Между сегментами по умолчанию работает isolate-private, межсегментный трафик открыт только явными permit-правилами на firewall. VPN-подсеть для AmneziaWG-клиентов — **`10.8.0.0/24`**.

### 2.1. Основной сегмент LAN (192.168.1.0/24)

| IP             | Хост                  | Роль                                                       |
|----------------|-----------------------|------------------------------------------------------------|
| 192.168.1.1    | Keenetic Giga         | Роутер, fallback DNS                                       |
| 192.168.1.2    | AdGuard Home (LXC)    | Основной DNS, split-horizon                                |
| 192.168.1.3    | Xray-Proxy (LXC)      | HTTP/SOCKS прокси и DNS-over-Xray для гео-обхода           |
| 192.168.1.10   | Proxmox host          | Гипервизор                                                 |
| 192.168.1.11   | Backup server         | Proxmox Backup Server + Restic snapshots                   |
| 192.168.1.15   | Traefik (LXC)         | Reverse proxy, CrowdSec, WireGuard клиент к VPS            |
| 192.168.1.17   | Vaultwarden (LXC)     | Vaultwarden Password Manager                               |
| 192.168.1.18   | Authelia (LXC)        | IdP, forward-auth, OIDC, SQLite + Redis                    |
| 192.168.1.20   | DockerHost (VM)       | Все основные self-hosted сервисы в Docker, Samba           |
| 192.168.1.40   | Dev (VM)              | VM для тестов и разработки                                 |
| 192.168.1.101  | Huawei Notebook       | Ноутбук на NixOS (host `huawei`)                           |
| 192.168.1.102  | DssMargo PC           | ПК жены                                                    |

### 2.2. Сегмент Cameras (192.168.30.0/24)

Отдельный изолированный сегмент для IP-камер — отдельный SSID `Izolda-Rally-IoT` (2.4 ГГц, WPA2), привязанный к этому сегменту в Keenetic. Роутер сегмента — `192.168.30.1`. DHCP-пул `192.168.30.30–40`, камерам зарезервированы статические адреса по MAC.

| IP             | Хост                  | Роль                                                   |
|----------------|-----------------------|--------------------------------------------------------|
| 192.168.30.30  | Tapo C320WS           | Уличная камера (RTSP)                                  |
| 192.168.30.31  | Tapo C220             | Камера в прихожей (RTSP + ONVIF на 2020)               |

#### Изоляция сегментов

На уровне Keenetic включён `isolate-private` — между сегментами запрещено всё, что не разрешено явно. Поэтому deny-правил на интерфейсе Cameras держать не нужно (и они даже вредны: режут ответные пакеты на разрешённые TCP-соединения, потому что firewall между сегментами в Keenetic не stateful для deny-правил после permit на другом интерфейсе).

#### Permit-правила

Заведены на интерфейсе **«Домашняя сеть»** (так как трафик инициирует DockerHost, сидящий в Home):

| Действие   | Источник       | Назначение         | Порт     | Назначение            |
|------------|----------------|--------------------|----------|-----------------------|
| Разрешить  | 192.168.1.20   | 192.168.30.0/24    | TCP 554  | RTSP для Frigate      |
| Разрешить  | 192.168.1.20   | 192.168.30.0/24    | TCP 2020 | ONVIF для Camera-Hall |

В сегменте Cameras также включён NAT для возможности обновления прошивок камер и работы Tapo-приложения (push-уведомления и т.п.). Если в будущем захочется отрезать камеры от интернета полностью — это делается через «Политику доступа» сегмента, а не через firewall-правила.

## 3. Внешний VPS (77.232.136.6)

Linux-VPS, выступает единственной точкой входа в инфраструктуру. Файрвол — **nftables**. Сертификаты не хранит и TLS не терминирует.

### 3.1. Открытые порты

| Порт         | Назначение                                              |
|--------------|---------------------------------------------------------|
| SSH (нестд.) | SSH, доступ только по ключу                             |
| 443/tcp      | HTTPS, проксируется в `wg0` к Traefik LXC               |
| 51820/udp    | WireGuard `wg0` (служебный туннель к Traefik LXC)       |
| 51821/udp    | AmneziaWG `awg0` для клиентов (телефон, ноут)           |

ACME-валидация делается по DNS-01 на Traefik, в HTTP challenge нет нужды, поэтому порт 80 не открыт.

### 3.2. NGINX (L4 stream)

NGINX работает только в L4-режиме (`stream`). Он не знает про SNI и не терминирует TLS — просто прозрачно перекидывает 443 в WireGuard-туннель с включённым **PROXY protocol**, чтобы Traefik видел реальные IP клиентов.

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 1024;
}

stream {
    log_format proxy '$remote_addr [$time_local] '
                     '$protocol $status $bytes_sent $bytes_received '
                     '$session_time -> $upstream_addr';
    access_log /var/log/nginx/stream.log proxy;

    server {
        listen 443 reuseport;
        proxy_pass 10.0.0.2:443;
        proxy_protocol on;
        proxy_connect_timeout 3s;
        proxy_timeout 300s;
    }
}
```

Соответственно на Traefik LXC entrypoint `websecure` слушает 443 с `proxyProtocol.trustedIPs` на адрес VPS внутри туннеля (`10.0.0.1`).

### 3.3. WireGuard и AmneziaWG

- **`wg0`** — служебный туннель между VPS (`10.0.0.1`) и Traefik LXC (`10.0.0.2`). По нему идёт весь публичный HTTPS-трафик и трафик VPN-клиентов.
- **`awg0`** — AmneziaWG-сервер для пиров (UDP/51821). Обфусцированный протокол для обхода DPI.

На VPS через nftables настроен **forwarding из `awg0` в `wg0`** — пакеты от пиров уходят дальше в служебный туннель к Traefik LXC. На Traefik LXC через nftables (таблица `ip nat`, chain `wg-nat`) сделан **MASQUERADE** — правило добавляется динамически через PostUp при подъёме wg0. Для устройств в LAN трафик VPN-клиентов выглядит как идущий с `192.168.1.15`. Подробности — в подразделе 6.7.

## 4. VPN-доступ к дому (AmneziaWG)

Внешние пиры (телефоны, ноутбуки) подключаются к `awg0` на VPS. Дальше их пакеты форвардятся в `wg0`, попадают на Traefik LXC и через MASQUERADE идут в LAN. Это даёт пирам полный доступ к домашней сети — как будто они физически дома.

### 4.1. Поток трафика

```
[Пир]  ── AWG / UDP 51821 ──>  [VPS]
                                  │
                                  │  nftables: forward awg0 → wg0
                                  ▼
                              [Traefik LXC]  ── nftables MASQUERADE ──>  [LAN]
```

### 4.2. Split-horizon DNS

AdGuard Home (192.168.1.2) настроен так, что внутренние имена `*.kvasok.xyz` резолвятся в LAN-адрес Traefik LXC (`192.168.1.15`), а публичные имена — через рекурсивные апстримы. Для специфичных доменов (`themoviedb.org`, `thetvdb.com`, `tmdb.org`) DNS-запросы идут через Xray на `192.168.1.3:10853` для гео-обхода.

VPN-клиент использует AdGuard как DNS, поэтому split-horizon работает и для него — изнутри туннеля имена резолвятся в LAN-адреса.

## 5. Proxmox host (192.168.1.10)

Гипервизор всего homelab. Здесь же определены параметры виртуализации, storage и реакции на пропадание питания.

### 5.1. Storage

| Storage          | Тип       | Физический носитель           | Назначение                                |
|------------------|-----------|-------------------------------|-------------------------------------------|
| `local`          | dir       | sda (256 GB SSD), pve-root    | ISO, шаблоны LXC, vzdump-дампы            |
| `zguests`        | zfspool   | nvme0n1 (1 TB NVMe)           | Виртуальные диски всех VM/LXC             |
| `pbs-homelab`    | pbs       | сетевой PBS на 192.168.1.11   | Бэкап-цель для VM/LXC и host backup       |

### 5.2. Дисковая структура

```
sda (256 GB SATA SSD):
├── EFI boot partition (1 GB)
├── pve-swap (8 GB)
└── pve-root (225 GB)  → /, и /var/lib/vz (storage `local`)

nvme0n1 (1 TB NVMe):
└── zguests (ZFS pool, lz4, ashift=12, volblocksize=16k)
    ├── system-диски LXC AdGuard, Xray, Traefik, Vaultwarden, Authelia
    ├── system-диск DockerHost VM (100 GB)
    └── system-диски Dev VM, Be-Free.Online VM
```

### 5.3. ZFS-пул zguests

Создан с параметрами `compression=lz4`, `atime=off`, `xattr=sa`, `acltype=posixacl`. **ZFS ARC ограничен 4 GB** (`/etc/modprobe.d/zfs.conf`: `options zfs zfs_arc_max=4294967296`), чтобы не отъедать память у DockerHost VM. Остальная RAM остаётся доступной для гостей.

### 5.4. Лимиты ресурсов гостей

Чтобы избежать борьбы за CPU/RAM между pve и гостями (особенно при старте DockerHost с десятками контейнеров), у VM выставлены разумные лимиты:

| Гость             | RAM (max/min)     | Cores | Startup order/up/down     |
|-------------------|-------------------|-------|---------------------------|
| AdGuard (102)     | 512 MB / 512 swap | 1     | order=1, up=5, down=30    |
| Xray (103)        | 512 MB / 512 swap | 1     | order=1, up=30, down=30   |
| Traefik (115)     | 1 GB / 1 GB swap  | 2     | order=2, up=5, down=30    |
| Vaultwarden (117) | 512 MB / 512 swap | 1     | order=3, up=5, down=30    |
| Authelia (118)    | 512 MB / 512 swap | 1     | order=3, up=10, down=30   |
| DockerHost        | 32G / 24G balloon | 4     | order=10, up=60, down=180 |
| Dev (140)         | 4 GB              | 2     | order=5, up=5, down=60    |
| Be-Free (150)     | 4 GB              | 2     | order=5, up=5, down=60    |

### 5.5. UPS и NUT

На pve-хосте установлен и настроен **NUT** (Network UPS Tools) для корректного обращения с UPS при отключении электричества.

Конфиги: `/etc/nut/` (бэкапится в PBS host backup, см. 12.3).

При срабатывании события `low battery` NUT инициирует graceful shutdown гостей в обратном порядке startup (см. таблицу 5.4): сначала DockerHost (с downtime до 180 с на корректное завершение Docker-контейнеров), последним — AdGuard и Xray. Это даёт максимум шансов корректно сохранить состояние БД и не потерять данные.

## 6. Traefik LXC (192.168.1.15)

Reverse proxy для всех сервисов. Дополнительно держит служебный WireGuard-клиент `wg0` к VPS (`10.0.0.2`).

LXC unprivileged (`unprivileged: 1`) — root внутри маппится в nobody на хосте через user namespaces. Из features включён только `nesting=1` — это нужно не для Docker (его в LXC нет), а для корректной работы systemd 257 в Debian Trixie: современный systemd активно использует user namespaces для собственного sandbox'а юнитов (`PrivateUsers=`, `PrivateTmp=` и т.п.), без `nesting=1` падают journald, tmpfiles-setup и часть служебных юнитов. Эскалации до хоста `nesting=1` в unprivileged-контейнере не даёт. Опасных features (`keyctl`, `mount=`, `mknod`) нет, hostpath-mounts отсутствуют.

### 6.1. Домены и сертификаты

- **kvasok.xyz** — основной домен homelab. Wildcard `*.kvasok.xyz` резолвится на VPS `77.232.136.6`. На нём публикуются self-hosted сервисы.
- Сертификаты Let's Encrypt получает **Traefik через DNS Challenge** (wildcard `*.kvasok.xyz`). На VPS никаких сертификатов нет — он работает только на L4 (stream-проксирование).
- TLS-резолвер по умолчанию для `*.kvasok.xyz` — `timewebcloud`, для отдельных хостов также есть `namecheap`.

### 6.2. Конфигурация

- Главный конфиг: `/etc/traefik/traefik.yaml`.
- Динамические провайдеры: `/etc/traefik/dynamic/*.yaml` (по файлу на сервис + общий `config.yaml` со всеми middleware и TLS-опциями).
- Access logs используется для парсера CrowdSec.
- Entrypoint `websecure` (443) принимает PROXY protocol от VPS (`proxyProtocol.trustedIPs: ["10.0.0.1"]`).

### 6.3. TLS-опции

В `tls.options.default` задано:

- `minVersion: VersionTLS12`
- `curvePreferences`: X25519, P256, P384, P521
- Современный набор cipher suites (ECDHE + GCM/ChaCha20).
- Default-сертификат-заглушка: `/etc/traefik/certs/default.{crt,key}` — отдаётся, если запрос пришёл без подходящего SNI.

### 6.4. Catch-all роутер

В `config.yaml` определён роутер **`catch-all`** с `priority: 1` и правилом `HostRegexp(`.+`) || PathPrefix(`/`)`. Он ловит всё, что не подошло ни под один другой роутер, прогоняет через `crowdsec` и `block-all` (allow-list только `127.0.0.1/32`) и отдаёт `noop@internal`. На практике это значит: **любой запрос на неизвестный хост получает 403** — нет утечки на default backend, нет шансов случайно открыть внутренний сервис из-за опечатки в конфиге.

### 6.5. Middleware

Конфигурация middleware разбита на «строительные блоки» и собранные из них цепочки.

#### Строительные блоки

| Middleware                    | Назначение                                                                  |
|-------------------------------|-----------------------------------------------------------------------------|
| `security-headers-minimal`    | sslRedirect, contentTypeNosniff, stsPreload, X-Forwarded-Proto=https        |
| `security-headers-additional` | XSS filter, forceSTS, includeSubdomains, STS 180 дней, referrer policy      |
| `security-headers-frame-block`| frameDeny, X-Frame-Options=SAMEORIGIN                                       |
| `block-all`                   | ipAllowList только `127.0.0.1/32` — фактически блок всего                   |
| `local`                       | ipAllowList: `192.168.1.0/24` (LAN) и `10.8.0.0/24` (VPN)                   |
| `crowdsec`                    | плагин CrowdSec (LAPI `127.0.0.1:8080`, mode `stream`, update раз в 60 сек, при недоступности LAPI продолжает работать с кэшем) |
| `rate-limit`                  | average 50, burst 20, period 1s                                             |
| `rate-limit-strict`           | average 10, burst 5, period 1s                                              |
| `buffering`                   | отключение буферизации (все лимиты в 0) — для стримов и больших аплоадов    |

#### Цепочки

```yaml
headers-chain:
  chain:
    middlewares:
      - security-headers-minimal
      - security-headers-additional
      - security-headers-frame-block

internal-chain:
  chain:
    middlewares:
      - local
      - headers-chain

external-chain:
  chain:
    middlewares:
      - crowdsec
      - rate-limit
      - headers-chain

external-chain-strict:
  chain:
    middlewares:
      - crowdsec
      - rate-limit-strict
      - headers-chain

external-chain-no-rate-limit:
  chain:
    middlewares:
      - crowdsec
      - headers-chain
```

#### Назначение цепочек

- **`internal-chain`** — только LAN и VPN. Применяется чисто к внутренним сервисам: Traefik dashboard, Proxmox UI, AdGuard, Portainer, pgAdmin, Backrest и т.д.
- **`external-chain`** — публичные сервисы с обычным rate-limit'ом.
- **`external-chain-strict`** — для случаев, где нужен жёсткий лимит на скорость запросов: Vaultwarden.
- **`external-chain-no-rate-limit`** — для случаев, когда лимит мешает. Например в Immich, когда листаешь галерею и идет множество запросов, чтобы при этом пользователя не заблокировало.

Многие сервисы дополнительно прикрыты Authelia через middleware `authelia` (forwardAuth) — обычно как `internal-chain + authelia` или `external-chain + authelia`.

### 6.6. Доверие к заголовкам и trusted IPs

В CrowdSec-плагине:

- `clientTrustedIPs`: `192.168.1.0/24`, `10.8.0.0/24` — запросы от этих источников не банятся (LAN и VPN-клиенты).
- `forwardedHeadersTrustedIPs`: `10.0.0.1/32` — Traefik доверяет `X-Forwarded-*` заголовкам только от VPS (наш wg0-сосед).

### 6.7. Файрвол (nftables)

Traefik LXC — единственная точка входа в LAN для всего внешнего трафика (публичный HTTPS через VPS и трафик VPN-клиентов AmneziaWG), поэтому к фильтрации применяется максимально консервативный whitelist-подход. Файрвол реализован на **nftables**, по аналогии с другими LXC в инфраструктуре. Раньше использовался ufw, но он давал слишком много неявных дефолтов (FORWARD policy ACCEPT, служебные правила для NetBIOS/SSDP/mDNS, дублирование IPv4/IPv6 цепочек) и плохо стыковался с динамическим MASQUERADE из wg0 PostUp.

#### Структура

В ядре активны две таблицы:

- `inet filter` — двухстековая (IPv4 + IPv6) таблица для фильтрации, содержит цепочки `input`, `forward`, `output`. Default policy: `input` — **drop**, `forward` — **drop**, `output` — accept.
- `ip nat` — IPv4-only таблица для MASQUERADE трафика VPN-клиентов в LAN. Содержит цепочку `postrouting` (хук `postrouting srcnat`) и именованную цепочку `wg-nat`, в которую `postrouting` делает `jump`. Сама `wg-nat` пустая в момент загрузки конфига — правило MASQUERADE добавляется в неё динамически через PostUp при подъёме wg0 (см. ниже).

#### Что разрешено в input

| Источник                          | Порт/тип        | Назначение                                                    |
|-----------------------------------|-----------------|---------------------------------------------------------------|
| loopback (`lo`)                   | любой           | внутреннее взаимодействие (CrowdSec LAPI, postfix и т.п.)     |
| любой                             | conntrack established/related | стандартное правило для ответных пакетов                     |
| любой                             | ICMPv4 (4 типа) | destination-unreachable, time-exceeded, parameter-problem, echo-request |
| любой                             | ICMPv6 (10 типов) | то что разрешал ufw + NDP-типы (необходимы для IPv6)        |
| `eth0`                            | UDP 67→68       | DHCP-клиент (eth0 получает IP по DHCP от Keenetic)            |
| LAN (192.168.1.0/24) + VPN (10.8.0.0/24) | TCP 22   | SSH из LAN и через VPN-туннель                                |
| LAN (192.168.1.0/24) на `eth0`    | TCP 443         | HTTPS из LAN (через split-horizon DNS возвращающий 192.168.1.15) |
| VPS (10.0.0.1) + VPN (10.8.0.0/24) на `wg0` | TCP 443 | HTTPS через wg0: публичный трафик от VPS с PROXY-protocol и внутренние запросы VPN-клиентов |

Всё остальное молча отбрасывается default policy.

#### Что разрешено в forward

VPN-клиенты получают **полный доступ к LAN на любые порты любых хостов**: `iifname "wg0" oifname "eth0" ip saddr 10.8.0.0/24 accept`. Это сознательное решение — основная цель AmneziaWG-туннеля состоит в том, чтобы из любой точки мира иметь доступ к домашней сети как будто физически дома, без ограничений по сервисам и портам. Безопасность здесь обеспечивается на уровне самого VPN (только авторизованные пиры подключаются) и на уровне отдельных сервисов внутри LAN (Authelia, локальные firewall'ы у Vaultwarden и Authelia, и т.д.).

Обратное направление (LAN-хосты инициируют соединения к VPN-клиентам) запрещено default policy DROP. Conntrack-правило `established,related accept` пропускает только ответные пакеты на уже установленные соединения.

Дополнительно в forward chain настроены anti-spoofing-правила: пакеты, пришедшие на `wg0` с source-IP вне `10.8.0.0/24` и `10.0.0.1`, дропаются (никто кроме VPN-клиентов и VPS-соседа не может легитимно появиться на этом интерфейсе); зеркально дропаются пакеты с `eth0`, у которых source-IP из `10.8.0.0/24` или `10.0.0.1` (в LAN таких адресов взяться неоткуда). Правила стоят после conntrack-блока, чтобы не ломать обратные пакеты по уже установленным соединениям. Это сохраняет смысл различия «трафик из LAN» vs «трафик из VPN» на уровне ядра — скомпрометированный VPN-клиент не сможет выдать свои пакеты за идущие из LAN, и наоборот.

#### Динамический MASQUERADE через wg0 hooks

VPN-клиенты приходят с source-IP из `10.8.0.0/24`. У LAN-хостов нет маршрута до этой подсети, поэтому при форварде в LAN адреса нужно подменять на адрес Traefik LXC (192.168.1.15) — это делает MASQUERADE.

Правило MASQUERADE добавляется не статически в `/etc/nftables.conf`, а динамически в момент подъёма wg0 — через `PostUp` хук в `/etc/wireguard/wg0.conf`:

```ini
PostUp = nft add rule ip nat wg-nat ip saddr 10.8.0.0/24 oifname eth0 masquerade
PostDown = nft flush chain ip nat wg-nat
```

Это позволяет хранить правило в одном логическом месте с конфигом туннеля и автоматически очищать его при опускании wg0. `nft flush chain` идемпотентен: если wg0 упадёт некорректно и потом поднимется, `flush + add` не оставят дубликатов в `wg-nat`.

Ключевая зависимость в порядке загрузки: при старте LXC сначала `nftables.service` загружает `/etc/nftables.conf` (создаёт пустую chain `wg-nat`), затем `wg-quick@wg0.service` поднимает туннель и через PostUp вставляет правило в уже существующую chain. Если перезагрузить только nftables без переподнятия wg0 — правило MASQUERADE потеряется (chain очистится при `flush ruleset`). В этом случае нужно либо `systemctl restart wg-quick@wg0`, либо вручную выполнить команду из PostUp.

#### Полный конфиг `/etc/nftables.conf`

```nft
#!/usr/sbin/nft -f

flush ruleset

# Сетевые алиасы
define LAN_NET = 192.168.1.0/24
define VPN_NET = 10.8.0.0/24
define VPS_WG  = 10.0.0.1

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;

        # Loopback
        iifname "lo" accept

        # Conntrack
        ct state established,related accept
        ct state invalid drop

        # ICMPv4 — что разрешает текущий ufw
        ip protocol icmp icmp type {
            destination-unreachable,
            time-exceeded,
            parameter-problem,
            echo-request
        } accept

        # ICMPv6 — необходимо для NDP и PMTUD на IPv6
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

        # DHCP-клиент: eth0 получает IP по DHCP от Keenetic
        iifname "eth0" udp sport 67 udp dport 68 accept

        # SSH из LAN и VPN
        tcp dport 22 ip saddr { $LAN_NET, $VPN_NET } accept

        # HTTPS из LAN (split-horizon DNS возвращает 192.168.1.15)
        iifname "eth0" tcp dport 443 ip saddr $LAN_NET accept

        # HTTPS через wg0:
        #   - от VPS (10.0.0.1) — публичный трафик с PROXY-protocol
        #   - от VPN-клиентов (10.8.0.0/24) — внутренние запросы к сервисам
        iifname "wg0" tcp dport 443 ip saddr { $VPS_WG, $VPN_NET } accept
    }

    chain forward {
        type filter hook forward priority filter; policy drop;

        # Conntrack
        ct state established,related accept
        ct state invalid drop

        # Anti-spoofing: пакет на wg0 может иметь source только из VPN или от VPS
        iifname "wg0" ip saddr != { $VPN_NET, $VPS_WG } drop

        # Anti-spoofing: пакет с eth0 не может выдавать себя за VPN-клиента или VPS
        iifname "eth0" ip saddr { $VPN_NET, $VPS_WG } drop

        # VPN-клиенты → LAN: полный доступ без ограничений по портам
        iifname "wg0" oifname "eth0" ip saddr $VPN_NET accept
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        jump wg-nat
    }

    # MASQUERADE правила добавляются динамически из wg0 PostUp
    chain wg-nat {
    }
}
```

#### Резервное копирование

Конфиги `/etc/nftables.conf` и `/etc/wireguard/wg0.conf` бэкапятся в составе общего PBS-снапшота Traefik LXC (см. раздел 12.3). Отдельный restic-бэкап для них не делается — конфиги маленькие, статичные и легко восстанавливаются вместе с LXC из PBS.

### 6.8. SSH-доступ

SSH на Traefik LXC доступен только из LAN (`192.168.1.0/24`) и из VPN (`10.8.0.0/24`) — это ограничено правилом в input chain nftables. Снаружи (через VPS) SSH не выставлен.

Конфигурация sshd ужесточена через drop-in `/etc/ssh/sshd_config.d/10-hardening.conf`:

- `PermitRootLogin prohibit-password` — root логин только по ключу, не по паролю.
- `PasswordAuthentication no` — парольная аутентификация полностью отключена.
- `KbdInteractiveAuthentication no` — закрыт обходной путь через keyboard-interactive.
- `X11Forwarding no` — на headless LXC не нужен.
- `AllowAgentForwarding no` — закрыт типичный путь lateral movement через ssh-agent.
- `AllowTcpForwarding no` — LXC нельзя использовать как туннельный прокси через `ssh -L`.

Drop-in положен отдельным файлом, основной `/etc/ssh/sshd_config` не тронут — изменения переживут апгрейды openssh-server.

Аутентификация — по ed25519-ключам. Защита от brute-force обеспечивается коллекцией CrowdSec `crowdsecurity/sshd` (см. раздел 7).

## 7. CrowdSec

CrowdSec engine крутится на Traefik LXC рядом с Traefik. Bouncer интегрирован как плагин Traefik (Yaegi).

### 7.1. Установленные коллекции

- `crowdsecurity/appsec-generic-rules`
- `crowdsecurity/appsec-virtual-patching`
- `crowdsecurity/base-http-scenarios`
- `crowdsecurity/http-cve`
- `crowdsecurity/linux`
- `crowdsecurity/sshd`
- `crowdsecurity/traefik`
- `crowdsecurity/whitelist-good-actors`

### 7.2. Whitelist на уровне engine

- `192.168.1.0/24` (LAN)
- `10.8.0.0/24` (VPN)

### 7.3. Известные нюансы

- LAPI слушает только `127.0.0.1:8080`.
- Bouncer работает в режиме `stream` — раз в 60 секунд пуллит из LAPI полный список банов и держит локальный кэш в памяти. При недоступности LAPI плагин продолжает работать с последней версией кэша: ранее забаненные IP остаются забаненными, новые баны не появляются до восстановления engine'а. Это сдвигает поведение от «при падении LAPI защита почти мгновенно деградирует к fail-open» к «защита замораживается на последнем состоянии». Архитектурно у этого плагина (`maxlerebourg/crowdsec-bouncer-traefik-plugin`) явного `defaultDecision: block` (fail-closed) режима нет — при достаточно длительном downtime engine'а новые атаки фильтроваться не будут. Защиту от этого даёт `Restart=always` в systemd-юните crowdsec.service (рестарт через 60 сек после падения).

## 8. Vaultwarden LXC (192.168.1.17)

Self-hosted password manager, находится в отдельном LXC для изоляции от соседних сервисов. Работает нативно (без Docker) на бинарнике, извлечённом из официального Docker-образа `vaultwarden/server`. База данных — SQLite.

Версия Vaultwarden зафиксирована, обновление вручную через replace бинарника + перезапуск systemd-сервиса (см. ниже).

### 8.1 Файловая структура

```
/usr/local/bin/vaultwarden            # бинарь, извлечён из официального Docker-образа
/usr/share/vaultwarden/web-vault/     # статика веб-интерфейса
/etc/vaultwarden/.env                 # конфигурация (env-переменные)
/etc/systemd/system/vaultwarden.service
/var/lib/vaultwarden/                 # домашняя директория системного юзера vaultwarden
├── data/                             # данные (БД, attachments, ключи)
│   ├── db.sqlite3                    # основная БД (SQLite в WAL-режиме)
│   ├── attachments/                  # вложения к записям
│   ├── rsa_key.pem                   # приватный ключ для подписи JWT
│   └── icon_cache/                   # кэш иконок сайтов (в бэкап не попадает)
├── .restic-password                  # encryption-пароль restic-репозитория (chmod 600)
└── .restic-http-password             # HTTP basic-auth пароль для rest-server (chmod 600)
```

Бинарь и web-vault извлекаются из официального Docker-образа через `skopeo` + `umoci` — это даёт ровно те же файлы, что использует Docker, но без Docker-обёртки. Для совместимости с динамической линковкой бинарника установлены `libmariadb3` и `libpq5`, хотя из СУБД используется только SQLite — Vaultwarden слинкован сразу со всеми тремя driver-библиотеками.

### 8.2 Systemd

Сервис `vaultwarden.service` запускает бинарник от системного юзера `vaultwarden` (UID 999). Включён sandbox-набор: `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `PrivateDevices`, `ProtectKernel*`, `RestrictNamespaces`, `LockPersonality`. Запись разрешена только в `/var/lib/vaultwarden` через `ReadWritePaths`. Перезапуск при падении — `Restart=always` с задержкой 10 секунд.

Сервис автозапускается при старте контейнера (`enabled`).

### 8.3 Конфигурация (`/etc/vaultwarden/.env`)

Ключевые параметры:

- `DATA_FOLDER=/var/lib/vaultwarden/data` — куда писать БД и attachments.
- `DATABASE_URL=/var/lib/vaultwarden/data/db.sqlite3` — путь к SQLite-файлу. Postgres не используется.
- `WEB_VAULT_FOLDER=/usr/share/vaultwarden/web-vault` — статика.
- `ROCKET_ADDRESS=192.168.1.17`, `ROCKET_PORT=8000` — слушает только на конкретном LAN-IP, не на `0.0.0.0`. Loopback и другие интерфейсы недоступны.
- `DOMAIN=https://vaultwarden.kvasok.xyz` — публичный URL, важен для CORS, WebAuthn (RPID), email-ссылок и push-нотификаций.
- `SIGNUPS_ALLOWED=false`, `INVITATIONS_ALLOWED=false` — регистрация и приглашения новых пользователей закрыты. Включаются временно только для разовых нужд (миграции, добавления пользователей).
- `IP_HEADER=X-Forwarded-For` — Vaultwarden читает реальный IP клиента из этого заголовка от Traefik. Безопасно благодаря nftables-фильтру (см. ниже): X-Forwarded-For может прислать только Traefik с 192.168.1.15.
- `ENABLE_WEBSOCKET=true` — для real-time sync между клиентами.
- `ADMIN_TOKEN` не задан — админка `/admin` отключена. Управление через прямой доступ к `.env` и БД, без UI.
- `LOG_LEVEL=warn`, `EXTENDED_LOGGING=true` — логи через journald (`journalctl -u vaultwarden`).

### 8.4 Сетевая фильтрация (nftables)

Vaultwarden принимает входящие соединения на `192.168.1.17:8000` **только** с адреса Traefik (192.168.1.15). Все остальные источники в LAN и из VPN получают `drop` на пакетах — соединение не устанавливается, отдаётся timeout. Это закрывает прямой доступ к Vaultwarden из LAN/VPN, минуя Traefik с его middleware-цепочкой (CrowdSec, rate-limit, security headers).

Конфиг в `/etc/nftables.conf`:

```
table inet filter {
    chain input {
        type filter hook input priority filter; policy accept;
        tcp dport 8000 ip saddr 192.168.1.15 accept
        tcp dport 8000 drop
    }
    chain forward { type filter hook forward priority filter; policy accept; }
    chain output  { type filter hook output  priority filter; policy accept; }
}
```

Глобальные политики оставлены `accept` — фильтр точечный, ограничивает только порт 8000. SSH (22), ICMP, исходящий трафик и established connections не затронуты. Конфиг загружается через `nftables.service` при старте контейнера.

### 8.5 Резервное копирование

Vaultwarden бэкапится **двумя независимыми механизмами**:

**1. PBS-снапшот всего LXC** — ежедневно в составе общего pve-задания. Закрывает сценарий «снёс LXC целиком, нужно восстановить за минуту». Снапшоты лежат на backup-сервере в `/mnt/data/pbs/datastore/ct/117/`.

**2. Restic-снапшот данных** — ежедневно через скрипт `/usr/local/sbin/vaultwarden-backup.sh`, запускаемый systemd-таймером `vaultwarden-backup.timer`. Закрывает сценарий «нужна конкретная версия БД от N дней назад» и даёт гранулярное восстановление.

Скрипт делает два шага:

1. Online-снапшот SQLite через `sqlite3 .backup` — это атомарная копия БД, безопасная для работающего Vaultwarden (не блокирует и не повреждает живой файл).
2. `restic backup` всей `/var/lib/vaultwarden/data/`, исключая живой `db.sqlite3`, WAL-файлы (`-shm`, `-wal`), `icon_cache/` и `tmp/`. В бэкап попадают: `db.sqlite3.backup` (consistent copy), `attachments/`, `rsa_key.pem`. Тег `vaultwarden`, host `vaultwarden`. Транспорт — rest-server на бэкап-сервере (`rest:http://vaultwarden:...@192.168.1.11:8000/vaultwarden/`) в режиме append-only.

Retention-блок выполняется централизованно на бэкап-сервере (см. раздел 12.1.4), потому что rest-server в append-only режиме блокирует операции удаления со стороны клиента.

**Восстановление БД**: после `restic restore` файл лежит как `db.sqlite3.backup` — нужно переименовать в `db.sqlite3` перед запуском Vaultwarden. WAL-файлы (`-shm`, `-wal`) при восстановлении не нужны и автоматически создадутся при первом старте.

### 8.6 Restic-репозиторий и подключение к бэкап-серверу

Vaultwarden бэкапится в отдельный restic-репозиторий `/mnt/data/restic/vaultwarden/` на бэкап-сервере (192.168.1.11). Это даёт изоляцию: компрометация Vaultwarden LXC не позволяет атакующему добраться до бэкапов других сервисов, потому что rest-server в режиме `--private-repos` показывает каждому юзеру только его собственный репо без доступа к чужим.

Подключение к бэкап-серверу идёт через rest-server по HTTP. URL репо: `rest:http://vaultwarden:<пароль>@192.168.1.11:8000/vaultwarden/`. HTTP basic-auth пароль хранится в `/var/lib/vaultwarden/.restic-http-password` (chmod 600, owner `vaultwarden:vaultwarden`).

Encryption-пароль репо (для шифрования содержимого данных) хранится в `/var/lib/vaultwarden/.restic-password` — это отдельный механизм. Один пароль — транспортная аутентификация в rest-server, другой — шифрование содержимого внутри репо. Encryption-пароль также продублирован на бэкап-сервере в `/etc/restic-retention/vaultwarden.password` для retention-операций.

Append-only свойство обеспечивается на стороне rest-server (см. раздел 12.1.3). Скомпрометированный Vaultwarden может только писать новые снапшоты — DELETE-операции возвращают 403. Retention применяется централизованно из единого скрипта на бэкап-сервере (см. раздел 12.1.4).

### 8.7 Обновление Vaultwarden

Образ обновляется заменой бинаря и web-vault, без перекомпиляции. Skopeo и umoci оставлены установленными для повторного использования.

Процедура (на примере выхода версии X.Y.Z):

```bash
cd /tmp
skopeo copy docker://docker.io/vaultwarden/server:X.Y.Z oci:vaultwarden-image:X.Y.Z
umoci unpack --rootless --image vaultwarden-image:X.Y.Z vaultwarden-new

systemctl stop vaultwarden

# backup of current version (rollback if needed)
cp /usr/local/bin/vaultwarden /usr/local/bin/vaultwarden.bak
cp -r /usr/share/vaultwarden/web-vault /usr/share/vaultwarden/web-vault.bak

install -o root -g root -m 755 /tmp/vaultwarden-new/rootfs/vaultwarden /usr/local/bin/vaultwarden
rm -rf /usr/share/vaultwarden/web-vault
cp -r /tmp/vaultwarden-new/rootfs/web-vault /usr/share/vaultwarden/

systemctl start vaultwarden
systemctl status vaultwarden
```

После нескольких дней успешной работы — удалить `.bak` и временные директории в `/tmp/`.

### 8.8 Зависимости от внешних сервисов

- **Traefik LXC (192.168.1.15)** — единственный разрешённый источник запросов к 8000 (через nftables). Без Traefik сервис недоступен снаружи LXC.
- **AdGuard Home (192.168.1.2)** — DNS, в том числе для split-horizon `vaultwarden.kvasok.xyz` → 192.168.1.15.
- **Backup-сервер (192.168.1.11)** — для restic-бэкапов и PBS-снапшотов.

## 9. Authelia LXC (192.168.1.18)

Self-hosted IdP (Identity Provider) и forward-auth прокси для Traefik. Вынесен в отдельный LXC для изоляции. Authelia хранит хеши паролей пользователей, TOTP-секреты, WebAuthn credentials, OIDC client secrets и signing keys, поэтому компрометация Authelia = компрометация всех аутентификаций ко всем сервисам инфраструктуры.

Работает нативно (без Docker) на бинарнике из официальных GitHub releases. База данных — SQLite. Sessions — локальный Redis в том же LXC.

Версия Authelia зафиксирована, обновление вручную через replace бинарника + перезапуск systemd-сервиса (см. ниже).

### 9.1 Файловая структура

```
/usr/local/bin/authelia                # бинарь, скачан с github.com/authelia/authelia/releases
/etc/authelia/                         # конфиги и секреты
├── configuration.yaml                 # головной конфиг (server, log, notifier, ntp)
├── storage.yaml                       # SQLite path
├── session.yaml                       # cookies + Redis connection
├── authentication-backend.yaml        # file backend + argon2 параметры
├── access-control.yaml                # default policy + rules
├── identity-providers.yaml            # OIDC clients и signing keys
├── regulation.yaml                    # rate limiting на неудачные логины
├── users.yaml                         # пользователи с argon2-хешами паролей
├── .env                               # env-переменные для systemd
└── secrets/
    ├── JWT_SECRET                     # подпись reset-password JWT
    ├── SESSION_SECRET                 # подпись session cookies
    ├── REDIS_PASSWORD                 # пароль локального Redis
    ├── STORAGE_ENCRYPTION_KEY         # шифрование чувствительных полей в БД
    ├── OIDC_HMAC_SECRET               # HMAC для OIDC
    └── oidc/jwks/rsa.2048.key         # RSA-ключ для подписи OIDC tokens

/etc/systemd/system/authelia.service
/var/lib/authelia/                     # домашняя директория системного юзера authelia
├── db.sqlite3                         # основная БД (SQLite, хранит TOTP, WebAuthn, OIDC consents)
├── notifier.log                       # системные уведомления (используется как fallback notifier)
├── .restic-password                   # encryption-пароль restic-репозитория (chmod 600)
├── .restic-http-password              # HTTP basic-auth пароль для rest-server (chmod 600)
└── .cache/restic/                     # кеш restic, в бэкап не включается
```

Бинарь Authelia скачивается напрямую с GitHub releases — это обычный Go-бинарь со всеми зависимостями статически вкомпилированными, никаких рантайм-библиотек ставить не нужно. В отличие от Vaultwarden, где разработчики не публикуют pre-built бинари и приходится извлекать из Docker-образа через skopeo+umoci, здесь всё проще.

### 9.2 Systemd

Сервис `authelia.service` запускает бинарник от системного юзера `authelia` (создан через `adduser --system --no-create-home`). Включён sandbox-набор: `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `PrivateDevices`, `PrivateUsers`, `ProtectKernel*`, `RestrictNamespaces`, `LockPersonality`, `CapabilityBoundingSet=` (пустой), `SystemCallFilter=@system-service`. Запись разрешена только в `/var/lib/authelia` через `ReadWritePaths`. Перезапуск при падении — `Restart=always` с задержкой 10 секунд. Зависимость от Redis: `Requires=redis-server.service` + `After=redis-server.service` гарантирует что Redis стартует первым.

ExecStart передаёт **семь** конфигурационных файлов через повторяющийся `--config` (Authelia мерджит их в один документ при старте). Это позволяет логически разделить конфиг по областям и проще ревьювить изменения.

Сервис автозапускается при старте контейнера (`enabled`).

### 9.3 Конфигурация секретов через env-переменные

Все секреты хранятся в отдельных файлах в `/etc/authelia/secrets/` и подгружаются через переменные окружения вида `AUTHELIA_*_FILE` — Authelia сама прочтёт файл по указанному пути. Env-переменные собраны в файле `/etc/authelia/.env`, который подключается в systemd через `EnvironmentFile=`:

```
AUTHELIA_SERVER_DISABLE_HEALTHCHECK=true
X_AUTHELIA_CONFIG_FILTERS=template
AUTHELIA_IDENTITY_VALIDATION_RESET_PASSWORD_JWT_SECRET_FILE=/etc/authelia/secrets/JWT_SECRET
AUTHELIA_SESSION_SECRET_FILE=/etc/authelia/secrets/SESSION_SECRET
AUTHELIA_SESSION_REDIS_PASSWORD_FILE=/etc/authelia/secrets/REDIS_PASSWORD
AUTHELIA_STORAGE_ENCRYPTION_KEY_FILE=/etc/authelia/secrets/STORAGE_ENCRYPTION_KEY
```

`X_AUTHELIA_CONFIG_FILTERS=template` включает Go-template обработку YAML — это нужно для `identity-providers.yaml`, в котором RSA-ключ для OIDC подгружается через `{{ secret "/etc/authelia/secrets/oidc/jwks/rsa.2048.key" | mindent 10 "|" | msquote }}`. Без этого фильтра шаблон-конструкции воспринимаются как невалидный YAML.

### 9.4 Ключевые параметры конфига

`server.address: tcp://192.168.1.18:9091/` — bind на конкретный LAN-IP, не на `0.0.0.0`, по аналогии с Vaultwarden.

`storage.local.path: /var/lib/authelia/db.sqlite3` — SQLite, без PostgreSQL.

`session.redis: { host: 127.0.0.1, port: 6379, database_index: 0 }` — локальный Redis в том же LXC. Пароль Redis подгружается из `REDIS_PASSWORD_FILE` через env. Sessions сохраняются между рестартами Authelia.

`authentication_backend.file.password.argon2`: `iterations: 3, parallelism: 4, memory: 65536` (64 MiB) — OWASP recommended параметры.

`notifier.filesystem.filename: /var/lib/authelia/notifier.log` — fallback-нотификатор (на случай если SMTP не настроен), пишет письма в файл.

`log.keep_stdout: true` — логи application-уровня уходят в journald (`journalctl -u authelia`). Файлового логирования нет — `ProtectSystem=strict` запрещает запись в `/var/log`, и это правильно.

### 9.5 Локальный Redis

Redis установлен в этом же LXC, не разделяется с другими сервисами. Конфигурация в `/etc/redis/redis.conf`:

- `bind 127.0.0.1 -::1` — только loopback, наружу не слушает
- `requirepass` — пароль из `/etc/authelia/secrets/REDIS_PASSWORD` (генерится через `openssl rand -base64 48`)
- `maxmemory 64mb` + `maxmemory-policy allkeys-lru` — ограничение памяти и LRU-вытеснение

Redis нужен только для хранения активных Authelia-сессий. Никакой другой сервис к этому Redis не подключается. При потере данных Redis (рестарт Redis или LXC) пользователи перелогинятся, но БД с TOTP/WebAuthn/OIDC-данными останется нетронутой — она в SQLite.

### 9.6 Сетевая фильтрация (nftables)

Authelia принимает входящие соединения на `192.168.1.18:9091` **только** с адреса Traefik (192.168.1.15). Все остальные источники в LAN и из VPN получают `drop` — соединение не устанавливается, отдаётся timeout. Это закрывает прямой доступ к Authelia из LAN/VPN, минуя Traefik с его middleware-цепочкой (CrowdSec, rate-limit, security headers).

Конфиг в `/etc/nftables.conf`:

```
table inet filter {
    chain input {
        type filter hook input priority filter; policy accept;
        tcp dport 9091 ip saddr 192.168.1.15 accept
        tcp dport 9091 drop
    }
    chain forward { type filter hook forward priority filter; policy accept; }
    chain output  { type filter hook output  priority filter; policy accept; }
}
```

Глобальные политики `accept` — фильтр точечный, ограничивает только порт 9091. SSH (22), ICMP, Redis (доступен только на loopback по своему bind) не затронуты.

### 9.7 Резервное копирование

Authelia бэкапится **двумя независимыми механизмами**, по той же схеме что и Vaultwarden:

**1. PBS-снапшот всего LXC** — ежедневно в составе общего pve-задания. Закрывает сценарий «снёс LXC целиком, нужно восстановить за минуту».

**2. Restic-снапшот данных и конфигов** — ежедневно через скрипт `/usr/local/sbin/authelia-backup.sh`, запускаемый systemd-таймером `authelia-backup.timer`. Закрывает сценарий «нужна конкретная версия БД от N дней назад» и даёт гранулярное восстановление.

Скрипт делает два шага:

1. Online-снапшот SQLite через `sqlite3 .backup` — атомарная копия БД, безопасная для работающего Authelia.
2. `restic backup` директорий `/var/lib/authelia/` и `/etc/authelia/`, исключая живой `db.sqlite3`, WAL-файлы (`-shm`, `-wal`), `.restic-password`, `.restic-http-password` (chicken-and-egg) и `.cache/`. В бэкап попадают: `db.sqlite3.backup` (consistent copy), `notifier.log`, все YAML-конфиги, секреты в `/etc/authelia/secrets/`. Тег `authelia`, host `authelia`. Транспорт — rest-server (`rest:http://authelia:...@192.168.1.11:8000/authelia/`) в режиме append-only.

Retention-блок из скрипта убран — выполняется централизованно на бэкап-сервере (см. раздел 12.1.4).

**Восстановление БД**: после `restic restore` файл лежит как `db.sqlite3.backup` — нужно переименовать в `db.sqlite3` перед запуском Authelia. WAL-файлы не нужны и автоматически пересоздадутся при первом старте.

### 9.8 Restic-репозиторий и подключение к бэкап-серверу

Authelia бэкапится в отдельный restic-репозиторий `/mnt/data/restic/authelia/` на бэкап-сервере (192.168.1.11). Изоляция между сервисами обеспечивается режимом `--private-repos` на rest-server.

Подключение к бэкап-серверу идёт через rest-server по HTTP. URL репо: `rest:http://authelia:<пароль>@192.168.1.11:8000/authelia/`. HTTP basic-auth пароль хранится в `/var/lib/authelia/.restic-http-password` (chmod 600, owner `authelia:authelia`).

Encryption-пароль репо хранится в `/var/lib/authelia/.restic-password` и продублирован на бэкап-сервере в `/etc/restic-retention/authelia.password`.

Append-only режим rest-server и централизованная retention на бэкап-сервере — см. разделы 12.1.3 и 12.1.4

### 9.9 Обновление Authelia

Бинарь скачивается с GitHub releases напрямую, без Docker. Процедура (на примере выхода версии vX.Y.Z):

```bash
cd /tmp
wget https://github.com/authelia/authelia/releases/download/vX.Y.Z/authelia-vX.Y.Z-linux-amd64.tar.gz
tar -xzf authelia-vX.Y.Z-linux-amd64.tar.gz

systemctl stop authelia

# backup of current version (rollback if needed)
cp /usr/local/bin/authelia /usr/local/bin/authelia.bak

install -o root -g root -m 755 authelia-linux-amd64 /usr/local/bin/authelia

systemctl start authelia
systemctl status authelia
journalctl -u authelia -n 50 --no-pager
```

После нескольких дней успешной работы — удалить `.bak` и временные файлы из `/tmp/`. При мажорных обновлениях (изменение схемы БД) Authelia автоматически выполнит миграцию SQLite при старте — это видно в логах строкой `Storage schema migration from N to M is complete`.

### 9.10 Зависимости от внешних сервисов

- **Traefik LXC (192.168.1.15)** — единственный разрешённый источник запросов к 9091 (через nftables). Без Traefik Authelia недоступна снаружи LXC.
- **AdGuard Home (192.168.1.2)** — DNS, в том числе для split-horizon `authelia.kvasok.xyz` → 192.168.1.15.
- **Backup-сервер (192.168.1.11)** — для restic-бэкапов и PBS-снапшотов.

## 10. DockerHost VM (192.168.1.20)

Ubuntu VM, основной хост для всех Docker-сервисов. ZFS делается на уровне VM (disk passthrough из Proxmox), чтобы корректно работали hardlinks для *-arr / qBittorrent / Jellyfin.

### 10.1. ZFS-датасеты

| Pool       | Mountpoint           | Назначение                           |
|------------|----------------------|--------------------------------------|
| `zdata`    | `/mnt/data`          | AppData всех приложений              |
| `zmedia`   | `/mnt/media`         | Медиатека Jellyfin                   |
| `zfrigate` | `/var/lib/frigate`   | Записи Frigate                       |

### 10.2. Docker networking

Default address pool в `/etc/docker/daemon.json` ограничен диапазоном `10.200.0.0/16` (size `/24`), чтобы Docker автоматически создаваемые сети не пересекались с физическими подсетями LAN/Cameras/VPN:

```json
{
  "default-address-pools": [
    {"base": "10.200.0.0/16", "size": 24}
  ]
}
```

### 10.3. Структура AppData

```
/mnt/data/AppData/
├── Anchor/
├── Immich/
├── Paperless/
├── Portainer/
├── Postgresql/         # общий PostgreSQL
├── Postgresql-Immich/  # отдельный PostgreSQL для Immich
├── Speedtest-Tracker/
└── ...
```

### 10.4. Контейнеры

Основные сервисы (не исчерпывающий список): immich (server + ML + redis + postgres), общий postgres + pgadmin, paperless, baikal (CardDAV/CalDAV), jellyfin, sonarr, radarr, qbittorrent, jellyseerr, frigate, filebrowser quantum, onlyoffice, mealie, portainer. Watchtower не используется — обновления вручную через `docker compose pull`.

#### Особенности

- **PostgreSQL** общий для большинства приложений. Для Immich — отдельный, потому что он требует специфичный образ с `pgvector/vectorchord`.
- **Docker network** `postgres` (`external: true`) — общая сеть для доступа к общему PostgreSQL.
- **Jellyfin / Radarr / Sonarr / Jellyseerr** настроены ходить в интернет через Xray-Proxy (`HTTP_PROXY=http://192.168.1.3:10809`).
- **Frigate** ходит на камеры в сегмент 192.168.30.0/24 по RTSP (порт 554) и ONVIF (порт 2020 для Camera-Hall). Доступ открыт permit-правилами Keenetic на интерфейсе «Домашняя сеть».

### 10.5. Samba

На DockerHost (не в контейнере) поднят Samba-сервер. Слушает только на **445/tcp**. Доступ с ноутбука (`192.168.1.101`) и ПК (`192.168.1.102`). Шары используют ZFS-пулы напрямую.

## 11. Аутентификация

Основной IdP — **Authelia**, развёрнут в отдельном LXC (`192.168.1.18`, см. раздел про **Authelia LXC**). Конфигурация разнесена по семи YAML-файлам в `/etc/authelia/`. 2FA через TOTP и WebAuthn, поддержка PKCE для мобильных приложений. Forward-auth делается middleware `authelia` в Traefik (`http://192.168.1.18:9091/api/authz/forward-auth`) — любой защищённый сервис получает `internal-chain + authelia` или `external-chain + authelia`.

## 12. Резервное копирование

Бэкапы организованы по **трехуровневой схеме**, где каждый уровень закрывает свою задачу и использует подходящий для неё инструмент. Все бэкапы складываются на один **бэкап-сервер 192.168.1.11**, но в разные структуры на одном RAID0 массиве.

### 12.1. Бэкап-сервер (192.168.1.11)

ОС — **Proxmox Backup Server** (поверх Debian 12). Помимо самого PBS, на сервере поднят `rest-server` для приёма restic-бэкапов от Vaultwarden, Authelia и DockerHost (см. раздел 12.1.3).

Дисковая структура:

```
sda (465 GB)           — системный SSD, ext4, ОС и PBS
sdb + sdc (2 × 2.7 TB) — mdadm RAID0 (md0), ext4, mountpoint /mnt/data
```

Структура `/mnt/data`:

```
/mnt/data/
├── pbs/
│   └── datastore/        # PBS datastore "Homelab"
│       ├── .chunks/      # дедуплицированные chunks
│       ├── ct/           # снапшоты LXC
│       ├── vm/           # снапшоты VM
│       └── host/         # host-backup'ы pve
└── restic/
    ├── vaultwarden/      # restic-репо для бэкапов с Vaultwarden LXC
    ├── authelia/         # restic-репо для бэкапов с Authelia LXC
    └── dockerhost/       # restic-репо для бэкапов с DockerHost VM
```

#### 12.1.1. Identity model PBS

Доступ к PBS со стороны pve и других интеграций реализован через разделение identity по назначению. На PBS есть два user-account, каждый со своими токенами и узко-определёнными правами. Это даёт два свойства: компрометация одного канала не затрагивает прав другого, и легко добавить новые интеграции — достаточно создать новый токен под уже существующим user.

Текущее состояние ACL:

| ugid                       | path                  | propagate | roleid            |
|----------------------------|-----------------------|-----------|-------------------|
| `monitoring@pbs`           | `/`                   | 1         | `Audit`           |
| `monitoring@pbs!homepage`  | `/`                   | 1         | `Audit`           |
| `pve-backup@pbs`           | `/datastore/Homelab`  | 1         | `DatastoreBackup` |
| `pve-backup@pbs!pve-host`  | `/datastore/Homelab`  | 1         | `DatastoreBackup` |
| `pve-backup@pbs!pve-main`  | `/datastore/Homelab`  | 1         | `DatastoreBackup` |

User-account `pve-backup@pbs` отвечает только за запись бэкапов. Под ним два токена: `pve-main` для бэкапов VM/LXC через pve storage (см. раздел 12.3), и `pve-host` для host backup конфигов pve (см. раздел 12.2). User-account `monitoring@pbs` отвечает только за read-only доступ для интеграций мониторинга. Под ним сейчас один токен — `homepage`, для виджета PBS в homepage dashboard. По мере добавления новых интеграций (Grafana, Uptime Kuma и т.п.) под этим же user создаются дополнительные токены вида `monitoring@pbs!<integration>` с той же ролью `Audit`.

**Архитектурно важная особенность PBS — privilege separation.** У всех токенов включён флаг `privsep=1` (дефолт). Это означает что эффективные права токена равны пересечению прав user и прав токена. User — это верхняя граница прав, токен может только сужать. Если у user нет прав на путь, никакой токен под ним не получит больше. Поэтому user-level ACL в таблице выше обязательны, а не избыточны: без них токены оказываются с пустыми эффективными правами и операции возвращают `permission check failed`. Это сознательное security-решение PBS, чтобы токен случайно не получил больше прав чем user.

**Свойство append-only обеспечивается тем, что роль `DatastoreBackup` не включает права `Datastore.Modify` и `Datastore.Prune`.** Атакующий, получивший токен `pve-main` через компрометацию pve, может только создавать новые снапшоты. Удаление существующих, изменение, prune — PBS возвращает 403 на уровне API. Это касается и человека-админа: чтобы удалить снапшот через сетевой токен, нужно повысить роль на самом PBS (что является осознанным локальным действием, а не результатом компрометации сетевого клиента).

Retention выполняется только локально на PBS через Prune Jobs от имени root@pam — это не сетевой путь, недоступен ни одному клиентскому токену. Garbage Collection и Verify Jobs работают так же — локально на PBS.

#### 12.1.2. SSH hardening на PBS

Конфигурация sshd ужесточена через drop-in `/etc/ssh/sshd_config.d/10-hardening.conf`:

- `PermitRootLogin prohibit-password` — root логин только по ключу, не по паролю.
- `PasswordAuthentication no` — парольная аутентификация полностью отключена.
- `KbdInteractiveAuthentication no` — закрыт обходной путь через keyboard-interactive.
- `X11Forwarding no` — на headless LXC не нужен.
- `AllowAgentForwarding no` — закрыт типичный путь lateral movement через ssh-agent.
- `AllowTcpForwarding no` — LXC нельзя использовать как туннельный прокси через `ssh -L`.

Drop-in положен отдельным файлом, основной `/etc/ssh/sshd_config` не тронут — изменения переживут апгрейды openssh-server.

#### 12.1.3. rest-server для restic-бэкапов

На бэкап-сервере поднят `rest-server` — HTTP-сервис для restic-клиентов, выступающий приёмником бэкапов от Vaultwarden, Authelia и DockerHost. Доступ ограничен HTTP API с режимом `--append-only` — клиент может писать новые снапшоты, но физически не может удалять или изменять существующие. DELETE-запросы возвращают 403 на уровне сервиса.

**Системный юзер:** `rest-server` (UID 106), отдельная группа `rest-server` (GID 988). Shell — `/usr/sbin/nologin`, никакого интерактивного доступа.

**Файловая структура:**

```
/usr/local/bin/rest-server          # бинарь, скачан с github.com/restic/rest-server
/etc/rest-server/.htpasswd          # HTTP basic-auth credentials (bcrypt)
/etc/systemd/system/rest-server.service
/var/log/rest-server/access.log     # access log запросов
/mnt/data/restic/                   # каталоги репо, owner rest-server:rest-server
├── vaultwarden/
├── authelia/
└── dockerhost/
```

**Конфигурация rest-server** через ExecStart в systemd-юните:

- `--listen 192.168.1.11:8000` — слушает только на LAN-интерфейсе, не на 0.0.0.0
- `--path /mnt/data/restic` — каталог с репо
- `--htpasswd-file /etc/rest-server/.htpasswd` — bcrypt-хеши HTTP-паролей
- `--append-only` — DELETE-операции запрещены, возвращают 403
- `--private-repos` — каждый аутентифицированный юзер видит только свой репо `/<username>/`, не имеет даже read-доступа к чужим
- `--log /var/log/rest-server/access.log` — отдельный access log

**HTTP-аутентификация.** В `/etc/rest-server/.htpasswd` три записи: `vaultwarden`, `authelia`, `dockerhost`. Файл создан через `htpasswd -B` (bcrypt). Owner `root:rest-server`, mode 640 — rest-server только читает.

**Транспорт.** HTTP без TLS — приемлемо внутри LAN, потому что содержимое снапшотов шифруется restic-клиентом перед отправкой (encryption-пароль не передаётся по сети, только сами зашифрованные данные). Перехват HTTP basic-auth credentials в LAN ограничен только активными атаками (ARP-spoofing) с уже скомпрометированного устройства. И даже при перехвате append-only не даёт атакующему уничтожить существующие бэкапы — только записать мусорные новые (DoS, обнаруживается мониторингом). Если когда-нибудь понадобится TLS, можно прицепить Traefik (route `restic.kvasok.xyz`) перед rest-server без миграции данных.

**Sandbox systemd-юнита:** `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `PrivateDevices`, `ProtectKernel*`, `RestrictNamespaces`, `LockPersonality`. Запись разрешена только в `/mnt/data/restic` и `/var/log/rest-server` через `ReadWritePaths`.

#### 12.1.4. Локальный restic для retention

Поскольку rest-server отвечает 403 на любые DELETE-операции, retention (`restic forget --prune`) делается локально на бэкап-сервере под root, прямым доступом к каталогам репо в `/mnt/data/restic/<service>/`. Это обходит rest-server полностью — restic работает с файлами на диске напрямую как с локальным репо.

**Файловая структура:**

```
/etc/restic-retention/              # encryption-пароли для retention (chmod 700)
├── vaultwarden.password
├── authelia.password
└── dockerhost.password
/usr/local/sbin/restic-retention.sh # единый скрипт retention для всех репо
/etc/systemd/system/restic-retention.service
/etc/systemd/system/restic-retention.timer
```

**Скрипт retention** проходит по массиву репо вида `<repo-dir>:<tag>` и применяет одну общую политику ко всем:

- `--keep-daily 7`
- `--keep-weekly 4`
- `--keep-monthly 6`
- `--group-by host,tags`
- `--prune` (физически удаляет осиротевшие chunks, перепаковывает packs)

**Расписание.** Systemd-timer запускает retention ежедневно в 03:00 MSK с `RandomizedDelaySec=30min`. Это после backup-таймеров клиентов (которые срабатывают в 01:00), чтобы не пересекаться по локам репо. `Persistent=true` — если PBS был выключен в момент срабатывания, скрипт запустится при следующем старте.

**Важная архитектурная особенность.** Encryption-пароли репо хранятся на двух хостах: на самом клиенте (для backup) и на бэкап-сервере (для retention). Это объективное ограничение restic — любая операция изменяющая снапшоты требует пароль шифрования, даже forget работающий локально с файлами. Изоляция бэкапов от компрометации клиента обеспечивается не разделением паролей, а append-only режимом rest-server: клиент технически не может выполнить DELETE независимо от того, есть у него encryption-пароль или нет. Если потребуется усилить изоляцию (защита конфиденциальности содержимого старых снапшотов при компрометации клиента), можно перейти на схему с `restic key add` — добавить независимый ключ только для retention на стороне сервера.

### 12.2. Уровень 1: Конфигурация pve-хоста через PBS host backup

**Что бэкапится:** критичные конфиги самого Proxmox-хоста — `/etc/pve` (конфиги всех VM/LXC, storage, пользователи), `/etc/network` (бридж, IP), `/etc/cron.d`, `/etc/nut` (UPS), `/root`, плюс `/etc/hosts`, `/etc/hostname`, `/etc/resolv.conf`.

**Чем:** `proxmox-backup-client` на pve, отдельный API-токен `pve-backup@pbs!pve-host` с ролью `DatastoreBackup`.

**Откуда:** скрипт `/usr/local/bin/pve-host-backup.sh` на pve, запускается systemd-таймером.

**Расписание:** ежедневно, `Persistent=true`.

**Зачем отдельно:** PBS бэкапит гостей, но не сам Proxmox. При полной потере системного диска pve без host backup пришлось бы восстанавливать `/etc/pve` и сетевые конфиги вручную. С host backup восстановление сводится к `proxmox-backup-client restore` поверх свежей установки.

**Файлы:**
- `/root/.pbs/host-backup.env` — credentials (chmod 600)
- `/usr/local/bin/pve-host-backup.sh` — скрипт бэкапа
- `/etc/systemd/system/pve-host-backup.service` — service-юнит
- `/etc/systemd/system/pve-host-backup.timer` — timer-юнит

### 12.3. Уровень 2: VM и LXC через PBS

**Что бэкапится:** образы дисков всех VM и LXC + их конфиги.

**Чем:** Proxmox Backup Server, datastore `Homelab` (`/mnt/data/pbs/datastore`).

**Откуда:** инициирует pve, через storage `pbs-homelab` (подключён через API-токен `pve-backup@pbs!pve-main` с ролью `DatastoreBackup`). Роль `DatastoreBackup` обеспечивает append-only-свойство: `pve` может писать новые снапшоты в PBS, но физически не может удалять или изменять существующие через тот же токен. Компрометация `pve` не даёт возможности уничтожить бэкапы. Подробнее про identity model PBS см. раздел 12.1.1.

**Расписание:** ежедневно, mode `snapshot`, compression `ZSTD`, selection `All` (все VM/LXC автоматически).

**Важный нюанс passthrough дисков DockerHost:** у DockerHost VM (120) прокинуты 7 физических дисков (scsi1–scsi7) для ZFS-пулов. На каждом из них стоит флаг **`backup=0`** в конфиге VM, чтобы PBS не пытался бэкапить ~27 ТБ ZFS-пулов. PBS бэкапит только системный диск VM (scsi0). Данные ZFS-пулов бэкапятся отдельно через restic (см. уровень 4) либо не бэкапятся вообще (медиатека, frigate-records).

### 12.4. Уровень 3: Данные клиентских сервисов через restic

PBS-снапшот целого LXC/VM закрывает сценарий «снёс целиком, нужно восстановить за минуту». Но для сервисов с собственным состоянием (БД, конфиги, данные) этого мало: при компрометации или повреждении нужна возможность достать конкретную версию файлов от N дней назад без подъёма всего хоста. Поэтому поверх PBS-снапшотов три сервиса (Vaultwarden, Authelia, DockerHost) делают независимый restic-бэкап своих данных.

**Транспорт.** Бэкап идёт через rest-server на бэкап-сервере в режиме `--append-only` + `--private-repos` (см. раздел 12.1.3). Каждый сервис аутентифицируется через HTTP basic-auth со своим паролем, видит только свой репо, физически не может удалять снапшоты — DELETE-операции возвращают 403 на уровне API.

**Backup.** Каждый сервис выполняет свой bash-скрипт под root (DockerHost) или под dedicated сервисным юзером (Vaultwarden, Authelia), запускается systemd-таймером ежедневно в 01:00 MSK. Скрипт делает encryption данных своим паролем перед отправкой.

**Retention.** Централизованно на бэкап-сервере под root, единый скрипт обрабатывает все три репо с одной политикой 7d/4w/6m (см. раздел 12.1.4).

#### Vaultwarden (192.168.1.17)

- **Что бэкапится:** `/var/lib/vaultwarden/data/` — БД (через online-снапшот `sqlite3 .backup`), `attachments/`, `rsa_key.pem`. Исключаются живой `db.sqlite3`, WAL-файлы, `icon_cache/`, `tmp/`.
- **Тег/host:** `vaultwarden` / `vaultwarden`.
- **URL репо:** `rest:http://vaultwarden:...@192.168.1.11:8000/vaultwarden/`.
- **Скрипт:** `/usr/local/sbin/vaultwarden-backup.sh`, таймер `vaultwarden-backup.timer`.
- **Восстановление БД:** после `restic restore` файл лежит как `db.sqlite3.backup` — переименовать в `db.sqlite3` перед запуском Vaultwarden.

#### Authelia (192.168.1.18)

- **Что бэкапится:** `/var/lib/authelia/` (БД через online-снапшот, `notifier.log`) и `/etc/authelia/` (все YAML-конфиги, секреты в `/etc/authelia/secrets/`). Исключаются живой `db.sqlite3`, WAL-файлы, `.restic-password`, `.restic-http-password`, `.cache/`.
- **Тег/host:** `authelia` / `authelia`.
- **URL репо:** `rest:http://authelia:...@192.168.1.11:8000/authelia/`.
- **Скрипт:** `/usr/local/sbin/authelia-backup.sh`, таймер `authelia-backup.timer`.
- **Восстановление БД:** аналогично Vaultwarden.

#### DockerHost (192.168.1.20)

- **Что бэкапится:** `/etc/samba/` (конфигурация Samba), `/home/romank/docker/` (compose-файлы с ENV), `/mnt/data/` (все Docker volumes и Samba shares).
- **Тег/host:** `dockerhost` / `dockerhost`.
- **URL репо:** `rest:http://dockerhost:...@192.168.1.11:8000/dockerhost/`.
- **Скрипт:** `/usr/local/sbin/dockerhost-backup.sh`, таймер `dockerhost-backup.timer`. Запускается под root, потому что бэкапит каталоги недоступные обычному юзеру.
- **Файлы паролей на хосте:** `/etc/restic-backup/encryption.password` и `/etc/restic-backup/http.password` (chmod 600, owner root).
- **Восстановление:** прямое `restic restore` нужного снапшота в нужный путь.

#### Зачем отдельно от PBS

- **Гранулярное восстановление** — отдельные файлы достаются за секунды, не нужно поднимать всю VM/LXC из PBS-снапшота
- **Долгая retention** — restic-снапшоты файлов весят мегабайты, PBS-снапшоты больших дисков весят гигабайты; restic можно хранить дольше
- **Эффективная дедупликация на уровне файлов** — для AppData это работает лучше, чем block-level дедупликация PBS поверх ZFS внутри VM

### 12.5. Retention-политики

#### PBS (для VM/LXC и host backup, единая политика)

Prune Job на PBS, ежедневно:

| Параметр | Значение |
|----------|----------|
| keep-last | 3 |
| keep-daily | 7 |
| keep-weekly | 4 |
| keep-monthly | 6 |

Garbage Collection (физическое удаление chunks): **воскресенье 07:00**.

Verify Job (проверка целостности chunks): **воскресенье 08:00**, skip-verified, re-verify after 30 дней.

#### Restic для клиентских сервисов

Единая retention-политика для всех трёх restic-репо (Vaultwarden, Authelia, DockerHost) — `7 daily / 4 weekly / 6 monthly` + `--prune`, группировка `--group-by host,tags`. Применяется централизованно из скрипта `/usr/local/sbin/restic-retention.sh` на бэкап-сервере, запускаемого systemd-таймером `restic-retention.timer` ежедневно в 03:00 MSK с `RandomizedDelaySec=30min`. Скрипты бэкапа на клиентах retention не выполняют — append-only режим rest-server физически блокирует DELETE-операции с их стороны.

### 12.6. Сценарий полного восстановления

При потере системного диска pve:

1. Установить свежий Proxmox на новый диск
2. Настроить базовую сеть (IP 192.168.1.10, бридж `vmbr0`), чтобы видел бэкап-сервер
3. Подключить PBS storage `pbs-homelab` через API-токен
4. Восстановить host backup: `proxmox-backup-client restore` для `/etc/pve`, `/etc/network`, `/etc/nut`, остальных конфигов
5. После перезапуска pve видит все конфиги VM/LXC из восстановленного `/etc/pve`
6. Восстановить VM/LXC из PBS-снапшотов через UI (Restore)
7. Запустить DockerHost VM, на ней сделать `zpool import zdata zmedia zfrigate` — ZFS-метаданные на физических дисках живы, пулы поднимутся
8. Запустить Docker-стэк через compose-файлы (либо они уже на восстановленной VM, либо тянутся из restic)
9. При необходимости восстановить отдельные файлы из restic. БД Vaultwarden — из `/mnt/data/restic/vaultwarden/`. БД и конфиги Authelia — из `/mnt/data/restic/authelia/`. Файлы AppData, compose-конфиги и Samba shares — из `/mnt/data/restic/dockerhost/`. Доступ к репо локально с бэкап-сервера: `restic -r /mnt/data/restic/<service>` с encryption-паролем из `/etc/restic-retention/<service>.password`.
