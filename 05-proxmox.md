---
name: proxmox
description: |
  Вычислительный слой homelab: два хоста Proxmox VE. PVE-Mini несёт сетевую плоскость (OPNsense, Omada, AmneziaWG, Xray), PVE — прикладные сервисы. Документ описывает сеть хостов, storage и ZFS, лимиты ресурсов гостей, UPS/NUT, host-firewall (nftables) и инвентарь VM/LXC.
---

# Гипервизоры Proxmox

Вычислительная нагрузка распределена на два хоста Proxmox VE. Разделение по роли: **PVE-Mini** несёт сетевую плоскость (маршрутизатор, контроллер коммутатора, VPN, прокси), **PVE** — прикладные сервисы (reverse-proxy, приложения, мониторинг). Оба хоста управляются в MGMT (VLAN 10); все гости получают VLAN тегированием на виртуальных интерфейсах.

Общие соглашения по LXC (unprivileged + `nesting=1`, sandbox, nftables) вынесены в `02-conventions.md` и здесь не повторяются.

---

## 1. PVE-Mini

Мини-ПК на Intel N150 с двумя сетевыми интерфейсами. Несёт сетевую плоскость дома: на нём запущен OPNsense, через который проходит весь трафик сети. Управляющий адрес хоста — `192.168.10.11` (MGMT).

### Сеть хоста

Два физических интерфейса разведены по ролям — один под WAN, второй под LAN-транк со всеми VLAN.

- **`nic0` (`enxe43a6e8558c1`) → `vmbr0`** — WAN. Мост без IP на хосте, чистый проброс физического интерфейса в WAN виртуальной машины OPNsense. В этот интерфейс заходит кабель провайдера.
- **`nic1` (`enxe43a6e8558c2`) → `vmbr1`** — LAN-транк. VLAN-aware мост (`bridge-vlan-aware yes`, `bridge-vids 2-4094`), несёт все VLAN тегированными. Подключён в транк-порт коммутатора (порт 8).
- **`vmbr1.10`** — VLAN-подынтерфейс поверх транка, управляющий адрес хоста `192.168.10.11/24` (MGMT), шлюз `192.168.10.1`. Через него же идёт default route хоста.

Хост не имеет IP на `vmbr0` — WAN целиком принадлежит OPNsense. Управление и маршрутизация самого хоста идут через `vmbr1.10` в MGMT.

### Гости

| CTID/VMID | Гость            | Тип | Сегмент        | Адрес                   |
| :-------- | :--------------- | :-- | :------------- | :---------------------- |
| 101       | OPNsense         | VM  | WAN + все VLAN | шлюз `.1` в каждом VLAN |
| 102       | Omada Controller | LXC | MGMT (10)      | `192.168.10.31`         |
| 103       | AmneziaWG        | LXC | INFRA (20)     | `192.168.20.11`         |
| 104       | Xray             | VM  | INFRA (20)     | `192.168.20.12`         |

**OPNsense (101)** — маршрутизатор и firewall сети. `net0` → `vmbr0` (WAN), `net1` → `vmbr1` (LAN-транк со всеми VLAN). CPU type `host`, virtio-сеть, hardware offload отключён. Детали — в `03-network.md` и `09-amneziawg.md`.

**Omada Controller (102)** — контроллер управляемого коммутатора TP-Link. Одна нога в MGMT.

**AmneziaWG (103)** — обфусцированный VPN-сервер удалённого доступа. Непривилегированный LXC с пробросом TUN-устройства. Детали — в `09-amneziawg.md`.

**Xray (104)** — прокси для гео-обхода. Детали — в `10-xray.md`.

---

## 2. PVE

Основной сервер, несёт прикладные сервисы. Подключён транком в порт 5 коммутатора. Управляющий адрес хоста — `192.168.10.12` (MGMT). Сетевой мост VLAN-aware; каждый гость получает свой VLAN тегом на виртуальном интерфейсе.

### Гости

| CTID/VMID | Гость          | Тип | Сегмент       | Адрес           |
| :-------- | :------------- | :-- | :------------ | :-------------- |
| 199  | Ansible        | LXC | MGMT (10)     | `192.168.10.99` |
| 220  | Homepage       | LXC | INFRA (20)    | `192.168.20.20` |
| 115  | Traefik        | LXC | DMZ (40)      | `192.168.40.11` |
| 117  | Vaultwarden    | LXC | SERVICES (50) | `192.168.50.11` |
| 118  | Authelia       | LXC | SERVICES (50) | `192.168.50.12` |
| 119  | Monitoring     | LXC | SERVICES (50) | `192.168.50.21` |
| 116  | Gotify         | LXC | SERVICES (50) | `192.168.50.22` |
| 531  | Jellyfin       | LXC | SERVICES (50) | `192.168.50.31` |
| 532  | Arr            | LXC | SERVICES (50) | `192.168.50.32` |
| 533  | qBittorrent    | LXC | SERVICES (50) | `192.168.50.33` |
| 534  | Organizer      | LXC | SERVICES (50) | `192.168.50.34` |
| 535  | Immich         | LXC | SERVICES (50) | `192.168.50.35` |
| 536  | Shares         | LXC | SERVICES (50) | `192.168.50.36` |
| 537  | Frigate        | LXC | SERVICES (50) | `192.168.50.37` |
| 538  | OnlyOffice     | LXC | SERVICES (50) | `192.168.50.38` |
| 580  | YandexDisk     | LXC | SERVICES (50) | `192.168.50.80` |
| 590  | PostgreSQL     | LXC | SERVICES (50) | `192.168.50.90` |
| 140  | Dev            | VM  | SERVICES (50) | `192.168.50.40` |
| 150  | Be-Free.Online | VM  | SERVICES (50) | `192.168.50.50` |

Детали каждого сервиса — в соответствующих документах (`08-traefik.md`, `11-authelia.md`, `12-vaultwarden.md`, `13-gotify.md`, `14-media-stack.md`, `15-monitoring.md`).

---

## 3. Storage и ZFS

### PVE

| Storage   | Тип     | Носитель             | Назначение                          |
| :-------- | :------ | :------------------- | :---------------------------------- |
| `local`   | dir     | SATA SSD, `pve-root` | ISO, шаблоны LXC, vzdump-дампы      |
| `zguests` | zfspool | NVMe                 | Виртуальные диски всех VM/LXC       |
| `zdata`   | zfspool | RAIDZ1 (4× HDD)      | Managed-volume данных сервисов      |
| `zssd`    | zfspool | SATA SSD             | Резервный SSD-пул                   |
| `pbs-main`| pbs     | PBS `192.168.10.15`  | Бэкап-цель для VM/LXC и host backup |

Пять ZFS-пулов, каждый монтируется в корень по своему имени.

```
SATA SSD (система):
├── EFI boot partition
├── pve-swap
└── pve-root  → /, и /var/lib/vz (storage `local`)

SATA SSD (резерв):
└── zssd → /zssd (ZFS pool, lz4, atime=off)

NVMe:
└── zguests → /zguests (ZFS pool, lz4, ashift=12)
    ├── system-диски (rootfs) всех LXC
    └── system-диски VM (Dev, Be-Free.Online)

HDD:
├── zdata → /zdata (RAIDZ1, 4 диска, lz4) — managed-volume данных сервисов
├── zmedia → /zmedia (одиночный диск) — медиатека
└── zfrigate → /zfrigate (одиночный диск) — записи камер
```

Пулы созданы с `compression=lz4` (кроме `zmedia`, где сжатие отключено — медиафайлы уже сжаты) и `atime=off`. **ZFS ARC ограничен 4 GB** (`/etc/modprobe.d/zfs.conf`: `options zfs zfs_arc_max=4294967296`), чтобы кэш файловой системы не конкурировал за память с гостями.

**Разделение по надёжности.** `zdata` (RAIDZ1 из четырёх дисков — переживает отказ одного) несёт важные данные, которые нельзя терять: managed-volume'ы сервисов (БД, конфиги). Rootfs гостей на одиночном NVMe (`zguests`) — восстановимы из PBS. Медиатека (`zmedia`) и записи камер (`zfrigate`) на одиночных дисках — крупные, восстановимые/некритичные данные.

**Managed-volume против bind-mount.** Данные, которые нужно бэкапить, размещаются на `zdata` как Proxmox-managed volume (`zdata:SIZE`, попадает в vzdump). Крупные датасеты — медиатека, фото, записи камер, файловые шары — монтируются bind-mount'ом обычного датасета с флагом `backup=0`, чтобы не гнать терабайты в PBS-снапшот.

`storage.cfg` держит mountpoint пула синхронно с ZFS-свойством `mountpoint`: смена точки монтирования пула требует одновременного обновления обоих, иначе LXC с томами на этом пуле не стартуют.

### PBS

Целевое хранилище бэкапов — Proxmox Backup Server на `192.168.10.15` (MGMT). Storage `pbs-main`, datastore `main`, namespace `pve`, токен `backup@pbs!pve`. Детали PBS и retention — в `06-backup.md`.

---

## 4. iGPU и хранилище данных

Пулы данных (`zdata`, `zmedia`, `zfrigate`) и iGPU принадлежат хосту PVE напрямую и раздаются в LXC, а не пробрасываются в отдельную VM.

**iGPU.** Intel iGPU управляется хостовым драйвером `i915`; устройства `/dev/dri/card0` (группа `video`) и `/dev/dri/renderD128` (группа `render`) разделяются между несколькими unprivileged LXC через idmap: Jellyfin (531, VAAPI-транскод, QuickSync), Immich (535) и Frigate (537). Аппаратный **транскод видео** (VAAPI) в unprivileged LXC работает; GPU-**compute** (OpenVINO inference для ML Immich и детекции Frigate) в unprivileged-контейнере не инициализируется, поэтому Immich и Frigate используют CPU. Детали GPU-проброса в медиа-LXC — в `14-media-stack.md`.

**Хранилище данных.** Медиатека, фото, записи камер и файловые шары лежат на HDD-пулах (`zmedia`, `zfrigate`, `zdata/Shares`) и монтируются в соответствующие LXC bind-mount'ом с `backup=0`. Связка qBittorrent → \*arr → Jellyfin работает на едином датасете `zmedia` (hardlinks требуют одной файловой системы) — все три сервиса монтируют `/zmedia` и его подкаталоги. Managed-volume'ы сервисов с важными данными — на `zdata`, попадают в vzdump.

---

## 5. Лимиты ресурсов и порядок запуска

Чтобы избежать борьбы за CPU/RAM при одновременном старте, гостям заданы лимиты RAM и порядок запуска через `startup` (order/up/down).

Принцип порядка: инфраструктурные гости стартуют первыми, прикладные — следом. Порядок ролей: PostgreSQL (общая БД) → Traefik → Authelia → Ansible → сервисы с зависимостью от БД → мониторинг и вспомогательные → медиастек → остальное. Обратный порядок применяется при выключении, в том числе по сигналу от UPS. ZFS ARC на PVE ограничен 4 GB (см. раздел 3), чтобы кэш файловой системы не конкурировал с памятью гостей.

---

## 6. UPS и NUT

Электропитание защищено UPS (CyberPower, протокол Q1), обслуживается через **NUT** (Network UPS Tools). UPS подключён по USB к PVE, который выступает NUT-primary (`upsd`, порт 3493); PVE-Mini — вторичный клиент. При разряде батареи NUT через `upssched` инициирует graceful shutdown гостей в порядке, обратном запуску. Мониторинг UPS через веб-дашборд PeaNUT вынесен в контейнер Monitoring (`15-monitoring.md`), который подключается к `upsd` на PVE.

Конфигурация NUT (`/etc/nut/`) входит в host backup хоста (см. `06-backup.md`).

---

## 7. Резервное копирование хостов

Гости (VM/LXC) бэкапятся в PBS автоматически (selection `All`). Сами хосты Proxmox бэкапятся отдельно — через PBS host backup: критичные конфиги `/etc/pve`, `/etc/network`, `/etc/nut`, `/root` и др. PBS не бэкапит сам себя как хост, поэтому конфиги хостов сохраняются отдельным заданием. Полная схема бэкапов, retention и сценарий восстановления — в `06-backup.md`.

---

## 8. Файрвол хостов (nftables)

Оба гипервизора и PBS несут собственный per-host nftables-firewall — внутрисегментный слой защиты (L2) в дополнение к межсегментным правилам OPNsense. Это тот же whitelist-шаблон, что и у сервисных LXC (`policy drop` на `input`, разрешено только явно перечисленное), развёрнутый декларативно ролью `nftables` из Ansible (см. `16-ansible.md`); общая двухслойная модель фильтрации — в `04-firewall.md`.

Ключевое отличие от сервисных контейнеров касается цепочки `forward`. На гипервизорах она оставлена `policy accept`: они несут гостей с bridged-трафиком (на PVE-Mini через мосты проходит весь трафик OPNsense — WAN, DNS, DHCP, inter-VLAN), и `policy drop` на `forward` оборвал бы транзит гостей. На PBS гостей нет, поэтому `forward` остаётся `drop`. Host-firewall в любом случае защищает только management-плоскость самого хоста (`input`), не вмешиваясь в трафик гостей.

Что разрешено во `input` сверх общего базлайна (SSH из MGMT и VPN, loopback, conntrack, базовые ICMP):

- **PVE-Mini** (`192.168.10.11`): `8006` (web/API) из MGMT, VPN, Traefik и Monitoring (pve-exporter); `9100` (node_exporter) от Monitoring.
- **PVE** (`192.168.10.12`): `8006` из MGMT, VPN, Traefik и Monitoring; `3493` (NUT upsd) от вторичного клиента PVE-Mini и от PeaNUT в Monitoring; `9100` от Monitoring.
- **PBS** (`192.168.10.15`): `8007` (web/API) из MGMT, VPN, Traefik и Monitoring (pbs-exporter); `9100` от Monitoring.

Файловый доступ в сети обеспечивает Samba в контейнере Shares (`192.168.50.36`, см. `14-media-stack.md`); NFS на хостах не используется. Встроенный `pve-firewall` выключен — фильтрацию несёт `nftables.service`, и включение pve-firewall параллельно создало бы две конкурирующие системы правил.
