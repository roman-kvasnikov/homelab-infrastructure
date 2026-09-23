---
name: conventions
description: |
  Общие соглашения и шаблоны для развёртывания сервисов homelab: базлайн unprivileged LXC, шаблон nftables, единый systemd-sandbox, hardening SSH, паттерн бэкапа данных сервиса, соглашения об именах и общие зависимости. Сервисные документы ссылаются сюда и описывают только свою специфику.
---

# Соглашения и шаблоны

Этот документ — единый источник для повторяющихся конструкций инфраструктуры. Сервисные файлы не дублируют эти блоки, а ссылаются на соответствующий раздел и описывают только специфику сервиса: имя, адрес, порт, конфиг, зависимости и отклонения от шаблона.

Плейсхолдеры в шаблонах записаны в угловых скобках (`<service>`, `<SERVICE_PORT>`) и подставляются под конкретный сервис.

---

## 1. Базлайн LXC

Все сервисные LXC — **unprivileged** (`unprivileged: 1`): root внутри контейнера маппится на непривилегированный UID хоста через user namespaces.

Один LXC — один сервис.

### Нативный LXC

Сервис работает нативным бинарником под systemd. Features: **`nesting=1`** — нужен современному systemd для собственного sandbox'а юнитов (mount/UTS namespaces, `ProtectProc`, `PrivateTmp` и т.п.). Остальные features не включаются.

### Docker-in-LXC

Для сервисов без вменяемой нативной установки. Features: **`nesting=1,keyctl=1`**. В `/etc/docker/daemon.json` задаётся `default-address-pools` из диапазона `10.200.0.0/16` (подсети `/24`), чтобы Docker-сети не пересекались с подсетями homelab:

```json
{
  "default-address-pools": [
    {"base": "10.200.0.0/16", "size": 24}
  ]
}
```

Порты контейнеров публикуются только на адресе LXC (`192.168.50.xx:<port>:<port>`), а не на `0.0.0.0`. Фильтрация опубликованных портов описана в разделе 2.

### Сеть

Интерфейс контейнера подключается к VLAN-aware мосту гипервизора с тегом VLAN, выбранного по роли сервиса (`03-network.md`); исключение — гости в MGMT на PVE, где MGMT приходит native и тег не ставится. Флаг `firewall=1` на интерфейсе не используется (`05-proxmox.md`). Адрес статический и выводится из ID гостя (`540` → `192.168.50.40`), шлюз и DNS — OPNsense в соответствующем VLAN (`192.168.<vlan>.1`).

Сервис слушает только на своём адресе, не на `0.0.0.0`.

### Хранилище

Rootfs лежит на `local-zfs`. Данные, которые нужно бэкапить, размещаются на Proxmox-managed volume (раздел 5): малое критичное состояние — на `local-zfs`, крупные ценные данные — на `zdata`. Крупные восстановимые медиа и записи — на bind-mount датасета, вне vzdump. Принцип размещения — в `05-proxmox.md`.

---

## 2. Шаблон nftables

Фильтрация на всех LXC и VM построена по whitelist-принципу: `policy drop` на `input` и `forward`, разрешено только явно перечисленное. Исходящий трафик не ограничивается. Конфиг `/etc/nftables.conf` генерируется ролью Ansible `nftables` (`16-ansible.md`) и загружается `nftables.service` при старте. Ручные правки не делаются — файл перезаписывается при следующей раскатке.

Блок `define` в каждом конфиге одинаков: весь адресный инвентарь из `group_vars/all/network.yml`. Имена строятся как `<СЕГМЕНТ>_NET` для подсети и `<СЕГМЕНТ>_<ИМЯ>_IP` для узла (`MGMT_NET`, `DMZ_TRAEFIK_IP`, `SERVICES_POSTGRES_IP`). Правила хоста ссылаются только на эти имена.

Стандартный сервис принимает подключения к своему порту от **Traefik**, SSH — от трёх управляющих узлов: административного ноутбука (из MGMT и через VPN) и управляющего LXC mgmt. Между сервисами различаются только строки правил сервиса.

```nft
#!/usr/sbin/nft -f

# Managed by Ansible - do not edit by hand.
# Source of truth: inventory vars for <service> + group_vars/all/network.yml

flush ruleset

define VPN_MGMT_NOTEBOOK_IP = 10.8.0.2
define MGMT_NOTEBOOK_IP = 192.168.10.50
define MGMT_MGMT_IP    = 192.168.10.99
define DMZ_TRAEFIK_IP  = 192.168.40.11
# ... full address inventory

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;

        # Loopback
        iifname "lo" accept

        # Conntrack
        ct state established,related accept
        ct state invalid drop

        # ICMPv4 - ping and path MTU discovery
        ip protocol icmp icmp type {
            destination-unreachable,
            time-exceeded,
            parameter-problem,
            echo-request
        } accept

        # ICMPv6 - ping, PMTUD and NDP
        ip6 nexthdr icmpv6 icmpv6 type {
            destination-unreachable,
            packet-too-big,
            time-exceeded,
            parameter-problem,
            echo-request,
            nd-router-advert,
            nd-neighbor-solicit,
            nd-neighbor-advert
        } accept

        # SSH connections
        tcp dport 22 ip saddr { $MGMT_NOTEBOOK_IP, $VPN_MGMT_NOTEBOOK_IP, $MGMT_MGMT_IP } accept

        # <Service> WebUI from Traefik
        tcp dport <SERVICE_PORT> ip saddr $DMZ_TRAEFIK_IP accept

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

Проверка на хосте после раскатки: `nft list ruleset`.

### Расширение шаблона

Сервис, которому нужны дополнительные источники или порты, получает свои правила в `nft_service_rules` своего host_vars: каждое правило — порт, протокол, источник и комментарий, рендерится отдельной `accept`-строкой в `input`. Дополнительный источник добавляется только под реальный поток (например, Homepage для виджета) и описывается в файле сервиса с обоснованием.

Хосты, которые маршрутизируют трафик (AmneziaWG, Xray) или несут гостей (гипервизоры), переопределяют политику `forward` и добавляют собственные правила; это описано в их документах.

### Docker-in-LXC

Опубликованный порт Docker принимает трафик через DNAT: пакет проходит `prerouting` → `forward` → контейнер и цепочку `input` не пересекает. Поэтому на Docker-хостах (`nft_docker_host: true`) whitelist дублируется в `forward`, а сама цепочка остаётся в `policy drop`:

```nft
    chain forward {
        type filter hook forward priority filter; policy drop;

        # Conntrack
        ct state established,related accept
        ct state invalid drop

        # Container egress (internet, DNS, backends)
        iifname "docker0" accept
        iifname "br-*" accept

        # <Service> WebUI from Traefik (published port, pre-DNAT)
        meta l4proto tcp ct status dnat ct original proto-dst <SERVICE_PORT> ip saddr $DMZ_TRAEFIK_IP accept
    }
```

Каждое правило из `nft_service_rules` рендерится в обе цепочки: в `input` — по `dport`, в `forward` — по исходному порту до трансляции (`ct original proto-dst`) и только для DNAT-трафика (`ct status dnat`). Правило описывается в inventory один раз. Исходящий трафик контейнеров (интернет, DNS, PostgreSQL) разрешён по входному интерфейсу Docker-моста. Контейнеры с `network_mode: host` в `forward` не участвуют — их порты защищает `input`.

`flush ruleset` удаляет NAT-цепочки, созданные Docker, а Docker пересоздаёт их только при старте демона, поэтому после каждой перезагрузки nftables на Docker-хосте роль перезапускает Docker.

---

## 3. systemd-sandbox

Нативный сервис запускается от выделенного системного юзера с единым набором sandbox-директив. Набор подобран под unprivileged LXC: всё, что требует BPF или вложенного user namespace, в нём не используется.

```ini
[Service]
User=<service>
Group=<service>

Restart=always
RestartSec=10

UMask=0077

# --- Privileges ---
NoNewPrivileges=true
CapabilityBoundingSet=
AmbientCapabilities=
RestrictSUIDSGID=true
RestrictRealtime=true
LockPersonality=true

# --- Filesystem ---
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
PrivateDevices=true
ReadWritePaths=/var/lib/<service>

# --- Kernel / system ---
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectKernelLogs=true
ProtectControlGroups=true
ProtectClock=true
ProtectHostname=true
ProtectProc=invisible

# --- Namespaces / IPC / address families ---
RestrictNamespaces=true
RemoveIPC=true
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6 AF_NETLINK

# --- Syscalls ---
SystemCallArchitectures=native
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources
SystemCallErrorNumber=EPERM
```

Файловая система процесса read-only, кроме `ReadWritePaths` — обычно это единственный `/var/lib/<service>`. Логи идут в journald (`journalctl -u <service>`), файлового логирования нет.

`SystemCallErrorNumber=EPERM` возвращает ошибку на запрещённый syscall вместо завершения процесса через SIGSYS.

### Применение

Набор оформляется drop-in'ом `/etc/systemd/system/<service>.service.d/hardening.conf`, а не правкой основного unit-файла — так изменения переживают обновление пакета. Если основной юнит уже задаёт `User`, `Group`, `ExecStart` или `Restart`, в drop-in они не дублируются.

Сервис, которому набор мешает, расширяет или ослабляет его в том же drop-in. Отклонения описываются в файле сервиса.

После изменений:

```bash
systemctl daemon-reload
systemctl restart <service>
systemctl cat <service>
systemd-analyze security <service>
```

### Зависимости и автозапуск

Зависимость от локальных компонентов объявляется через `Requires=` + `After=`. Все сервисы включены в автозапуск (`systemctl enable`).

---

## 4. Hardening SSH

На всех хостах sshd ужесточается drop-in'ом `/etc/ssh/sshd_config.d/10-hardening.conf`, который раскатывает роль Ansible `ssh_hardening` (`16-ansible.md`). Основной `/etc/ssh/sshd_config` не изменяется.

```
PermitRootLogin prohibit-password
PasswordAuthentication no
KbdInteractiveAuthentication no
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```

Аутентификация — только по ed25519-ключам. Сетевой доступ к SSH ограничен nftables (раздел 2).

Проверка перед перезапуском: `sshd -t`. Во время изменений держать открытой вторую SSH-сессию.

---

## 5. Бэкап данных сервиса

Все LXC бэкапятся целиком через vzdump в PBS (`06-backup.md`).

### Managed-volume под данные

Данные сервиса (БД, конфиг, состояние) размещаются на отдельном Proxmox-managed volume, а не на rootfs. Mount point добавляется с **явным `backup=1`** — без него том в vzdump не попадает.

```bash
pct set <vmid> -mp0 <storage>:<size>,mp=/var/lib/<service>,backup=1
```

Внутри контейнера владелец точки монтирования меняется на юзера сервиса: `chown <service>:<service> /var/lib/<service>`.

Bind-mount'ы с крупными данными добавляются без `backup=1` и в vzdump не входят.

### Консистентность SQLite

Для сервисов на SQLite рядом с рабочей базой поддерживается консистентная копия. Таймер `<service>-db-backup.timer` ежечасно запускает oneshot `<service>-db-backup.service`, который выполняет `sqlite3 db.sqlite3 ".backup db.sqlite3.bak"`. Vzdump захватывает готовый `.bak` в консистентном состоянии.

Oneshot-юнит работает от юзера сервиса и несёт тот же sandbox-набор (раздел 3).

### Восстановление

LXC поднимается из PBS-снапшота. Для SQLite-сервиса перед стартом рабочая база и её WAL/`-shm` удаляются, а `db.sqlite3.bak` переименовывается в `db.sqlite3`.

---

## 6. Соглашения об именах

- **kebab-case** — hostname, имена гостей Proxmox (LXC и VM), имена файлов, systemd-юнитов, storage ID, идентификаторы сервисов.
- **snake_case** — переменные в bash-скриптах.
- **UPPER_SNAKE_CASE** — ENV-константы и `define` в nftables (`<СЕГМЕНТ>_<ИМЯ>_IP`, раздел 2).
- **snake_case с суффиксом типа** — алиасы OPNsense (`traefik_lxc`, `dev_vm`, `pve_host`, `xray_ports`): дефис в именах алиасов недопустим (`04-firewall.md`).
- Без type-префиксов: `vaultwarden`, а не `lxc-vaultwarden`.
- Одно имя сервиса для всех связанных сущностей: hostname, юзер, юнит, директория данных, DNS-имя.

Документация — на русском. Комментарии в конфигах, коде и заголовки алертов — на английском.

---

## 7. Общие зависимости сервисов

Типовой сервисный LXC зависит от четырёх внешних узлов. В файле сервиса перечисляются только дополнительные зависимости.

- **Traefik** (`192.168.40.11`, DMZ) — единственный источник запросов к порту сервиса.
- **Unbound на OPNsense** (шлюз VLAN сервиса) — DNS, включая split-horizon `*.kvasok.xyz → 192.168.40.11`.
- **PBS** (`192.168.10.15`, MGMT) — приёмник vzdump-снапшотов.
- **mgmt** (`192.168.10.99`, MGMT) — управляющий узел: Ansible раскатывает nftables, SSH hardening и конфигурацию сервиса.
