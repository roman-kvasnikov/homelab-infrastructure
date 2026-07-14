---
name: backup
description: |
  Трёхуровневая схема резервного копирования на единый бэкап-сервер (PBS): host-backup конфигов гипервизоров, PBS-снапшоты всех VM/LXC, и restic-снапшоты данных клиентских сервисов через rest-server. Документ описывает бэкап-сервер, identity-модель PBS, rest-server, retention и сценарий полного восстановления. Используй для вопросов по бэкапам, PBS, restic, восстановлению, retention.
---

# Резервное копирование

Бэкапы организованы по трёхуровневой схеме, где каждый уровень закрывает свою задачу подходящим инструментом. Всё складывается на один бэкап-сервер `192.168.10.15` (MGMT), в разные структуры на одном RAID0-массиве.

Три уровня: конфигурация гипервизоров через PBS host-backup (уровень 1), образы всех VM и LXC через PBS (уровень 2), данные стейтфул-сервисов через restic (уровень 3).

## 1. Бэкап-сервер

ОС — Proxmox Backup Server (поверх Debian). Помимо PBS, на сервере поднят `rest-server` для приёма restic-бэкапов от Vaultwarden, Authelia и DockerHost.

Дисковая структура: системный SSD (ext4, ОС и PBS) + два диска в mdadm RAID0 (`md0`, ext4, mountpoint `/mnt/data`).

```
/mnt/data/
├── pbs/
│   └── datastore/        # PBS datastore "Homelab"
│       ├── .chunks/      # deduplicated chunks
│       ├── ns/           # namespaces (pve, pve-mini)
│       ├── ct/           # LXC snapshots
│       ├── vm/           # VM snapshots
│       └── host/         # pve host backups
└── restic/
    ├── vaultwarden/      # restic repo for Vaultwarden LXC
    ├── authelia/         # restic repo for Authelia LXC
    └── dockerhost/       # restic repo for DockerHost VM
```

SSH ужесточён общим drop-in `10-hardening.conf` (см. `conventions.md`).

### 1.1. Identity-модель PBS

Доступ к PBS разделён по назначению между двумя user-account, каждый со своими токенами и узкими правами. Это даёт изоляцию (компрометация одного канала не затрагивает другой) и лёгкую расширяемость (новый токен под существующим user).

Текущее состояние:

| User | Токен | Path | Роль | Назначение |
| :--- | :--- | :--- | :--- | :--- |
| `backup@pbs` | — | `/datastore/Homelab` | `DatastoreBackup` | базовый доступ записи |
| `backup@pbs` | `pve-mini` | `/datastore/Homelab` | `DatastoreBackup` | бэкапы с PVE-Mini |
| `backup@pbs` | `pve` | `/datastore/Homelab` | `DatastoreBackup` | бэкапы с PVE |
| `monitoring@pbs` | — | `/` | `Audit` | read-only база |
| `monitoring@pbs` | `homepage` | `/` | `Audit` | виджет PBS в Homepage |
| `monitoring@pbs` | `pbs-exporter` | `/` | `Audit` | pbs-exporter в Prometheus |

`backup@pbs` отвечает только за запись бэкапов — под ним два токена, `pve-mini` (PVE-Mini) и `pve` (PVE-Main). `monitoring@pbs` отвечает только за read-only доступ мониторинга — токены `homepage` (виджет Homepage) и `pbs-exporter` (стек Prometheus), оба с ролью `Audit`.

**Privilege separation.** У токенов включён `privsep=1` (дефолт): эффективные права токена — пересечение прав user и токена. User — верхняя граница, токен может только сужать. Поэтому user-level ACL обязательны — без них токены получают пустые эффективные права и операции возвращают `permission check failed`.

**Append-only свойство.** Роль `DatastoreBackup` не включает `Datastore.Modify` и `Datastore.Prune`. Атакующий с токеном `pve-mini`/`pve` (через компрометацию гипервизора) может только создавать новые снапшоты — удаление, изменение, prune PBS возвращает 403 на уровне API. Retention выполняется только локально на PBS от root@pam — это не сетевой путь, недоступный клиентским токенам. Garbage Collection и Verify так же локальны.

### 1.2. rest-server для restic

`rest-server` — HTTP-сервис для restic-клиентов, приёмник бэкапов Vaultwarden, Authelia и DockerHost. Режим `--append-only` + `--private-repos`: клиент может писать новые снапшоты, но физически не может удалять/изменять существующие (DELETE → 403), и видит только свой репо.

Системный юзер `rest-server` (nologin). ExecStart: `--listen 192.168.10.15:8000` (только LAN-интерфейс), `--path /mnt/data/restic`, `--htpasswd-file /etc/rest-server/.htpasswd` (bcrypt), `--append-only`, `--private-repos`, `--log /var/log/rest-server/access.log`. Sandbox-набор из `conventions.md`, запись только в `/mnt/data/restic` и `/var/log/rest-server`.

HTTP-аутентификация: в `.htpasswd` три записи (`vaultwarden`, `authelia`, `dockerhost`), bcrypt. Транспорт HTTP без TLS — приемлемо в LAN, потому что содержимое снапшотов шифруется restic-клиентом до отправки (encryption-пароль по сети не идёт). Даже при перехвате HTTP basic-auth append-only не даёт уничтожить бэкапы — только записать мусорные новые (DoS, виден мониторингом).

### 1.3. Локальный restic для retention

Retention запускается на PBS под юзером `rest-server` (тем же, что владеет репозиториями). Это критично: когда `restic prune` перепаковывает packs и пересоздаёт index, новые файлы наследуют owner процесса. Под root новые index/pack оказались бы `root:root` 600, и rest-server (другой юзер) не смог бы их прочитать — следующий backup-клиент получал бы `500`. Единый owner на всём `/mnt/data/restic/` исключает этот класс проблем.

Скрипт `/usr/local/sbin/restic-retention.sh` проходит по всем трём репо с единой политикой: `--keep-daily 7 --keep-weekly 4 --keep-monthly 6 --group-by host,tags --prune`. Запускается systemd-таймером ежедневно в 03:00 с `RandomizedDelaySec=30min` (после backup-таймеров клиентов в 01:00). Encryption-пароли репо хранятся в `/etc/restic-retention/<service>.password`. Изоляция от компрометации клиента обеспечивается append-only режимом rest-server, а не разделением паролей (клиент технически не может выполнить DELETE независимо от наличия пароля).

## 2. Уровень 1: конфигурация гипервизоров (PBS host-backup)

**Что:** критичные конфиги самих гипервизоров — `/etc/pve` (конфиги VM/LXC, storage, users), `/etc/network`, `/etc/cron.d`, `/etc/nut`, `/root`, плюс `/etc/hosts`, `/etc/hostname`, `/etc/resolv.conf`.

**Чем:** `proxmox-backup-client` на каждом гипервизоре, токен `backup@pbs!pve` (PVE-Main) и `backup@pbs!pve-mini` (PVE-Mini), роль `DatastoreBackup`.

**Откуда:** скрипт `/usr/local/bin/pve-host-backup.sh` на каждом хосте, systemd-таймер, ежедневно, `Persistent=true`. Один и тот же скрипт поднят на обоих гипервизорах.

**Зачем отдельно:** PBS бэкапит гостей, но не сам Proxmox. При потере системного диска гипервизора host-backup сводит восстановление к `proxmox-backup-client restore` поверх свежей установки.

## 3. Уровень 2: VM и LXC (PBS)

**Что:** образы дисков всех VM и LXC + их конфиги.

**Чем:** PBS, datastore `Homelab` (`/mnt/data/pbs/datastore`), namespaces `pve` (гости PVE-Main) и `pve-mini` (гости PVE-Mini).

**Откуда:** инициируют оба гипервизора через storage `pbs` (токены `backup@pbs!pve` и `backup@pbs!pve-mini`, роль `DatastoreBackup` — append-only). Ежедневно, mode `snapshot`, compression `ZSTD`, selection `All`.

**Namespace + owner.** Бэкап-группы наследуют owner при создании. Каждый гипервизор пишет в свой namespace (`pve` / `pve-mini`) — это разделяет снапшоты по источнику и упрощает управление правами.

**Passthrough-диски DockerHost.** У DockerHost VM проброшены физические диски под ZFS-пулы — на каждом флаг `backup=0`, чтобы PBS не пытался бэкапить многотерабайтные пулы. PBS бэкапит только системный диск VM; данные ZFS — отдельно через restic (уровень 3) либо не бэкапятся (медиатека, frigate-записи).

## 4. Уровень 3: данные сервисов (restic)

PBS-снапшот целого LXC/VM закрывает «снёс целиком, восстановить за минуту». Но для сервисов с состоянием нужна конкретная версия файлов от N дней назад без подъёма всего хоста. Поэтому поверх PBS три сервиса (Vaultwarden, Authelia, DockerHost) делают независимый restic-бэкап данных.

**Транспорт.** rest-server на PBS в режиме `--append-only` + `--private-repos` (см. 1.2). Каждый сервис аутентифицируется своим HTTP basic-auth, видит только свой репо, не может удалять снапшоты.

**Backup.** Каждый сервис выполняет свой bash-скрипт (под dedicated юзером или root), systemd-таймер ежедневно в 01:00. Encryption данных своим паролем до отправки.

**Retention.** Централизованно на PBS под `rest-server`, единый скрипт, политика 7d/4w/6m (см. 1.3). Клиенты retention не выполняют — append-only физически блокирует DELETE с их стороны.

| Сервис | Что бэкапится | Тег/host | Репо |
| :--- | :--- | :--- | :--- |
| Vaultwarden | `/var/lib/vaultwarden/data/` (БД через online-снапшот, attachments, rsa_key) | `vaultwarden` | `rest:http://vaultwarden:...@192.168.10.15:8000/vaultwarden/` |
| Authelia | `/var/lib/authelia/` (БД online) + `/etc/authelia/` (конфиги, секреты) | `authelia` | `rest:http://authelia:...@192.168.10.15:8000/authelia/` |
| DockerHost | `/etc/samba/`, compose-файлы, `/mnt/data/` (Docker volumes + Samba shares) | `dockerhost` | `rest:http://dockerhost:...@192.168.10.15:8000/dockerhost/` |

Скрипты: `vaultwarden-backup.sh`, `authelia-backup.sh`, `dockerhost-backup.sh` (последний под root — бэкапит недоступные обычному юзеру каталоги). БД SQLite бэкапятся через online-снапшот `sqlite3 .backup` (консистентная копия работающей БД).

**Зачем отдельно от PBS:** гранулярное восстановление отдельных файлов за секунды; долгая retention (restic-снапшоты файлов весят мегабайты против гигабайт PBS-снапшотов дисков); эффективная файловая дедупликация для AppData.

## 5. Retention-политики

**PBS (VM/LXC и host-backup, единая):** Prune Job ежедневно — keep-last 3, keep-daily 7, keep-weekly 4, keep-monthly 6. Garbage Collection — воскресенье 07:00. Verify Job — воскресенье 08:00, skip-verified, re-verify after 30 дней.

**Restic (клиентские сервисы):** единая политика 7 daily / 4 weekly / 6 monthly + `--prune`, группировка `host,tags`, централизованно из `restic-retention.sh` на PBS, таймер ежедневно 03:00 + `RandomizedDelaySec=30min`. Клиенты retention не выполняют.

## 6. Сценарий полного восстановления

При потере системного диска гипервизора:

1. Установить свежий Proxmox на новый диск.
2. Настроить базовую сеть, чтобы видеть бэкап-сервер.
3. Подключить PBS storage `pbs` через токен.
4. Восстановить host-backup: `proxmox-backup-client restore` для `/etc/pve`, `/etc/network`, `/etc/nut` и прочих конфигов.
5. После перезапуска гипервизор видит все конфиги VM/LXC из восстановленного `/etc/pve`.
6. Восстановить VM/LXC из PBS-снапшотов (из нужного namespace `pve`/`pve-mini`) через UI.
7. Запустить DockerHost VM, сделать `zpool import` ZFS-пулов — метаданные на физических дисках живы, пулы поднимутся.
8. Запустить Docker-стек через compose-файлы.
9. При необходимости восстановить отдельные файлы из restic (локально с PBS: `restic -r /mnt/data/restic/<service>` с паролем из `/etc/restic-retention/<service>.password`).

## 7. Зависимости

- **PBS (`192.168.10.15`)** — цель всех бэкапов, хост rest-server и retention.
- **Гипервизоры (PVE-Mini, PVE)** — источники host-backup и PBS-снапшотов через токены `backup@pbs!pve-mini` / `backup@pbs!pve`.
- **Vaultwarden, Authelia, DockerHost** — restic-клиенты (уровень 3).
- **Monitoring** — pbs-exporter и restic-метрики следят за свежестью бэкапов (см. `monitoring.md`).
