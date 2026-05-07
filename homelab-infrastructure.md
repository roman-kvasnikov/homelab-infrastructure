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

Роутер — Keenetic Giga (192.168.1.1). Основная LAN — 192.168.1.0/24 (SSID `Izolda-Rally`), для камер выделен отдельный сегмент 192.168.30.0/24. Между сегментами по умолчанию работает isolate-private, межсегментный трафик открыт только явными permit-правилами на firewall. VPN-подсеть для AmneziaWG-клиентов — **`10.8.0.0/24`**.

### 2.1. Основной сегмент LAN (192.168.1.0/24)

| IP             | Хост                  | Роль                                                       |
|----------------|-----------------------|------------------------------------------------------------|
| 192.168.1.1    | Keenetic Giga         | Роутер, fallback DNS                                       |
| 192.168.1.2    | AdGuard Home (LXC)    | Основной DNS, split-horizon                                |
| 192.168.1.3    | Xray-Proxy (LXC)      | HTTP/SOCKS прокси и DNS-over-Xray для гео-обхода           |
| 192.168.1.10   | Proxmox host          | Гипервизор                                                 |
| 192.168.1.11   | Backup server         | Хранилище для restic, поднимается через WoL                |
| 192.168.1.15   | Traefik LXC           | Reverse proxy, CrowdSec, WireGuard клиент к VPS            |
| 192.168.1.20   | DockerHost VM         | Все основные self-hosted сервисы в Docker, Samba           |
| 192.168.1.40   | Work VM               | Рабочая VM, доступ по SSH                                  |
| 192.168.1.101  | Huawei notebook       | Ноутбук на NixOS (host `huawei`)                           |
| 192.168.1.102  | DssMargo PC           | ПК партнёра                                                |

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

Порт 80 не открыт — ACME-валидация делается по DNS-01 на Traefik, в HTTP challenge нет нужды.

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

На VPS через nftables настроен **forwarding из `awg0` в `wg0`** — пакеты от пиров уходят дальше в служебный туннель к Traefik LXC. На Traefik LXC сделан **MASQUERADE**, поэтому для устройств в LAN трафик VPN-клиентов выглядит как идущий с `192.168.1.15`.

## 4. VPN-доступ к дому (AmneziaWG)

Внешние пиры (телефоны, ноутбуки) подключаются к `awg0` на VPS. Дальше их пакеты форвардятся в `wg0`, попадают на Traefik LXC и через MASQUERADE идут в LAN. Это даёт пирам полный доступ к домашней сети — как будто они физически дома.

### 4.1. Поток трафика

```
[Пир]  ── AWG / UDP 51821 ──>  [VPS]
                                  │
                                  │  forward awg0 → wg0
                                  ▼
                              [Traefik LXC]  ── MASQUERADE ──>  [LAN]
```

### 4.2. Сценарии использования

- Доступ к чисто внутренним сервисам, которые в публичный DNS не торчат (Proxmox UI, Backrest, AdGuard и т.п.).
- Прямые соединения по локальным IP `192.168.1.x` — SSH, SMB, доступ к железу.

### 4.3. Split-horizon DNS

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
    ├── system-диск DockerHost (100 GB)
    ├── system-диски LXC AdGuard, Xray, Traefik
    └── system-диски Dev VM, Be-Free.Online VM
```

### 5.3. ZFS-пул zguests

Создан с параметрами `compression=lz4`, `atime=off`, `xattr=sa`, `acltype=posixacl`. **ZFS ARC ограничен 4 GB** (`/etc/modprobe.d/zfs.conf`: `options zfs zfs_arc_max=4294967296`), чтобы не отъедать память у DockerHost VM. Остальная RAM остаётся доступной для гостей.

### 5.4. Лимиты ресурсов гостей

Чтобы избежать борьбы за CPU/RAM между pve и гостями (особенно при старте DockerHost с десятками контейнеров), у VM выставлены разумные лимиты:

| Гость         | RAM (max/min)     | Cores | Startup order/up/down     |
|---------------|-------------------|-------|---------------------------|
| DockerHost    | 32G / 24G balloon | 4     | order=10, up=60, down=180 |
| Dev (140)     | 4 GB              | 2     | order=5, up=5, down=60    |
| Be-Free (150) | 4 GB              | 2     | order=5, up=5, down=60    |
| AdGuard (102) | 512 MB / 512 swap | 1     | order=1, up=5, down=30    |
| Xray (103)    | 512 MB / 512 swap | 1     | order=1, up=30, down=30   |
| Traefik (115) | 1 GB / 1 GB swap  | 2     | order=2, up=5, down=30    |

### 5.5. UPS и NUT

На pve-хосте установлен и настроен **NUT** (Network UPS Tools) для корректного обращения с UPS при отключении электричества.

Конфиги: `/etc/nut/` (бэкапится в PBS host backup, см. 10.3).

При срабатывании события `low battery` NUT инициирует graceful shutdown гостей в обратном порядке startup (см. таблицу 5.4): сначала DockerHost (с downtime до 180 с на корректное завершение Docker-контейнеров), последним — AdGuard и Xray. Это даёт максимум шансов корректно сохранить состояние БД и не потерять данные.

## 6. Traefik LXC (192.168.1.15)

Reverse proxy для всех сервисов. Дополнительно держит служебный WireGuard-клиент `wg0` к VPS (`10.0.0.2`).

### 6.1. Домены и сертификаты

- **kvasok.xyz** — основной домен homelab. Wildcard `*.kvasok.xyz` резолвится на VPS `77.232.136.6`. На нём публикуются self-hosted сервисы.
- Сертификаты Let's Encrypt получает **Traefik через DNS Challenge** (wildcard `*.kvasok.xyz`). На VPS никаких сертификатов нет — он работает только на L4 (stream-проксирование).
- TLS-резолвер по умолчанию — `timewebcloud`, для отдельных хостов также `namecheap`.

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
| `crowdsec`                    | плагин CrowdSec (LAPI `127.0.0.1:8080`, mode `live`)                        |
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

- **`internal-chain`** — только LAN и VPN. Применяется к админским / чисто внутренним сервисам: Traefik dashboard, Proxmox UI, AdGuard, Portainer, pgAdmin, Backrest, Frigate.
- **`external-chain`** — публичные сервисы с обычным rate-limit'ом: Jellyfin, Nextcloud, Vaultwarden, Paperless, Microbin, Baikal, Mealie, Filebrowser, OnlyOffice.
- **`external-chain-strict`** — для случаев, где нужен жёсткий лимит на скорость запросов.
- **`external-chain-no-rate-limit`** — для случаев, когда лимит мешает. Например в Immich, когда листаешь галерею и идет множество запросов, чтобы при этом пользователя не заблокировало.

Многие сервисы дополнительно прикрыты Authelia через middleware `authelia` (forwardAuth) — обычно как `internal-chain + authelia` или `external-chain + authelia`.

### 6.6. Доверие к заголовкам и trusted IPs

В CrowdSec-плагине:

- `clientTrustedIPs`: `192.168.1.0/24`, `10.8.0.0/24` — запросы от этих источников не банятся (LAN и VPN-клиенты).
- `forwardedHeadersTrustedIPs`: `10.0.0.1/32` — Traefik доверяет `X-Forwarded-*` заголовкам только от VPS (наш wg0-сосед).

## 7. CrowdSec

Engine крутится на Traefik LXC рядом с Traefik. Bouncer интегрирован как плагин Traefik (Yaegi).

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
- Если CrowdSec engine упадёт, bouncer продолжит работу с последним списком, но новые баны добавляться не будут. Поведение при недоступности LAPI настраивается параметром `defaultDecision` плагина.

## 8. DockerHost VM (192.168.1.20)

Ubuntu VM, основной хост для всех Docker-сервисов. ZFS делается на уровне VM (disk passthrough из Proxmox), чтобы корректно работали hardlinks для *-arr / qBittorrent / Jellyfin.

### 8.1. ZFS-датасеты

| Pool       | Mountpoint           | Назначение                           |
|------------|----------------------|--------------------------------------|
| `zdata`    | `/mnt/data`          | AppData всех приложений              |
| `zmedia`   | `/mnt/media`         | Медиатека Jellyfin                   |
| `zfrigate` | `/var/lib/frigate`   | Записи Frigate                       |

### 8.2. Docker networking

Default address pool в `/etc/docker/daemon.json` ограничен диапазоном `10.200.0.0/16` (size `/24`), чтобы Docker автоматически создаваемые сети не пересекались с физическими подсетями LAN/Cameras/VPN:

```json
{
  "default-address-pools": [
    {"base": "10.200.0.0/16", "size": 24}
  ]
}
```

### 8.3. Структура AppData

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

### 8.4. Контейнеры

Основные сервисы (не исчерпывающий список): immich (server + ML + redis + postgres), общий postgres + pgadmin, vaultwarden, nextcloud, paperless, microbin, baikal (CardDAV/CalDAV), authelia, jellyfin, sonarr, radarr, qbittorrent, jellyseerr, frigate, filebrowser quantum, onlyoffice, mealie, portainer, backrest. Watchtower не используется — обновления вручную через `docker compose pull`.

#### Особенности

- **PostgreSQL** общий для большинства приложений (vaultwarden, nextcloud, paperless, authelia и т.п.). Для Immich — отдельный, потому что он требует специфичный образ с pgvector/vectorchord.
- **Docker network** `postgres` (`external: true`) — общая сеть для доступа к общему PostgreSQL.
- **Jellyfin / Radarr / Sonarr / Jellyseerr** настроены ходить в интернет через Xray-Proxy (`HTTP_PROXY=http://192.168.1.3:10809`).
- **Frigate** ходит на камеры в сегмент 192.168.30.0/24 по RTSP (порт 554) и ONVIF (порт 2020 для Camera-Hall). Доступ открыт permit-правилами Keenetic на интерфейсе «Домашняя сеть».
- **Backrest** делает SFTP-бэкапы на `192.168.1.11`. Перед бэкапом будит backup-сервер через `wakeonlan` (по SSH с проброшенным ключом запускается на DockerHost-хосте), после бэкапа выключает его командой `shutdown` по SSH.

### 8.5. Samba

На DockerHost (не в контейнере) поднят Samba-сервер. Слушает только на **445/tcp**. Доступ с ноутбука (`192.168.1.101`) и ПК (`192.168.1.102`). Шары используют ZFS-пулы напрямую.

## 9. Аутентификация

Основной IdP — **Authelia**. Конфигурация OIDC-клиентов прописывается в `configuration.yml`. 2FA через TOTP, поддержка PKCE для мобильных приложений. Forward-auth делается middleware `authelia` в Traefik — любой защищённый сервис получает `internal-chain + authelia` или `external-chain + authelia`.

## 10. Резервное копирование

Бэкапы организованы по **трёхуровневой схеме**, где каждый уровень закрывает свою задачу и использует подходящий для неё инструмент. Все бэкапы складываются на один **бэкап-сервер 192.168.1.11**, но в разные структуры на одном RAID0 массиве.

### 10.1. Бэкап-сервер (192.168.1.11)

ОС — **Proxmox Backup Server** (поверх Debian 12). Помимо самого PBS, на сервере поднят **SSH/SFTP-доступ** для приёма restic-бэкапов с DockerHost VM.

Дисковая структура:

```
sda (465 GB)         — системный SSD, ext4, ОС и PBS
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
    └── dockerhost/       # restic-репо для бэкапов с DockerHost VM
```

### 10.2. Уровень 1: VM и LXC через PBS

**Что бэкапится:** образы дисков всех VM и LXC + их конфиги.

**Чем:** Proxmox Backup Server, datastore `Homelab` (`/mnt/data/pbs/datastore`).

**Откуда:** инициирует pve, через storage `pbs-homelab` (подключён через API-токен `pve-backup@pbs!pve-main` с ролью `DatastoreAdmin`).

**Расписание:** ежедневно в **05:00**, mode `snapshot`, compression `ZSTD`, selection `All` (все VM/LXC автоматически).

**Важный нюанс passthrough дисков DockerHost:** у DockerHost VM (120) прокинуты 7 физических дисков (scsi1–scsi7) для ZFS-пулов. На каждом из них стоит флаг **`backup=0`** в конфиге VM, чтобы PBS не пытался бэкапить ~27 ТБ ZFS-пулов. PBS бэкапит только системный диск VM (scsi0, 100 GB). Данные ZFS-пулов бэкапятся отдельно через restic (см. уровень 3) либо не бэкапятся вообще (медиатека, frigate-records).

### 10.3. Уровень 2: Конфигурация pve-хоста через PBS host backup

**Что бэкапится:** критичные конфиги самого Proxmox-хоста — `/etc/pve` (конфиги всех VM/LXC, storage, пользователи), `/etc/network` (бридж, IP), `/etc/cron.d`, `/etc/nut` (UPS), `/root`, плюс `/etc/hosts`, `/etc/hostname`, `/etc/resolv.conf`.

**Чем:** `proxmox-backup-client` на pve, отдельный API-токен `pve-backup@pbs!pve-host` с ролью `DatastoreBackup`.

**Откуда:** скрипт `/usr/local/bin/pve-host-backup.sh` на pve, запускается systemd-таймером.

**Расписание:** ежедневно в **04:30** (за 30 минут до бэкапа VM/LXC), `Persistent=true`.

**Зачем отдельно:** PBS бэкапит гостей, но не сам Proxmox. При полной потере системного диска pve без host backup пришлось бы восстанавливать `/etc/pve` и сетевые конфиги вручную. С host backup восстановление сводится к `proxmox-backup-client restore` поверх свежей установки.

**Файлы:**
- `/root/.pbs/host-backup.env` — credentials (chmod 600)
- `/usr/local/bin/pve-host-backup.sh` — скрипт бэкапа
- `/etc/systemd/system/pve-host-backup.service` — service-юнит
- `/etc/systemd/system/pve-host-backup.timer` — timer-юнит

### 10.4. Уровень 3: Данные внутри DockerHost через restic

**Что бэкапится:**
- `/mnt/data/` — все volumes сервисов в Docker
- `/home/romank/docker/` — compose-файлы с ENV (дополнительная страховка к PBS-бэкапу системного диска)

**Чем:** Backrest (GUI для restic) на DockerHost VM.

**Куда:** SFTP на бэкап-сервер, репозиторий `/mnt/data/restic/dockerhost/`.

**Зачем отдельно от PBS:**
- **Гранулярное восстановление** — отдельные файлы достаются за секунды, не нужно поднимать всю VM из PBS-снапшота
- **Долгая retention** — restic-снапшоты файлов весят мегабайты, PBS-снапшоты больших дисков весят гигабайты; restic можно хранить дольше
- **Эффективная дедупликация на уровне файлов** — для AppData это работает лучше, чем blok-level дедупликация PBS поверх ZFS внутри VM

### 10.5. Retention-политики

#### PBS (для VM/LXC и host backup, единая политика)

Prune Job на PBS, расписание **06:00** ежедневно (через час после бэкапа VM/LXC):

| Параметр | Значение |
|----------|----------|
| keep-last | 3 |
| keep-daily | 7 |
| keep-weekly | 4 |
| keep-monthly | 6 |

Garbage Collection (физическое удаление chunks): **воскресенье 07:00**.

Verify Job (проверка целостности chunks): **воскресенье 08:00**, skip-verified, re-verify after 30 дней.

#### Restic

Retention настраивается на стороне Backrest, политики задаются для каждой задачи отдельно. Дампы PostgreSQL и AppData хранятся дольше, чем системные снапшоты, потому что физически весят меньше.

### 10.6. Цепочка задач Backrest

```
wake-backup-server (WoL + ожидание SSH)
        ↓
restic-backups-<job-1>
        ↓
restic-backups-<job-2>
        ↓
...
restic-backups-<last-job>
        ↓ OnSuccess
shutdown-backup-server (SSH shutdown -h now)
```

Бэкап-сервер большую часть суток выключен. Backrest:
1. Отправляет WoL magic packet на MAC бэкап-сервера
2. Ждёт, пока SSH станет доступен
3. Прогоняет restic-задачи последовательно
4. По успешному завершению последней задачи — `shutdown -h now` по SSH

**TODO:** координация WoL с PBS-окном бэкапов (04:30–08:00). Сейчас расписания PBS и Backrest могут конфликтовать — если Backrest выключит сервер раньше, чем pve успеет залить свои бэкапы. Нужно либо:
- Добавить WoL на стороне pve перед PBS-бэкапом и держать сервер включённым на всё PBS-окно
- Либо синхронизировать расписания так, чтобы Backrest гарантированно завершался ДО PBS-окна и сервер просыпался от pve самостоятельно

### 10.7. Сценарий полного восстановления

При потере системного диска pve:

1. Установить свежий Proxmox на новый диск
2. Настроить базовую сеть (IP 192.168.1.10, бридж `vmbr0`), чтобы видел бэкап-сервер
3. Подключить PBS storage `pbs-homelab` через API-токен
4. Восстановить host backup: `proxmox-backup-client restore` для `/etc/pve`, `/etc/network`, `/etc/nut`, остальных конфигов
5. После перезапуска pve видит все конфиги VM/LXC из восстановленного `/etc/pve`
6. Восстановить VM/LXC из PBS-снапшотов через UI (Restore)
7. Запустить DockerHost VM, на ней сделать `zpool import zdata zmedia zfrigate` — ZFS-метаданные на физических дисках живы, пулы поднимутся
8. Запустить Docker-стэк через compose-файлы (либо они уже на восстановленной VM, либо тянутся из restic)
9. При необходимости восстановить отдельные файлы AppData из restic
