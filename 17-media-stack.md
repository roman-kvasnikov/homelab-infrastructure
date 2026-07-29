---
name: media-stack
description: |
  Медиа-стек на нативных LXC: Jellyfin, arr-стек (Prowlarr, Sonarr, Radarr) и qBittorrent в отдельных непривилегированных контейнерах SERVICES. Общие медиаданные на ZFS-датасете хоста через bind-mount, доступ по общей группе GID 981, hardlink между torrents и media. Документ описывает раскладку контейнеров, общую группу и idmap, bind-mounts, сервис-юзеры, исходящий прокси, установку сервисов. Используй для вопросов по медиасерверу, Jellyfin, arr-стеку, Sonarr, Radarr, Prowlarr, qBittorrent, hardlink, скачиванию и раздаче.
---

# Медиа-стек

Медиа-стек работает нативно (без Docker) в трёх непривилегированных LXC в SERVICES.
Jellyfin отдаёт медиа, контейнер arr держит Prowlarr, Sonarr и Radarr, qBittorrent изолирован в собственном контейнере.
Каждый сервис — системный юзер с systemd-юнитом, по нативному паттерну Vaultwarden и Authelia.
Базлайн LXC, systemd-sandbox, SSH-hardening и шаблон nftables — см. `02-conventions.md`.

## 1. Контейнеры

| CTID | Hostname | IP | Сервисы |
|------|----------|-----|---------|
| 531 | jellyfin | 192.168.50.31 | Jellyfin |
| 532 | arr | 192.168.50.32 | Prowlarr, Sonarr, Radarr |
| 533 | qbittorrent | 192.168.50.33 | qBittorrent |

Все три — unprivileged Debian на VLAN 50 (SERVICES), rootfs на `zguests`, шлюз и DNS — OPNsense в VLAN 50 (`192.168.50.1`).
Изоляция qBittorrent в отдельном контейнере выбрана сознательно: торрент-клиент тянет недоверенный контент и общается с произвольными пирами — наиболее вероятная точка компрометации стека.

## 2. Общая группа и данные

Медиаданные лежат на выделенном ZFS-датасете хоста и bind-mount'ятся в контейнеры.
Sonarr, Radarr и qBittorrent делят эти данные через общую группу с GID `981` — тот же PGID, что использовался в Docker.

Раскладка на хосте — в стиле TRaSH: загрузки и медиатека под общим родителем, чтобы работал hardlink.
Родитель `data` содержит `torrents` и `media`, в каждом — `movies` и `tv`.

Владелец дерева — `root:981`, режим `2775`.
Setgid-бит гарантирует, что новые файлы и папки наследуют группу 981, а group-write разрешает всем медиасервисам создавать файлы и линковать их.

## 3. Маппинг GID

Каждый контейнер маппит GID 981 напрямую хост↔контейнер, чтобы общая группа резолвилась в одну и ту же identity с обеих сторон.
Без маппинга дефолтный offset unprivileged-контейнера превратил бы GID 981 в хостовый 101981 и сломал общий доступ.

Маппинг — четыре строки `lxc.idmap` в конфиге каждого контейнера.

```
lxc.idmap: u 0 100000 65536
lxc.idmap: g 0 100000 981
lxc.idmap: g 981 981 1
lxc.idmap: g 982 100982 64554
```

Проброс одного GID требует одной парной строки на хосте в `/etc/subgid`.

```
root:981:1
```

Строка в `/etc/subgid` глобальная, добавляется один раз для всех контейнеров, а не на каждый.

Внутри каждого контейнера создаётся группа `media` с GID 981, в неё включаются сервис-юзеры, которым нужен доступ к данным.

```
groupadd -g 981 media
```

## 4. Bind-mounts

Jellyfin монтирует только медиатеку, на запись — он пишет обложки и trickplay рядом с медиа.

```
pct set 531 -mp0 <host_media_path>/media,mp=/data/media
```

Контейнер arr монтирует всё дерево `data` на запись — Sonarr и Radarr создают hardlink между `torrents` и `media`.

```
pct set 532 -mp0 <host_media_path>,mp=/data
```

qBittorrent монтирует только `torrents`, по принципу минимума привилегий — download-клиент не должен видеть медиатеку.
Hardlink при этом работает, потому что его создают arr-сервисы, видящие оба каталога.

```
pct set 533 -mp0 <host_media_path>/torrents,mp=/data/torrents
```

Внутренние пути короткие и стабильные, чтобы при миграции менялась только хостовая сторона bind-mount, а конфиги сервисов оставались нетронутыми.

## 5. Сервис-юзеры

Prowlarr не работает с медиаданными и запускается под собственной приватной группой.

```
adduser --system --group --no-create-home --disabled-password prowlarr
```

Sonarr, Radarr и qBittorrent работают с данными и используют `media` как основную группу.

```
adduser --system --no-create-home --disabled-password --ingroup media sonarr
adduser --system --no-create-home --disabled-password --ingroup media radarr
adduser --system --no-create-home --disabled-password --ingroup media qbittorrent
```

## 6. Исходящий прокси

Все три контейнера гонят исходящий трафик через Xray HTTP-прокси (`192.168.20.12:10809`) — Cloudflare блокирует прямые запросы к release-эндпоинтам Servarr и GitHub.
Прокси задаётся глобально через systemd `DefaultEnvironment`, поэтому его наследуют все сервисы и все команды `systemd-run`.

```
DefaultEnvironment="HTTP_PROXY=http://192.168.20.12:10809" "HTTPS_PROXY=http://192.168.20.12:10809" "NO_PROXY=localhost,127.0.0.1,192.168.0.0/16,172.16.0.0/12,10.0.0.0/8"
```

Строка дописывается в `/etc/systemd/system.conf` и применяется через `systemctl daemon-reexec`.
`NO_PROXY` покрывает все локальные сети, чтобы межсервисный и inter-VLAN трафик шёл мимо прокси.
Доступ SERVICES → INFRA на порт 10809 открывается правилом на OPNsense (см. `04-firewall.md`).

Скачивания, которым нужно пройти Cloudflare, запускаются в прокси-окружении.

```
systemd-run --wait --pipe curl -L -o /tmp/file "<url>"
```

## 7. Locale

В каждом контейнере генерируется locale `en_US.UTF-8`, чтобы сервисы корректно обрабатывали не-ASCII имена файлов.

```
apt install -y locales
sed -i 's/# en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen
locale-gen
update-locale LANG=en_US.UTF-8
```

## 8. Раскладка приложений

Бинарь каждого приложения лежит в `/opt`, изменяемое состояние — в `/var/lib`, code и state разделены.
Servarr самообновляются, записывая в свой каталог `/opt`, поэтому директория приложения принадлежит сервис-юзеру.
Каталоги состояния — режим `750`: базы сервисов приватные и не шарятся.

## 9. Установка Servarr

Sonarr, Radarr и Prowlarr — self-contained .NET-приложения из официальных release-эндпоинтов Servarr.
Каждое скачивается через прокси, распаковывается в `/opt`, отдаётся сервис-юзеру, указывается на каталог данных в `/var/lib`.

Prowlarr и Radarr — эндпоинт `servarr.com`.

```
curl -L -o /tmp/prowlarr.tar.gz "https://prowlarr.servarr.com/v1/update/master/updatefile?os=linux&runtime=netcore&arch=x64"
curl -L -o /tmp/radarr.tar.gz "https://radarr.servarr.com/v1/update/master/updatefile?os=linux&runtime=netcore&arch=x64"
```

Sonarr — эндпоинт `services.sonarr.tv`.

```
curl -L -o /tmp/sonarr.tar.gz "https://services.sonarr.tv/v1/download/main/latest?version=4&os=linux&arch=x64"
```

Systemd-юнит Servarr запускает бинарь с явным каталогом данных.
Sonarr и Radarr задают `Group=media` и `UMask=0002`, чтобы импортированные файлы попадали в группу 981 с group-write.
Prowlarr работает под своей группой — доступа к данным у него нет.

```
[Unit]
Description=Sonarr
After=network.target

[Service]
Type=simple
User=sonarr
Group=media
UMask=0002
ExecStart=/opt/Sonarr/Sonarr -nobrowser -data=/var/lib/sonarr
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
ProtectSystem=full
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Порты сервисов: Prowlarr 9696, Sonarr 8989, Radarr 7878.

## 10. Установка qBittorrent

qBittorrent — статический бинарь `qbittorrent-nox` из проекта userdocs, ветка libtorrent v1.2 (совпадает с текущей конфигурацией).
Бинарь кладётся в `/opt/qbittorrent`, делается исполняемым, отдаётся `qbittorrent:media`.

```
curl -L -o /opt/qbittorrent/qbittorrent-nox "https://github.com/userdocs/qbittorrent-nox-static/releases/download/release-5.2.3_v1.2.20/x86_64-qbittorrent-nox"
chmod +x /opt/qbittorrent/qbittorrent-nox
```

Systemd-юнит запускает headless с явным профилем и WebUI-портом.

```
[Unit]
Description=qBittorrent-nox
After=network.target

[Service]
Type=simple
User=qbittorrent
Group=media
UMask=0002
ExecStart=/opt/qbittorrent/qbittorrent-nox --profile=/var/lib/qbittorrent --webui-port=8080
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
ProtectSystem=full
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

При первом старте qBittorrent пишет временный пароль WebUI в лог, пока не задан постоянный.

## 11. Hardlink

Hardlink, создаваемый медиа-юзером между `torrents` и `media`, делит один inode на оба пути — вся цепочка общей группы работает end-to-end.
Счётчик ссылок равен двум, оба имени резолвятся в один inode: импорт не занимает лишнего места, оригинал остаётся доступен для раздачи.

## 12. Адресация между сервисами

Сервисы обращаются друг к другу по адресу, а не по Docker-именам.
Prowlarr, Sonarr и Radarr общаются через `localhost` — они в одном контейнере arr.
arr-сервисы обращаются к qBittorrent по `192.168.50.33:8080` — он в отдельном контейнере.

## 13. Публикация и фильтрация

Целевое состояние по базлайну `02-conventions.md`: каждый сервис слушает только на своём адресе, nftables `policy drop`, входящие только от Traefik, SSH из MGMT и VPN.
Jellyfin публикуется под `chain-external`, arr-сервисы и qBittorrent WebUI — под админской цепочкой с Authelia (детали цепочек — `08-traefik.md`).

## Pending

- nftables-шаблон на всех трёх контейнерах (сейчас сервисы слушают на `0.0.0.0`, фильтрация не настроена).
- Явный bind сервисов на свой адрес вместо `0.0.0.0`.
- Публикация через Traefik и подключение Authelia для админских WebUI.
- Проброс iGPU в Jellyfin для аппаратного транскода (шаринг одной iGPU между несколькими LXC).
- Kill-switch / VPN для qBittorrent, чтобы торрент-трафик не утекал мимо туннеля.
- Перенос боевых данных с DockerHost: раскладка config по нативной структуре, сохранение идентичных внутренних путей ради БД сервисов, переназначение внутрисетевых адресов с Docker-имён на IP/localhost.
- Замена тестового источника bind-mount на боевой медиа-датасет.
