---
name: gotify
description: |
  Gotify — сервер push-уведомлений. LXC в SERVICES, нативный Go-бинарник, SQLite, доступ только через Traefik под собственной авторизацией (без Authelia). Источник алертов мониторинга. Документ описывает установку, конфигурацию, фильтрацию, публикацию, приложение grafana-alerts и бэкап. Используй для вопросов по Gotify, push-уведомлениям, алертингу.
---

# Gotify

Сервер push-уведомлений в отдельном непривилегированном LXC (`192.168.50.22`, SERVICES). Работает нативно (без Docker) на Go-бинарнике Gotify v2.9.1 под systemd, БД — SQLite. Источник уведомлений для алертов мониторинга и прочих сервисов: каждое приложение-источник имеет свой токен, клиенты (телефон, браузер) принимают сообщения. Базлайн LXC, systemd-sandbox, SSH-hardening и nftables-шаблон — см. `02-conventions.md`.

## 1. Файловая структура и systemd

```
/usr/local/bin/gotify                 # binary from github.com/gotify/server/releases
/etc/gotify/config.yml                 # configuration
/var/lib/gotify/data/gotify.db         # main DB (SQLite): users, apps, messages
```

Сервис `gotify.service` от системного юзера `gotify`, sandbox-набор из `02-conventions.md`, запись только в `/var/lib/gotify`, `Restart=always`, автозапуск. Слушает `192.168.50.22:8060` (конкретный LAN-IP, не `0.0.0.0`).

## 2. Сетевая фильтрация

nftables по шаблону сервисного LXC (`02-conventions.md`): `policy drop`, порт `8060` только с Traefik (`192.168.40.11`), SSH из MGMT и VPN. Прямой доступ к 8060 мимо Traefik закрыт.

## 3. Публикация и авторизация

Gotify опубликован через Traefik как `gotify.kvasok.xyz` под цепочкой `chain-external` (CrowdSec + rate-limit + security-headers), **без** middleware `authelia`. Это сознательное исключение из общего правила (обычно сервисы прикрыты Authelia): forward-auth Authelia несовместим с родной моделью авторизации Gotify. Gotify использует собственную авторизацию (логин/пароль для UI и приложений, токен приложения для отправки сообщений) и Basic Auth для ряда эндпоинтов; при включённом forward-auth запросы к `/manifest.json` и API заворачиваются на страницу логина Authelia до срабатывания родной авторизации Gotify, и клиент не может аутентифицироваться. Поэтому Gotify выведен из-под Authelia и защищён только собственной авторизацией. Детали цепочек — см. `08-traefik.md`.

## 4. Приложение grafana-alerts

Для алертинга Grafana в Gotify заведено отдельное приложение `grafana-alerts` со своим токеном. Grafana шлёт уведомления о срабатывании правил через webhook на Gotify API (`https://gotify.kvasok.xyz/message?token=<токен>`), путь Grafana → Traefik → Gotify. Токен хранится в provisioning-конфиге контакта на стороне Grafana; в Gotify это обычное приложение-источник. Детали алертинга — см. `15-monitoring.md`.

## 5. Резервное копирование

Только PBS-снапшот всего LXC в составе общего pve-задания. Отдельный restic не заводится: критичного point-in-time состояния, ради которого нужен гранулярный restic (как БД паролей у Vaultwarden или TOTP/OIDC у Authelia), у Gotify нет — SQLite с подписками и историей сообщений некритична, а file-level restore одного файла из CT-снапшота PBS при необходимости доступен. См. `06-backup.md`.

## 6. Зависимости

- **Traefik (`192.168.40.11`)** — единственный разрешённый источник запросов к 8060 (nftables). Без Traefik сервис недоступен снаружи LXC.
- **Unbound на OPNsense** — DNS, split-horizon `gotify.kvasok.xyz → 192.168.40.11`.
- **Monitoring / Grafana** — источник алертов для приложения `grafana-alerts`.
- **PBS (`192.168.10.15`)** — PBS-снапшоты.
