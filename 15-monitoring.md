---
name: monitoring
description: |
  Monitoring — стек наблюдаемости: Prometheus + Grafana с набором экспортеров. Нативный LXC в SERVICES, собирает метрики со всех хостов и сервисов, отдаёт дашборды в Grafana и шлёт алерты в Gotify. Документ описывает установку и systemd, сетевую фильтрацию, публикацию, конфигурацию Prometheus и экспортеров, Grafana, алертинг и бэкап. Используй для вопросов по мониторингу, Prometheus, Grafana, экспортерам, алертингу.
---

# Monitoring

Стек наблюдаемости в отдельном непривилегированном LXC (`192.168.50.21`, SERVICES, CTID `521`). Работает нативно (без Docker): Prometheus собирает метрики, Grafana отдаёт дашборды и рассылает алерты, рядом под systemd крутится набор экспортеров. Prometheus (`0.0.0.0:9090`) и Grafana (`0.0.0.0:3000`) слушают на всех интерфейсах, доступ ограничивается nftables; локальные экспортеры слушают на `127.0.0.1` и скрейпятся Prometheus'ом по loopback. Базлайн LXC, systemd-sandbox, SSH-hardening и nftables-шаблон — см. `02-conventions.md`.

## 1. Файловая структура и systemd

```
/usr/local/bin/prometheus              # Prometheus binary
/etc/prometheus/prometheus.yml         # scrape config
/var/lib/prometheus/                   # TSDB (retention 90d)
/etc/grafana/grafana.ini               # Grafana config
/etc/grafana/provisioning/             # datasources + alerting as code
/var/lib/grafana/grafana.db            # Grafana DB (dashboards, users)
```

Prometheus запущен как `prometheus.service` от системного юзера `prometheus`: `ExecStart` с `--config.file=/etc/prometheus/prometheus.yml`, `--storage.tsdb.path=/var/lib/prometheus`, `--storage.tsdb.retention.time=90d`, `--web.listen-address=0.0.0.0:9090`, `--web.enable-lifecycle` (перечитывание конфига по HUP/`/-/reload`). Sandbox: `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`/`PrivateDevices`, `ReadWritePaths=/var/lib/prometheus`, `Restart=always`.

Grafana — пакет `grafana` версии `13.1.0` под `grafana-server.service`, конфиг `/etc/grafana/grafana.ini`, provisioning в `/etc/grafana/provisioning/`, БД (дашборды, пользователи) — SQLite `/var/lib/grafana/grafana.db`.

Локальные экспортеры под systemd, каждый слушает только loopback и скрейпится Prometheus'ом по `127.0.0.1`: `node_exporter` (`9100`) — метрики самого хоста Monitoring, `proxmox-pve-exporter` (`9221`) — опрос API обоих гипервизоров, `pbs-exporter` (`10019`) — опрос API PBS, `blackbox_exporter` (`9115`) — HTTP-пробы внешних сервисов.

## 2. Сетевая фильтрация

nftables по шаблону сервисного LXC: `policy drop` на входе, loopback и conntrack established/related, базовые ICMP/ICMPv6. Разрешённые входящие: SSH (`22`) из MGMT и VPN; Grafana (`3000`) только с Traefik (`192.168.40.11`) и DockerHost (`192.168.50.30`, виджет Homepage); Prometheus (`9090`) только с Traefik. Локальные экспортеры (`9100`, `9221`, `10019`, `9115`) наружу не открыты — Prometheus берёт их по loopback. Chain `forward` — `drop`, `output` — `accept`.

Prometheus скрейпит цели в других сегментах: `node_exporter` на MGMT-хостах (`192.168.10.11`, `192.168.10.12`, `192.168.10.15`, порт `9100`), `node_exporter` на DockerHost (`192.168.50.30:9100`), метрики Traefik (`192.168.40.11:8081`) и CrowdSec (`192.168.40.11:6060`), а через локальные экспортеры — API гипервизоров и PBS. Эти межсегментные обращения (SERVICES → MGMT и SERVICES → DMZ) открываются явными правилами на OPNsense и на nftables целевых хостов (типовой случай «открытие порта для Monitoring LXC» — см. `02-conventions.md`).

## 3. Публикация и авторизация

Grafana опубликована через Traefik как `grafana.kvasok.xyz` (`root_url = https://grafana.kvasok.xyz/`, `http_addr` пуст → слушает `0.0.0.0:3000`) под цепочкой `chain-admin` плюс middleware `authelia`: сетевое ограничение `allow-mgmt-ips` (MGMT + VPN) и security-headers, поверх которых forward-auth Authelia (TOTP/WebAuthn) требует второй фактор (см. `08-traefik.md`, `12-authelia.md`). Prometheus UI (`9090`) фронтится Traefik под той же связкой `chain-admin` + `authelia` (порт `9090` на nftables открыт только с Traefik).

Аутентификация Grafana — собственная, локальные пользователи (логин/пароль); внешние провайдеры (OIDC/`generic_oauth`, `auth.proxy`) и анонимный доступ выключены. У Prometheus UI своей аутентификации нет, поэтому единственный фактор доступа к нему — Authelia перед Traefik. Итоговая модель защиты админок мониторинга: сетевое ограничение до MGMT + VPN, forward-auth Authelia со вторым фактором и — для Grafana — собственный локальный вход.

## 4. Prometheus и экспортеры

`scrape_interval` и `evaluation_interval` — `15s`, `external_labels: monitor=homelab`. Джобы:

- **prometheus** — сам себя (`127.0.0.1:9090`).
- **node** — `node_exporter` (`9100`) на пяти хостах с метками `instance`: `monitoring` (`127.0.0.1`), `pve-mini` (`192.168.10.11`), `pve` (`192.168.10.12`), `pbs` (`192.168.10.15`), `dockerhost` (`192.168.50.30`).
- **pve** и **pve-mini** — через локальный `proxmox-pve-exporter` (`127.0.0.1:9221`, `metrics_path=/pve`, модули `pve`/`pve-mini`), реальные цели опроса — `192.168.10.12` и `192.168.10.11`.
- **pbs** — локальный `pbs-exporter` (`127.0.0.1:10019`, `scrape_interval=60s`), опрашивает API PBS read-only токеном `monitoring@pbs!pbs-exporter` (роль `Audit`, см. `05-backup.md`).
- **blackbox-http** — локальный `blackbox_exporter` (`127.0.0.1:9115`, модуль `http_2xx`, `60s`): пробы доступности `vaultwarden`, `authelia`, `anchor`, `plumio`, `filebrowser`, `immich`, `linkwarden` (`*.kvasok.xyz`).
- **crowdsec** — `192.168.40.11:6060` (`30s`).
- **traefik** — `192.168.40.11:8081` (`30s`).

TSDB хранится 90 дней (`/var/lib/prometheus`).

## 5. Grafana

Grafana `13.1.0`, единственный data source — Prometheus, заведён как код (`/etc/grafana/provisioning/datasources/prometheus.yml`). Дашборды живут в БД Grafana (`grafana.db`), не в provisioning (там только sample-заглушки), а вот alerting вынесен в код (см. раздел 6). Аутентификация — локальная (раздел 3).

## 6. Алертинг

Grafana unified alerting, вся конфигурация — как код в `/etc/grafana/provisioning/alerting/`: `contactpoints.yaml` (контактные точки), `policies.yaml` (маршрутизация уведомлений), `rules-backups.yaml` и `rules-capacity.yaml` (правила — свежесть бэкапов по метрикам `pbs-exporter` и ёмкость/ресурсы хостов). Уведомления уходят в Gotify: контактная точка шлёт webhook на приложение `grafana-alerts` (`https://gotify.kvasok.xyz/message?token=<токен>`), путь Grafana → Traefik → Gotify (см. `12-gotify.md`). Заголовки алертов — на английском (см. `02-conventions.md`).

## 7. Резервное копирование

Только PBS-снапшот всего LXC в составе общего pve-задания (см. `05-backup.md`). Отдельный restic не заводится: метрики Prometheus (TSDB) эфемерны и восстановимы, а критичное состояние Grafana невелико и покрыто снапшотом — data source и алертинг и так лежат кодом в `/etc/grafana/provisioning/`, дашборды из БД восстанавливаются вместе с LXC.

## 8. Зависимости

- **Traefik (`192.168.40.11`)** — публикует Grafana и Prometheus UI под `chain-admin`; одновременно цель скрейпа (метрики Traefik `8081` и CrowdSec `6060`).
- **Хосты с `node_exporter`** — PVE-Mini, PVE, PBS (MGMT, `9100`) и DockerHost (`192.168.50.30:9100`).
- **API гипервизоров и PBS** — источники для `proxmox-pve-exporter` и `pbs-exporter` (токен `monitoring@pbs!pbs-exporter`, роль `Audit`).
- **Gotify (`192.168.50.22`)** — доставка алертов через приложение `grafana-alerts`.
- **OPNsense** — межсегментные allow-правила для скрейпа (SERVICES → MGMT `9100`, SERVICES → DMZ `8081`/`6060`).
- **PBS (`192.168.10.15`)** — PBS-снапшот системного диска и read-only метрики через `pbs-exporter`.
