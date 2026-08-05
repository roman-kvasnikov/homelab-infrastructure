---
name: backup
description: |
  Схема резервного копирования на бэкап-сервер PBS: host-backup конфигов гипервизоров и PBS-снапшоты всех VM/LXC. Данные stateful-сервисов защищаются через managed-volume в составе PBS-снапшота плюс application-consistent SQLite-снапшот. Документ описывает бэкап-сервер, identity-модель PBS, паттерн консистентности данных, retention и сценарий полного восстановления. Используй для вопросов по бэкапам, PBS, восстановлению, retention.
---

# Резервное копирование

Бэкапы организованы по двухуровневой схеме, где каждый уровень закрывает свою задачу. Всё складывается на один бэкап-сервер `192.168.10.15` (MGMT).

Два уровня: конфигурация гипервизоров через PBS host-backup (уровень 1) и образы всех VM и LXC через PBS-снапшоты (уровень 2). Данные stateful-сервисов не выделяются в отдельный инструмент — они лежат на managed-volume, который попадает в PBS-снапшот LXC вместе с rootfs; консистентность БД обеспечивается application-consistent снапшотом рядом с боевой базой (см. раздел 4).

## 1. Бэкап-сервер

ОС — Proxmox Backup Server (поверх Debian). Управляющий адрес `192.168.10.15` (MGMT). SSH ужесточён общим drop-in `10-hardening.conf` (см. `02-conventions.md`).

Datastore `main` расположен на дисковом хранилище сервера. Структура PBS-datastore:

```
main/
├── .chunks/      # deduplicated chunks
├── ns/pve/       # namespace "pve" — снапшоты гостей и host-backup
├── ct/           # LXC snapshots
├── vm/           # VM snapshots
└── host/         # host backups гипервизоров
```

### 1.1. Identity-модель PBS

Доступ к PBS разделён по назначению между двумя user-account, каждый со своими токенами и узкими правами. Это даёт изоляцию (компрометация одного канала не затрагивает другой) и лёгкую расширяемость (новый токен под существующим user).

| User             | Токен          | Path              | Роль              | Назначение                |
| :--------------- | :------------- | :---------------- | :---------------- | :------------------------ |
| `backup@pbs`     | —              | `/datastore/main` | `DatastoreBackup` | базовый доступ записи     |
| `backup@pbs`     | `pve`          | `/datastore/main` | `DatastoreBackup` | бэкапы с PVE              |
| `backup@pbs`     | `pve-mini`     | `/datastore/main` | `DatastoreBackup` | бэкапы с PVE-Mini         |
| `monitoring@pbs` | —              | `/`               | `Audit`           | read-only база            |
| `monitoring@pbs` | `homepage`     | `/`               | `Audit`           | виджет PBS в Homepage     |
| `monitoring@pbs` | `pbs-exporter` | `/`               | `Audit`           | pbs-exporter в Prometheus |

`backup@pbs` отвечает только за запись бэкапов — под ним два токена, `pve` (PVE) и `pve-mini` (PVE-Mini). `monitoring@pbs` отвечает только за read-only доступ мониторинга — токены `homepage` (виджет Homepage) и `pbs-exporter` (стек Prometheus), оба с ролью `Audit`.

**Privilege separation.** У токенов включён `privsep=1` (дефолт): эффективные права токена — пересечение прав user и токена. User — верхняя граница, токен может только сужать. Поэтому user-level ACL обязательны — без них токены получают пустые эффективные права и операции возвращают `permission check failed`.

**Append-only свойство.** Роль `DatastoreBackup` не включает `Datastore.Modify` и `Datastore.Prune`. Атакующий с токеном `pve`/`pve-mini` (через компрометацию гипервизора) может только создавать новые снапшоты — удаление, изменение, prune PBS возвращает 403 на уровне API. Retention выполняется только локально на PBS от `root@pam` — это не сетевой путь, недоступный клиентским токенам. Garbage Collection и Verify так же локальны.

## 2. Уровень 1: конфигурация гипервизоров (PBS host-backup)

**Что:** критичные конфиги самих гипервизоров — `/etc/pve` (конфиги VM/LXC, storage, users), `/etc/network`, `/etc/cron.d`, `/etc/nut`, `/root`, плюс `/etc/hosts`, `/etc/hostname`, `/etc/resolv.conf`.

**Чем:** `proxmox-backup-client` на каждом гипервизоре, токен `backup@pbs!pve` (PVE) и `backup@pbs!pve-mini` (PVE-Mini), роль `DatastoreBackup`.

**Откуда:** скрипт `/usr/local/bin/pve-host-backup.sh` на каждом хосте, systemd-таймер, ежедневно, `Persistent=true`. Один и тот же скрипт поднят на обоих гипервизорах.

**Зачем отдельно:** PBS бэкапит гостей, но не сам Proxmox. При потере системного диска гипервизора host-backup сводит восстановление к `proxmox-backup-client restore` поверх свежей установки.

## 3. Уровень 2: VM и LXC (PBS)

**Что:** образы дисков всех VM и LXC + их конфиги.

**Чем:** PBS, datastore `main`, namespace `pve`.

**Откуда:** инициируют оба гипервизора через storage `pbs-main` (токены `backup@pbs!pve` и `backup@pbs!pve-mini`, роль `DatastoreBackup` — append-only). Ежедневно, mode `snapshot`, compression `ZSTD`, selection `All`.

**Что в снапшот не попадает.** Крупные датасеты монтируются в гостей bind-mount'ом с флагом `backup=0` и в PBS-снапшот не входят: медиатека (`zmedia`), записи камер (`zfrigate`), файловые шары (`zdata/Shares`), фотоархив Immich. Managed-volume'ы с важными данными сервисов (`zdata:...`) флага `backup=0` не несут и попадают в снапшот вместе с rootfs гостя.

## 4. Данные stateful-сервисов

PBS-снапшот целого LXC закрывает сценарий «снёс целиком, восстановить за минуту». Данные сервиса, которые нужно резервировать, размещаются на managed-volume пула `zdata` — такой том входит в PBS-снапшот вместе с rootfs, поэтому восстановление LXC возвращает сервис вместе с его состоянием. Отдельного инструмента для данных не требуется.

**Консистентность SQLite.** Снапшот снимает файловую систему в один момент, но живой SQLite может быть в середине транзакции с неслитым WAL. Для сервисов на SQLite рядом с боевой базой поддерживается application-consistent копия: systemd-таймер `<service>-db-backup.timer` ежечасно запускает `sqlite3 db.sqlite3 ".backup db.sqlite3.bak"` — атомарный снапшот, безопасный на работающем сервисе. PBS-снапшот захватывает свежий `.bak` в консистентном виде; при восстановлении из него берётся рабочая база (переименование `db.sqlite3.bak` → `db.sqlite3` перед стартом сервиса). Паттерн и восстановление детально описаны в `02-conventions.md`.

**Важные данные на надёжном пуле.** Managed-volume'ы сервисов лежат на `zdata` (RAIDZ1, переживает отказ одного диска), а не на одиночном NVMe rootfs. Vaultwarden и Authelia держат на `zdata` не только данные (`/var/lib/<service>`), но и конфиги (`/etc/vaultwarden`, `/etc/authelia` — отдельным managed-volume), чтобы всё критичное было на отказоустойчивом пуле и в PBS-снапшоте. Детали — в `11-authelia.md` и `12-vaultwarden.md`.

## 5. Retention-политики

**PBS (VM/LXC и host-backup, единая):** Prune Job ежедневно — keep-last 3, keep-daily 7, keep-weekly 4, keep-monthly 6. Garbage Collection — воскресенье 07:00. Verify Job — воскресенье 08:00, skip-verified, re-verify after 30 дней.

Retention выполняется локально на PBS от `root@pam`; клиентские токены гипервизоров (append-only) prune не выполняют и физически не могут.

## 6. Сценарий полного восстановления

При потере системного диска гипервизора:

1. Установить свежий Proxmox на новый диск.
2. Настроить базовую сеть, чтобы видеть бэкап-сервер.
3. Подключить PBS storage `pbs-main` через токен.
4. Восстановить host-backup: `proxmox-backup-client restore` для `/etc/pve`, `/etc/network`, `/etc/nut` и прочих конфигов.
5. После перезапуска гипервизор видит все конфиги VM/LXC из восстановленного `/etc/pve`.
6. Восстановить VM/LXC из PBS-снапшотов (namespace `pve`) через UI.
7. Импортировать ZFS-пулы данных (`zpool import`) — метаданные на дисках живы, пулы поднимутся; проверить синхронность mountpoint пулов и `storage.cfg`.
8. Для сервиса на SQLite после восстановления взять консистентную базу из `db.sqlite3.bak`.

## 7. Зависимости

- **PBS (`192.168.10.15`)** — цель всех бэкапов, хост retention.
- **Гипервизоры (PVE-Mini, PVE)** — источники host-backup и PBS-снапшотов через токены `backup@pbs!pve-mini` / `backup@pbs!pve`.
- **Monitoring** — pbs-exporter следит за свежестью бэкапов (см. `15-monitoring.md`).
