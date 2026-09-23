---
name: proxmox
description: |
  Вычислительный слой homelab: два хоста Proxmox VE. PVE-Mini несёт сетевую плоскость (OPNsense, Omada, AmneziaWG, Xray), PVE — прикладные сервисы, AI-агента и Home Assistant. Документ описывает сеть хостов, storage и ZFS-пулы, размещение данных гостей, iGPU и общие группы, порядок запуска, UPS/NUT, host-firewall (nftables) и инвентарь VM/LXC.
---

# Гипервизоры Proxmox

Вычислительная нагрузка распределена на два хоста Proxmox VE. Разделение по роли: **PVE-Mini** несёт сетевую плоскость (маршрутизатор, контроллер коммутатора, VPN, прокси), **PVE** — прикладные сервисы. Оба хоста управляются из MGMT (VLAN 10); гости получают VLAN тегом на виртуальном интерфейсе.

CTID/VMID на PVE совпадает с последним октетом адреса гостя и кодирует сегмент: `5xx` — SERVICES (`192.168.50.xx`), `4xx` — DMZ, `2xx` — INFRA, `199` — MGMT. На PVE-Mini гости нумеруются последовательно (`101–104`).

Общие соглашения по LXC (unprivileged, features, sandbox, nftables) вынесены в `02-conventions.md` и здесь не повторяются.

---

## 1. PVE-Mini

Мини-ПК на Intel N150 с двумя сетевыми интерфейсами Intel I226-V. Несёт сетевую плоскость дома: на нём запущен OPNsense, через который проходит весь трафик сети. Управляющий адрес хоста — `192.168.10.11` (MGMT).

### Сеть хоста

Два физических интерфейса разведены по ролям — один под WAN, второй под LAN-транк со всеми VLAN. Оба моста VLAN-aware (`bridge-vlan-aware yes`, `bridge-vids 2-4094`).

- **`nic0` → `vmbr0`** — WAN. Мост без IP на хосте, в него заходит кабель провайдера; единственный потребитель — WAN-интерфейс OPNsense.
- **`nic1` → `vmbr1`** — LAN-транк, несёт все VLAN тегированными. Подключён в порт 8 коммутатора.
- **`vmbr1.10`** — VLAN-подынтерфейс поверх транка: адрес хоста `192.168.10.11/24`, шлюз `192.168.10.1`. MGMT приходит на этот линк тегированным, а не native.

Хост не имеет IP на `vmbr0` — WAN целиком принадлежит OPNsense.

### Storage

Один SATA SSD (Samsung 860 EVO M.2 500GB) под систему и гостей, ZFS на хосте нет.

| Storage     | Тип     | Назначение                                     |
| :---------- | :------ | :--------------------------------------------- |
| `local`     | dir     | ISO, шаблоны, import                           |
| `local-lvm` | lvmthin | Диски всех гостей PVE-Mini                     |
| `pbs-main`  | pbs     | Бэкап-цель, datastore `main`, namespace `pve-mini` |

### Гости

| ID  | Имя              | Тип | VLAN           | Адрес                   | RAM  | Назначение                        |
| :-- | :--------------- | :-- | :------------- | :---------------------- | :--- | :-------------------------------- |
| 101 | OPNsense         | VM  | WAN + все VLAN | шлюз `.1` в каждом VLAN | 6 GB | Маршрутизатор, firewall, DHCP/DNS |
| 102 | OmadaController  | LXC | MGMT (10)      | `192.168.10.31`         | 4 GB | Контроллер коммутатора TP-Link    |
| 103 | AmneziaWG        | LXC | INFRA (20)     | `192.168.20.11`         | 512 MB | VPN удалённого доступа          |
| 104 | Xray             | VM  | INFRA (20)     | `192.168.20.12`         | 1 GB | Прокси гео-обхода                 |

**OPNsense (101)** — `net0` → `vmbr0` (WAN), `net1` → `vmbr1` (LAN-транк, VLAN-интерфейсы внутри VM), CPU type `host`, virtio-сеть. Детали — в `03-network.md` и `04-firewall.md`.

**OmadaController (102)** — контроллер управляемого коммутатора TP-Link, интерфейс в MGMT (`tag=10`).

**AmneziaWG (103)** — обфусцированный VPN-сервер удалённого доступа. Unprivileged LXC с пробросом TUN-устройства (`lxc.cgroup2.devices.allow: c 10:200 rwm` и bind-mount `/dev/net/tun`). Детали — в `09-amneziawg.md`.

**Xray (104)** — прокси для гео-обхода и прозрачного проксирования. Детали — в `10-xray.md`.

---

## 2. PVE

Основной сервер, несёт прикладные сервисы. 62 GiB RAM. Управляющий адрес хоста — `192.168.10.12` (MGMT).

### Сеть хоста

Единственный VLAN-aware мост `vmbr0` (`bridge-vlan-aware yes`, `bridge-vids 2-4094`) с портами `nic0` и `nic1`. Рабочий линк — `nic0`, подключён транком в порт 5 коммутатора; на нём включён Wake-on-LAN (`ethtool -s nic0 wol g`). `nic1` входит в мост, но кабель в него не подключён.

Адрес хоста `192.168.10.12/24` (шлюз `192.168.10.1`) назначен прямо на `vmbr0`: MGMT приходит на порт native. Поэтому гости в MGMT подключаются без `tag=`, гости в остальных сегментах — с тегом своего VLAN.

### Гости

| ID  | Имя            | Тип | VLAN          | Адрес           | RAM    | Назначение                                        |
| :-- | :------------- | :-- | :------------ | :-------------- | :----- | :------------------------------------------------ |
| 199 | mgmt           | LXC | MGMT (10)     | `192.168.10.99` | 2 GB   | Ansible control node (`16-ansible.md`)            |
| 220 | homepage       | LXC | INFRA (20)    | `192.168.20.20` | 1 GB   | Дашборд Homepage                                  |
| 411 | traefik        | LXC | DMZ (40)      | `192.168.40.11` | 1 GB   | Reverse-proxy + CrowdSec (`08-traefik.md`)        |
| 511 | vaultwarden    | LXC | SERVICES (50) | `192.168.50.11` | 512 MB | Менеджер паролей (`12-vaultwarden.md`)            |
| 512 | authelia       | LXC | SERVICES (50) | `192.168.50.12` | 512 MB | IdP, forward-auth (`11-authelia.md`)              |
| 521 | monitoring     | LXC | SERVICES (50) | `192.168.50.21` | 2 GB   | Prometheus, Grafana, PeaNUT (`15-monitoring.md`)  |
| 522 | gotify         | LXC | SERVICES (50) | `192.168.50.22` | 512 MB | Push-уведомления (`13-gotify.md`)                 |
| 531 | jellyfin       | LXC | SERVICES (50) | `192.168.50.31` | 2 GB   | Медиасервер (`14-media-stack.md`)                 |
| 532 | arr            | LXC | SERVICES (50) | `192.168.50.32` | 2 GB   | Prowlarr, Sonarr, Radarr (`14-media-stack.md`)    |
| 533 | qbittorrent    | LXC | SERVICES (50) | `192.168.50.33` | 4 GB   | Торрент-клиент (`14-media-stack.md`)              |
| 534 | organizer      | LXC | SERVICES (50) | `192.168.50.34` | 2 GB   | Anchor, ByteStash, Baikal (Docker-in-LXC)         |
| 535 | immich         | LXC | SERVICES (50) | `192.168.50.35` | 6 GB   | Фотоархив (Docker-in-LXC)                         |
| 536 | shares         | LXC | SERVICES (50) | `192.168.50.36` | 2 GB   | Samba, FileBrowser                                |
| 537 | frigate        | LXC | SERVICES (50) | `192.168.50.37` | 4 GB   | NVR (Docker-in-LXC)                               |
| 538 | onlyoffice     | LXC | SERVICES (50) | `192.168.50.38` | 4 GB   | Сервер документов (Docker-in-LXC)                 |
| 539 | open-webui     | LXC | SERVICES (50) | `192.168.50.39` | 4 GB   | Open WebUI (Docker-in-LXC)                        |
| 541 | tdarr          | LXC | SERVICES (50) | `192.168.50.41` | 16 GB  | Транскодирование медиатеки (Docker-in-LXC)        |
| 545 | forgejo        | LXC | SERVICES (50) | `192.168.50.45` | 2 GB   | Self-hosted Git                                   |
| 580 | yandex-disk    | LXC | SERVICES (50) | `192.168.50.80` | 512 MB | Синхронизация Яндекс.Диска (Docker-in-LXC)        |
| 590 | postgres       | LXC | SERVICES (50) | `192.168.50.90` | 2 GB   | Общий PostgreSQL и Redis                          |
| 540 | Dev            | VM  | SERVICES (50) | `192.168.50.40` | 4 GB   | Среда разработки                                  |
| 550 | Be-Free.Online | VM  | SERVICES (50) | `192.168.50.50` | 4 GB   | Панель Remnawave VPN-сервиса                      |
| 570 | Hermes         | VM  | SERVICES (50) | `192.168.50.70` | 4 GB   | AI-агент                                          |
| 571 | Home-Assistant | VM  | SERVICES (50) | `192.168.50.71` | 2 GB   | Home Assistant                                    |

Features контейнеров: нативные LXC — `nesting=1`; Docker-in-LXC organizer, immich, frigate, onlyoffice, yandex-disk — `nesting=1,keyctl=1`; open-webui и tdarr (тоже Docker) — `nesting=1`; forgejo — без features.

VM используют CPU type `x86-64-v2-AES` и virtio-сеть; диски — на `local-zfs`.

---

## 3. Storage и ZFS

### Storage PVE

| Storage     | Тип     | Пул          | Назначение                                                   |
| :---------- | :------ | :----------- | :----------------------------------------------------------- |
| `local`     | dir     | `rpool`      | ISO, шаблоны LXC, import, backup                             |
| `local-zfs` | zfspool | `rpool/data` | Rootfs всех LXC, диски всех VM, малые managed-volume'ы       |
| `zdata`     | zfspool | `zdata`      | Крупные managed-volume'ы ценных данных                       |
| `zssd`      | zfspool | `zssd`       | Scratch-пул                                                  |
| `pbs-main`  | pbs     | —            | Бэкап-цель, datastore `main`, namespace `pve`                |

`zmedia` и `zfrigate` не заведены как storage Proxmox — они подключаются к гостям только bind-mount'ом.

### Диски и пулы

```
NVMe (Netac 1TB):
└── rpool → система PVE + storage local / local-zfs

SATA SSD (2× P3-256, stripe):
└── zssd → /zssd — scratch (tdarr-cache)

HDD:
├── zdata    → /zdata    RAIDZ1, 4× WD40EZAX (4 TB) — фото, шары, Forgejo, Immich
├── zmedia   → /zmedia   WD120EDAZ (12 TB), одиночный — медиатека
└── zfrigate → /zfrigate HGST HTS545050 (500 GB), одиночный — записи камер
```

| Пул        | compression | atime |
| :--------- | :---------- | :---- |
| `rpool`    | `on`        | `on`  |
| `zdata`    | `lz4`       | `off` |
| `zmedia`   | `off`       | `off` |
| `zfrigate` | `lz4`       | `off` |
| `zssd`     | `lz4`       | `on`  |

На `zmedia` сжатие отключено — медиафайлы уже сжаты.

**ZFS ARC ограничен 16 GiB** (`/etc/modprobe.d/zfs.conf`: `options zfs zfs_arc_max=17179869184`, применено в initramfs) — баланс между кэшем файловой системы и памятью гостей на хосте с 62 GiB RAM без swap.

`storage.cfg` держит mountpoint пулов `zdata` и `zssd` синхронно с ZFS-свойством `mountpoint`: смена точки монтирования пула требует одновременного обновления обоих, иначе LXC с томами на этом пуле не стартуют.

### Размещение данных гостей

Данные размещаются по двум признакам — размер и ценность.

**Малое критичное состояние** (конфиги, SQLite, БД PostgreSQL) лежит на `local-zfs` (NVMe) отдельными managed-volume с `backup=1`. Оно попадает в ежедневный vzdump-снапшот PBS вместе с rootfs, поэтому отказ NVMe закрывается восстановлением гостей из PBS целиком — rootfs и так лежат на том же диске, отдельный отказоустойчивый пул для малых томов доступности не добавляет.

**Крупные ценные данные** (фото, домашнее видео, файловые шары, git-репозитории) лежат на `zdata` (RAIDZ1, переживает отказ одного диска). Те, что не помещаются в vzdump, защищаются отдельным file-level бэкапом (`06-backup.md`).

**Крупные восстановимые данные** (медиатека, записи камер) лежат на одиночных дисках `zmedia` и `zfrigate` и не бэкапятся.

**Scratch** — `zssd`, данные без ценности (кэш транскода).

| Гость       | Точка монтирования                         | Источник                       | backup |
| :---------- | :----------------------------------------- | :----------------------------- | :----- |
| traefik     | `/etc/traefik`, `/etc/crowdsec`            | `local-zfs` managed-volume     | `1`    |
| vaultwarden | `/var/lib/vaultwarden/data`, `/etc/vaultwarden` | `local-zfs` managed-volume | `1`    |
| authelia    | `/var/lib/authelia`, `/etc/authelia`       | `local-zfs` managed-volume     | `1`    |
| organizer   | `/data`                                    | `local-zfs` managed-volume     | `1`    |
| frigate     | `/config`                                  | `local-zfs` managed-volume     | `1`    |
| frigate     | `/media/frigate`                           | bind `/zfrigate`               | `0`    |
| postgres    | `/var/lib/postgresql`                      | `local-zfs` managed-volume     | `1`    |
| immich      | `/data/pgdata`                             | `zdata` managed-volume         | `1`    |
| immich      | `/data/media`                              | `zdata` managed-volume (2.5 TB) | `0`   |
| forgejo     | `/var/lib/forgejo`                         | `zdata` managed-volume         | `1`    |
| shares      | `/shares`                                  | bind `/zdata/Shares`           | `0`    |
| yandex-disk | `/data`                                    | bind `/zdata/Shares/YandexDisk` | `0`   |
| jellyfin    | `/data/Media`                              | bind `/zmedia/Media`           | `0`    |
| jellyfin    | `/home-media`, `/rally-media`              | bind `/zdata/Shares/Media`, `/zdata/Shares/Shared/Rally` | `0` |
| arr         | `/data`                                    | bind `/zmedia`                 | `0`    |
| qbittorrent | `/data/Torrents`                           | bind `/zmedia/Torrents`        | `0`    |
| tdarr       | `/cache`                                   | bind `/zssd/tdarr-cache`       | `0`    |
| tdarr       | `/home-media`, `/rally-media`              | bind `/zdata/Shares/Media`, `/zdata/Shares/Shared/Rally` | `0` |

Остальные гости держат состояние на rootfs.

### PBS

Целевое хранилище бэкапов — Proxmox Backup Server на `192.168.10.15` (MGMT). На обоих хостах storage называется `pbs-main` (datastore `main`), namespace соответствует хосту: `pve` и `pve-mini`, токены `backup@pbs!pve` / `backup@pbs!pve-mini`. На стороне Proxmox задано `prune-backups keep-all=1` — retention выполняется только на PBS. Детали — в `06-backup.md`.

---

## 4. iGPU и общие группы

Пулы данных и iGPU принадлежат хосту PVE и раздаются в LXC, а не пробрасываются в отдельную VM.

**iGPU.** Intel iGPU управляется хостовым драйвером `i915`. Устройства `/dev/dri/card0` (`226:0`, группа `video`, GID 44) и `/dev/dri/renderD128` (`226:128`, группа `render`, GID 993) разделяются между четырьмя unprivileged LXC: jellyfin, immich, frigate, tdarr. Каждый получает `lxc.cgroup2.devices.allow` на оба устройства, bind-mount через `lxc.mount.entry` и idmap GID 44 и 993 напрямую хост↔контейнер. Аппаратный транскод видео (VAAPI/QuickSync) в unprivileged LXC работает; GPU-compute (OpenVINO) не инициализируется, поэтому ML Immich и детекция Frigate работают на CPU.

**Общие группы.** Доступ нескольких контейнеров к одним данным идёт через группы, GID которых маппится idmap'ом напрямую хост↔контейнер:

| Группа  | GID  | Данные                     | Контейнеры                             |
| :------ | :--- | :------------------------- | :------------------------------------- |
| `media` | 981  | `zmedia`                   | jellyfin, arr, qbittorrent             |
| `shares`| 1001 | `zdata/Shares`             | shares, yandex-disk, jellyfin, tdarr   |

Детали idmap медиастека — в `14-media-stack.md`.

---

## 5. Порядок запуска

Порядок задан через `startup` (order/up/down) независимо на каждом хосте. Инфраструктурные гости стартуют первыми, прикладные — следом; при выключении, в том числе по сигналу UPS, порядок обратный.

| order | PVE-Mini                        | PVE                                                     |
| :---- | :------------------------------ | :------------------------------------------------------ |
| 1     | OPNsense (`up=60`, `down=120`)  | postgres (`up=15`)                                      |
| 2     | OmadaController, AmneziaWG, Xray | traefik (`up=5`)                                       |
| 3     | —                               | authelia (`up=10`)                                      |
| 4     | —                               | mgmt                                                    |
| 5     | —                               | organizer, immich, shares, open-webui, tdarr, forgejo   |
| 6     | —                               | vaultwarden, monitoring, gotify, onlyoffice, Dev        |
| 7     | —                               | jellyfin, arr, qbittorrent, frigate                     |
| 8     | —                               | homepage, yandex-disk                                   |
| 9     | —                               | Be-Free.Online                                          |

Hermes и Home-Assistant без `startup` — Proxmox запускает их после всех гостей с заданным порядком.

---

## 6. UPS и NUT

Электропитание защищено UPS (CyberPower UT2200EG), обслуживается через **NUT** (Network UPS Tools). UPS подключён по USB к PVE (драйвер `usbhid-ups`), который выступает NUT-primary (`upsd`, порт 3493); PVE-Mini и PBS — вторичные клиенты (`upsmon` в режиме secondary). При разряде батареи NUT через `upssched` инициирует graceful shutdown гостей в порядке, обратном запуску. Мониторинг UPS через веб-дашборд PeaNUT вынесен в контейнер monitoring (`15-monitoring.md`), который подключается к `upsd` на PVE.

Конфигурация NUT (`/etc/nut/`) входит в host backup хоста (см. `06-backup.md`).

---

## 7. Резервное копирование хостов

Гости (VM/LXC) бэкапятся в PBS автоматически (selection `All`). Сами хосты Proxmox бэкапятся отдельно — через PBS host backup: критичные конфиги `/etc/pve`, `/etc/network`, `/etc/nut`, `/root` и др. Полная схема бэкапов, retention и сценарий восстановления — в `06-backup.md`.

---

## 8. Файрвол хостов (nftables)

Оба гипервизора и PBS несут собственный per-host nftables-firewall — внутрисегментный слой защиты (L2) в дополнение к межсегментным правилам OPNsense. Это тот же whitelist-шаблон, что и у сервисных LXC (`policy drop` на `input`, разрешено только явно перечисленное), развёрнутый декларативно ролью `nftables` из Ansible (см. `16-ansible.md`); общая двухслойная модель фильтрации — в `04-firewall.md`.

Ключевое отличие от сервисных контейнеров касается цепочки `forward`. На гипервизорах она оставлена `policy accept`: они несут гостей с bridged-трафиком (на PVE-Mini через мосты проходит весь трафик OPNsense — WAN, DNS, DHCP, inter-VLAN), и `policy drop` на `forward` оборвал бы транзит гостей. На PBS гостей нет, поэтому `forward` остаётся `drop`. Host-firewall защищает только management-плоскость самого хоста (`input`), не вмешиваясь в трафик гостей.

Что разрешено во `input` сверх общего базлайна (SSH из MGMT и VPN, loopback, conntrack, базовые ICMP):

- **PVE-Mini** (`192.168.10.11`): `8006` (web/API) из MGMT, VPN, Traefik и Monitoring (pve-exporter); `9100` (node_exporter) от Monitoring.
- **PVE** (`192.168.10.12`): `8006` из MGMT, VPN, Traefik и Monitoring; `3493` (NUT upsd) от вторичных клиентов PVE-Mini и PBS и от PeaNUT в Monitoring; `9100` от Monitoring.
- **PBS** (`192.168.10.15`): `8007` (web/API) из MGMT, VPN, Traefik и Monitoring (pbs-exporter); `9100` от Monitoring.

Файловый доступ в сети обеспечивает Samba в контейнере shares (`192.168.50.36`); NFS на хостах не используется. Встроенный `pve-firewall` выключен — фильтрацию несёт `nftables.service`, и включение pve-firewall параллельно создало бы две конкурирующие системы правил.
