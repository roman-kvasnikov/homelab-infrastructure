---
name: ansible
description: |
  Ansible — декларативное управление конфигурацией homelab с управляющего узла mgmt. Репозиторий — источник правды для per-host nftables и SSH-hardening всех гостей, гипервизоров и PBS; роли рендерят канонические конструкции из `02-conventions.md` по данным инвентаря. Документ описывает структуру репозитория, `ansible.cfg` и модель доступа, инвентарь homelab (оси групп, `group_vars`, whitelist'ы по хостам), роли `nftables`, `ssh_hardening` и `ssh_copy_id`, плейбуки baseline и обслуживания, а также отдельный инвентарь нод Remnawave.
---

# Ansible

Конфигурацией homelab управляет **mgmt** — управляющий LXC (CT `199`, `192.168.10.99`, MGMT) на PVE. Репозиторий лежит в `/opt/ansible`, плейбуки запускаются на mgmt от пользователя `ansible`; сам mgmt обслуживается локально (`ansible_connection: local`), остальные хосты — по SSH. Помимо Ansible, mgmt — рабочее место AI-агента (сессии Claude Code под пользователем `claude`) и других инструментов управления инфраструктурой.

Репозиторий — источник правды: `/etc/nftables.conf` и drop-in'ы sshd каждого хоста генерируются шаблонами ролей из инвентаря и на хостах руками не правятся. Канонические конструкции (шаблон nftables, SSH-baseline) описаны в `02-conventions.md`; здесь — как они выражены в коде и где хранится специфика хостов.

## 1. Структура репозитория

```
ansible.cfg
site.yml                        # полный converge: импортирует playbooks/baseline.yml
requirements.yml
bin/
  ssh-key                       # обёртка над playbooks/ops/ssh-copy-id.yml
inventories/
  homelab/                      # инвентарь по умолчанию
    hosts.yml
    group_vars/
      all/
        connection.yml          # ansible_user, admin_user
        network.yml             # admin_workstation_addrs, nft_defines
      vm.yml                    # доступ к VM через admin_user + become
      tcp_forwarding.yml        # Match-исключение AllowTcpForwarding
    host_vars/<host>.yml        # по файлу на хост
  remnanodes/                   # отдельный проект: ноды Remnawave
    hosts.yml
    group_vars/all.yml
playbooks/
  baseline.yml                  # роли ssh_hardening + nftables на группу baseline
  ops/
    apt-upgrade.yml
    reboot.yml
    ssh-copy-id.yml
    nftables-audit.yml
  remnanodes/
    compose-update.yml
roles/
  nftables/                     # /etc/nftables.conf + аудит
  ssh_hardening/                # /etc/ssh/sshd_config.d/10-hardening.conf, 20-match.conf
  ssh_copy_id/                  # authorized_keys
```

Общая логика живёт в ролях, всё, что отличает хосты, — в `group_vars` и `host_vars`. Имена ролей — snake_case (так требует Ansible); имена хостов в инвентаре совпадают с именами гостей Proxmox (`05-proxmox.md`).

## 2. ansible.cfg и модель доступа

`[defaults]` делает `inventories/homelab` инвентарём по умолчанию (нодам Remnawave нужен явный `-i inventories/remnanodes`), задаёт `roles_path = ./roles` и `interpreter_python = /usr/bin/python3`, скрывает skipped-хосты и выводит результаты в YAML (`[callback_default] result_format = yaml`). `host_key_checking = true` вместе с `StrictHostKeyChecking=accept-new` принимает ключ нового хоста при первом подключении, но падает явно при смене ключа известного.

`[ssh_connection]` включает `pipelining = true` и мультиплексирование (`ControlMaster=auto`, `ControlPersist=60s`): одно SSH-соединение переиспользуется для серии задач.

Аутентификация — только по ed25519-ключам, секреты и приватные ключи в репозиторий не коммитятся. Модель доступа:

| Хосты                | Подключение                                                              | Где задано                     |
| :------------------- | :----------------------------------------------------------------------- | :----------------------------- |
| `proxmox`, `lxc`     | `root`                                                                   | `group_vars/all/connection.yml` |
| `vm`                 | `romank` (`admin_user`) + `become`; sshd пускает только его (`AllowUsers`) | `group_vars/vm.yml`            |
| `mgmt`               | `ansible_connection: local`                                              | `hosts.yml`                    |
| `entrypoint` (VPS)   | `super`, порт `12122`, `become`, `ansible_ssh_pipelining: false`          | `host_vars/entrypoint.yml`     |

Pipelining на VPS отключён из-за `requiretty` в sudoers.

## 3. Инвентарь homelab

### Группы

Хосты сгруппированы по двум осям плюс функциональные группы:

| Группа                             | Состав                                   | Назначение                                                  |
| :--------------------------------- | :--------------------------------------- | :---------------------------------------------------------- |
| `vps`, `proxmox`, `lxc`, `vm`      | по типу узла                             | платформа — что это за хост                                 |
| `pve_guests`, `pve_mini_guests`    | по гипервизору                           | размещение — для `--limit` при работах на хосте             |
| `baseline`                         | `proxmox` + `lxc` + `vm`                 | цель ролей `ssh_hardening` и `nftables`                     |
| `tcp_forwarding`                   | `traefik`, `homepage`, `mgmt`, `dev`     | хосты, где админ-ноутбуку разрешены SSH-туннели            |

`vps` в `baseline` не входит сознательно: `entrypoint` — интернет-хост на нестандартном порту со своим firewall, LAN-ruleset на него раскатываться не должен.

Размещение не совпадает с VLAN: `homepage` адресован в INFRA, но живёт на PVE; `amneziawg` из того же INFRA — на PVE-Mini.

### Хосты

| Хост             | Адрес                  | Платформа | Размещение |
| :--------------- | :--------------------- | :-------- | :--------- |
| `entrypoint`     | VPS                    | vps       | —          |
| `pve-mini`       | `192.168.10.11`        | proxmox   | —          |
| `pve`            | `192.168.10.12`        | proxmox   | —          |
| `pbs`            | `192.168.10.15`        | proxmox   | —          |
| `omada`          | `192.168.10.31`        | lxc       | PVE-Mini   |
| `mgmt`           | `192.168.10.99` (local) | lxc      | PVE        |
| `amneziawg`      | `192.168.20.11`        | lxc       | PVE-Mini   |
| `homepage`       | `192.168.20.20`        | lxc       | PVE        |
| `traefik`        | `192.168.40.11`        | lxc       | PVE        |
| `vaultwarden` … `postgres` | `192.168.50.11` – `.90` | lxc | PVE     |
| `xray`           | `192.168.20.12`        | vm        | PVE-Mini   |
| `dev`, `be-free-online`, `hermes` | `192.168.50.40`, `.50`, `.70` | vm | PVE |

Сервисные LXC на PVE — все гости SERVICES из `05-proxmox.md`: vaultwarden, authelia, monitoring, gotify, jellyfin, arr, qbittorrent, organizer, immich, shares, frigate, onlyoffice, open-webui, tdarr, forgejo, yandex-disk, postgres.

Вне инвентаря — appliance'ы, которые не управляются ролями: VM `opnsense` и VM `home-assistant` (HAOS). Их адреса при этом есть в `nft_defines` и используются как источники в чужих правилах.

### group_vars

**`all/connection.yml`** — `ansible_user: root` и `admin_user: romank`; `admin_user` подставляется везде, где нужен человеческий аккаунт (`AllowUsers`, Match-блоки, логин на VM).

**`all/network.yml`** — адресная карта, единственный источник адресов для firewall-правил и SSH Match:

- `admin_workstation_addrs` — адреса админ-ноутбука (`192.168.10.50/32`, `10.8.0.2/32`) для Match-исключений sshd;
- `nft_defines` — словарь имя → адрес, который роль `nftables` рендерит блоком `define` в начало каждого ruleset'а. Имена: `<СЕГМЕНТ>_NET` для подсети, `<СЕГМЕНТ>_<ИМЯ>_IP` для узла, где имя — имя гостя в верхнем регистре без дефисов (`SERVICES_OPENWEBUI_IP`, `SERVICES_HOMEASSISTANT_IP`). Словарь покрывает весь адресный инвентарь (`03-network.md`), включая узлы вне Ansible. Смена адреса правится только здесь.

**`vm.yml`** — подключение под `admin_user` с `become` и `ssh_extra_directives: [AllowUsers romank]`: VM недоступны под root.

**`tcp_forwarding.yml`** — `ssh_match_blocks` с исключением `AllowTcpForwarding yes` для пользователей из `tcp_forwarding_users` (по умолчанию `root` и `romank`) и только с `admin_workstation_addrs`. Глобальный baseline forwarding запрещает (`02-conventions.md`, раздел 4). На mgmt список расширен пользователями `ansible` и `claude`.

### host_vars и whitelist'ы

`host_vars/<host>.yml` хранит специфику узла: чаще всего `nft_service_rules` — порты сервиса и разрешённые источники. SSH у всех хостов открыт стандартному набору из дефолтов роли — админ-ноутбуку (MGMT и VPN) и mgmt; единственное расширение — shares, где SSH доступен ещё Hermes (`nft_ssh_saddr`).

Фактический набор (вывод `playbooks/ops/nftables-audit.yml`), разрешено во `input` сверх SSH:

| Хост             | Порт → источники                                                                                           | `forward` |
| :--------------- | :--------------------------------------------------------------------------------------------------------- | :-------- |
| `pve-mini`       | 8006 ← MGMT, VPN, Homepage, Traefik, Monitoring; 9100 ← Monitoring                                          | `accept`  |
| `pve`            | 8006 ← MGMT, VPN, Homepage, Traefik, Monitoring; 3493 ← PVE-Mini, PBS, Monitoring; 9100 ← Monitoring         | `accept`  |
| `pbs`            | 8007 ← MGMT, VPN, Homepage, Traefik, Monitoring; 9100 ← Monitoring                                          | `drop`    |
| `omada`          | 8043 ← Traefik, Homepage; UDP 29810/19810/27001 и TCP 29811–29817 ← коммутатор                               | `drop`    |
| `mgmt`           | —                                                                                                           | `drop`    |
| `amneziawg`      | UDP 51820 ← any                                                                                             | `drop` + `awg0` in/out |
| `xray`           | 10808, 10809, 12345 ← any                                                                                   | `accept` + nat |
| `homepage`       | 3000 ← Traefik                                                                                              | `drop`    |
| `traefik`        | eth0: 443 ← MGMT, INFRA, TRUSTED, SERVICES, IOT, VPN; 2222 ← MGMT, VPN; 8079 ← Homepage; 8081, 6060 ← Monitoring. wg0: 443 ← VPS | `drop` |
| `vaultwarden`    | 8000 ← Traefik                                                                                              | `drop`    |
| `authelia`       | 9091 ← Traefik                                                                                              | `drop`    |
| `monitoring`     | 3000, 9090, 8080 ← Traefik, Homepage                                                                        | `drop`    |
| `gotify`         | 8060 ← Traefik, Homepage                                                                                    | `drop`    |
| `jellyfin`       | 8096 ← Traefik, Homepage, arr                                                                               | `drop`    |
| `arr`            | 5055, 9696, 7878, 8989 ← Traefik, Homepage                                                                  | `drop`    |
| `qbittorrent`    | 8080 ← Traefik, Homepage, arr                                                                               | `drop`    |
| `organizer`      | 3000, 3100, 5000, 8080, 8081, 8082 ← Traefik                                                                | Docker    |
| `immich`         | 2283 ← Traefik                                                                                              | Docker    |
| `shares`         | 445 ← MGMT, VPN, TRUSTED; 3300 ← Traefik, onlyoffice, Homepage                                               | `drop`    |
| `frigate`        | 5000 ← Traefik                                                                                              | Docker    |
| `onlyoffice`     | 80 ← Traefik, shares                                                                                        | Docker    |
| `open-webui`     | 3000 ← Traefik                                                                                              | Docker    |
| `tdarr`          | 8265 ← Traefik                                                                                              | Docker    |
| `forgejo`        | 3000, 2222 ← Traefik                                                                                        | `drop`    |
| `yandex-disk`    | —                                                                                                           | Docker    |
| `postgres`       | 5432, 6379 ← SERVICES                                                                                       | `drop`    |
| `dev`            | 80 ← any                                                                                                    | `drop`    |
| `be-free-online` | 3000, 3010, 2112 ← Traefik                                                                                  | `drop`    |
| `hermes`         | —                                                                                                           | `drop`    |

«Docker» в колонке `forward` — `nft_docker_host: true`: `policy drop` с тем же whitelist'ом по исходному порту до DNAT (`02-conventions.md`, раздел 2).

Особые случаи:

- **Гипервизоры** держат `forward accept`: через их мосты идёт bridged-трафик гостей, в том числе весь трафик OPNsense на PVE-Mini (`05-proxmox.md`).
- **amneziawg** — `forward drop` с правилами `iifname "awg0" accept` / `oifname "awg0" accept`: маршрутизирует VPN-пиров без маскарадинга (`09-amneziawg.md`).
- **xray** — `forward accept` и дополнительная таблица `ip xray_nat` в `nft_extra_tables`: TCP транзитного трафика к не-приватным адресам перенаправляется на прозрачный порт 12345 (`10-xray.md`). Порты прокси открыты для всех: кто может до них дойти, решает OPNsense (`04-firewall.md`).
- **traefik** — правила привязаны к интерфейсам (`iif`): `eth0` для внутренних VLAN, `wg0` для туннеля с VPS; 2222 — git-SSH-entrypoint для Forgejo.
- **postgres** принимает подключения от всего сегмента SERVICES: у сервисов разные клиенты БД, авторизация — на уровне PostgreSQL.
- **dev** — тестовая машина, порт 80 открыт без ограничения источника внутри сегмента.

## 4. Роль nftables

Роль рендерит `/etc/nftables.conf` из шаблона `nftables.conf.j2`. Структура результата — шаблон из `02-conventions.md` (раздел 2): заголовок `Managed by Ansible` с источником, блок `define` из `nft_defines`, таблица `inet filter` с цепочками `input` (`policy drop`), `forward` и `output` (`policy accept`), затем `nft_extra_tables`.

| Переменная           | По умолчанию                                                     | Назначение                                                            |
| :------------------- | :--------------------------------------------------------------- | :-------------------------------------------------------------------- |
| `nft_defines`        | инвентарь                                                        | `{ИМЯ: значение}`, рендерится в `define`                              |
| `nft_ssh_port`       | `ansible_port \| default(22)`                                    | порт SSH — следует инвентарю, чтобы не отрезать хост на нестандартном порту |
| `nft_ssh_saddr`      | `{ $MGMT_NOTEBOOK_IP, $VPN_MGMT_NOTEBOOK_IP, $MGMT_MGMT_IP }`     | источники SSH                                                         |
| `nft_service_rules`  | `[]`                                                             | правила хоста: `comment`, `port`, опционально `proto`, `iif`, `saddr` |
| `nft_forward_policy` | `drop`                                                           | политика `forward`                                                    |
| `nft_forward_rules`  | `[]`                                                             | сырые строки внутри `forward`                                         |
| `nft_extra_tables`   | `""`                                                             | сырой nft-текст после таблицы filter (например, NAT-таблица Xray)     |
| `nft_docker_host`    | `false`                                                          | Docker-in-LXC: `forward drop` с зеркалом `nft_service_rules` по исходному порту до DNAT, `nft_forward_policy` игнорируется, Docker перезапускается после reload |

Правило в `nft_service_rules`: `port` — одиночный или набор `{ ... }`, `proto` — `tcp` по умолчанию, `saddr` без значения — любой источник, `iif` привязывает правило к интерфейсу. Каждое правило рендерится строкой `accept` с комментарием над ней; на Docker-хостах — ещё и строкой в `forward`.

Задача деплоит шаблон с `validate: nft -c -f %s` — синтаксически битый ruleset не записывается — и `backup: true`. Изменение уведомляет два handler'а: `Reload nftables` (reload службы) и `Restart docker` (только при `nft_docker_host`: `flush ruleset` удаляет NAT-цепочки Docker, а Docker пересоздаёт их лишь при старте демона). Отдельная задача держит `nftables` в `enabled` и `started`.

`tasks/audit.yml` — режим аудита: без подключения к хостам вычисляет из inventory, что каждый хост разрешает во `input`, и печатает карточку на хост.

## 5. Роль ssh_hardening

Роль управляет двумя drop-in'ами в `/etc/ssh/sshd_config.d/`, не трогая основной `sshd_config`, чтобы изменения переживали обновление openssh-server.

- **`10-hardening.conf`** — baseline из дефолтов роли: `PermitRootLogin prohibit-password`, `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `X11Forwarding no`, `AllowAgentForwarding no`, `AllowTcpForwarding no`, `ClientAliveInterval 300`, `ClientAliveCountMax 2`, плюс `ssh_extra_directives`.
- **`20-match.conf`** — блоки `Match` из `ssh_match_blocks` (`comment`, `match`, `directives`); каждый ослабляет директиву для конкретных пользователей и адресов. Файл создаётся только при непустом списке, иначе удаляется.

Оба файла деплоятся с `backup: true`; после деплоя `sshd -t` проверяет полную эффективную конфигурацию, handler делает reload sshd. `ssh_extra_directives` использует группа `vm` (`AllowUsers`), `ssh_match_blocks` — группа `tcp_forwarding`. Обоснование baseline — `02-conventions.md`, раздел 4.

## 6. Роль ssh_copy_id

Добавляет и удаляет публичные ключи в `authorized_keys`. Ключи берутся из `roles/ssh_copy_id/files/` по списку `ssh_copy_id_keys` (у каждого — `file` и опциональный `state: absent`). Повседневная обёртка — `bin/ssh-key` (например, `bin/ssh-key add romank xray`).

## 7. Плейбуки

| Плейбук                             | Цель                                    | Что делает                                                                  |
| :---------------------------------- | :-------------------------------------- | :-------------------------------------------------------------------------- |
| `site.yml`                          | `baseline`                              | полный converge; импортирует `playbooks/baseline.yml`                       |
| `playbooks/baseline.yml`            | `baseline`                              | роли `ssh_hardening` и `nftables`; теги `baseline`, `ssh_hardening`, `nftables` |
| `playbooks/ops/apt-upgrade.yml`     | `-e target=`, по умолчанию `baseline`   | `apt dist-upgrade` по одному хосту (`serial: 1`), отчёт о необходимости перезагрузки; сам не перезагружает |
| `playbooks/ops/reboot.yml`          | только явный `-e target=`               | поочерёдная перезагрузка с ожиданием возврата хоста                         |
| `playbooks/ops/ssh-copy-id.yml`     | только явный `-e target=`               | роль `ssh_copy_id`                                                          |
| `playbooks/ops/nftables-audit.yml`  | `baseline`, `connection: local`         | карточки `input`-правил всех хостов без SSH в гости                         |
| `playbooks/remnanodes/compose-update.yml` | `remnanodes`                      | pull образов и пересоздание стека remnanode по одной ноде                   |

Плейбуки с опасным действием (`reboot`, `ssh-copy-id`) без `-e target=<group>` не матчат ни одного хоста — случайный прогон на весь инвентарь исключён. Ops-плейбуки работают и с инвентарём remnanodes (`-i inventories/remnanodes -e target=remnanodes`).

### Порядок раскатки

Изменения ролей и host_vars сначала проверяются без применения — `ansible-playbook site.yml --check --diff --limit <host> --tags nftables` — и раскатываются по одному хосту. При изменениях SSH или firewall держится открытая вторая SSH-сессия, а на крайний случай — консоль через `pct enter` / `qm terminal` на гипервизоре.

## 8. Инвентарь remnanodes

Отдельный инвентарь для нод VPN-сервиса Remnawave (`be-free-online`), независимый от homelab и никогда не пересекающийся с ним: запуск только с явным `-i inventories/remnanodes`. Группа `remnanodes` — по одной ноде на локацию (`de`, `ee`, `fi`, `fr`, `nl`, `nld`, `pl`, `uk`). Роли baseline на ноды не применяются; с ними работают `compose-update.yml` и ops-плейбуки.

## 9. Зависимости

- **mgmt в MGMT** — стандартный `nft_ssh_saddr` всех хостов разрешает SSH с `192.168.10.99`, поэтому Ansible достаёт любой узел; OPNsense даёт mgmt доступ во все приватные сети (`04-firewall.md`).
- **`group_vars/all/network.yml`** — единственный источник адресов для рендера правил.
- **`02-conventions.md`** — канонические шаблоны nftables и SSH, которые воплощают роли.
- **`04-firewall.md`** — двухслойная модель фильтрации; роль `nftables` разворачивает её внутрисегментный слой.
