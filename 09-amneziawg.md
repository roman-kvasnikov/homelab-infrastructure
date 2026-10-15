---
name: amneziawg
description: |
  AmneziaWG 3.1 — обфусцированный VPN-сервер для удалённого доступа в домашнюю сеть. LXC в INFRA, вход по белому IP через порт-форвард на OPNsense, минуя VPS. Документ описывает установку и обновление из исходников, конфигурацию сервера, параметры обфускации AWG 3.x, клиентов и split-DNS, маршрутизацию VPN-подсети, политику доступа по пирам, nftables, автозапуск и связку с OPNsense. Используй для вопросов по удалённому доступу к дому, VPN, AmneziaWG.
---

# AmneziaWG

AmneziaWG обеспечивает удалённый доступ в домашнюю сеть извне. Протокол — WireGuard с обфускацией уровня AmneziaWG 3.1: случайные заголовки пакетов, паддинг всех типов сообщений, шифрование заголовков и пакеты-обманки перед рукопожатием. Криптография WireGuard не изменена. Подключившись, клиент получает доступ к внутренним сегментам в объёме, определённом правилами OPNsense для его адреса в туннеле.

AmneziaWG развёрнут непривилегированным LXC `amneziawg` (CT 103) в сегменте INFRA, `192.168.20.11`. Слушает UDP `51820`. Вход снаружи — по белому IP дома через порт-форвард на OPNsense, напрямую, без участия VPS. VPN-подсеть клиентов — `10.8.0.0/24`, адрес сервера в туннеле — `10.8.0.1`.

Реализация — userspace (`amneziawg-go`): загрузить kernel-модуль AmneziaWG из LXC нельзя, поэтому интерфейс поднимается через userspace-бэкенд. Это штатно для контейнера и не требует модулей на хосте.

Версии компонентов:

| Компонент | Версия | Путь | Источник |
| --- | --- | --- | --- |
| `amneziawg-go` | `v3.1.20260828` | `/usr/local/bin/amneziawg-go` | статическая сборка из исходников на Dev VM |
| `amneziawg-tools` (`awg`, `awg-quick`) | `v3.1.20260812` | `/usr/bin/awg`, `/usr/bin/awg-quick` | сборка из исходников в LXC |

Оба компонента установлены вне пакетного менеджера: в репозиториях Debian 12 AmneziaWG нет. Обновление описано в разделе 3.

## 1. Поток трафика

```
[Client outside home] --AWG / UDP 51820--> [home public IP]
      --> [OPNsense: port forward 51820 -> 192.168.20.11]
            --> [AmneziaWG LXC awg0 10.8.0.1]
                  --> [OPNsense: route 10.8.0.0/24 via AMNEZIA_GW]
                        --> [internal segments]
```

Клиент подключается на белый IP дома, порт 51820. OPNsense пробрасывает этот UDP-порт на AmneziaWG-LXC в INFRA. LXC принимает туннель и форвардит трафик клиента в сеть с сохранением оригинального адреса `10.8.0.<N>`. Обратный путь обеспечен статическим маршрутом на OPNsense: `10.8.0.0/24` через шлюз `192.168.20.11`.

## 2. Конфигурация сервера

Конфиг — `/etc/amnezia/amneziawg/awg0.conf`, права `600`.

```
[Interface]
PrivateKey = <server private key>
Address = 10.8.0.1/24
ListenPort = 51820
# Obfuscation - must match all clients
S1 = <...>
S2 = <...>
S3 = <...>
S4 = <...>
H1 = <min-max>
H2 = <min-max>
H3 = <min-max>
H4 = <min-max>
HeaderProtectionKey = <header protection key>

[Peer]
PublicKey = <client public key>
PresharedKey = <preshared key>
AllowedIPs = 10.8.0.<N>/32
```

**Interface.** Адрес и порт туннеля, приватный ключ сервера и серверные параметры обфускации. Клиентские параметры (`Jc/Jmin/Jmax`, `I1–I5`) на сервере не задаются: их генерирует инициатор соединения, сервер такие пакеты игнорирует.

**Peers.** Каждый клиент — отдельная секция `[Peer]` с публичным ключом клиента, симметричным preshared-ключом и `AllowedIPs = 10.8.0.<N>/32` (уникальный адрес клиента в VPN-подсети).

Фактические значения параметров обфускации в документацию не выносятся: репозиторий публичный, а уникальный набор параметров — часть защиты от DPI. Источник истины — `awg0.conf` на сервере и шаблон `/root/awg-v3-params` (раздел 4.3).

### 2.1. Параметры обфускации

| Параметр | Сторона | Назначение | Ограничения |
| --- | --- | --- | --- |
| `S1` | обе, совпадают | паддинг инициации рукопожатия | `S1 + 56 ≠ S2`, иначе инициация и ответ одного размера |
| `S2` | обе, совпадают | паддинг ответа на рукопожатие | см. `S1` |
| `S3` | обе, совпадают | паддинг cookie-пакетов | не больше 64 для клиентов на kernel-модуле |
| `S4` | обе, совпадают | паддинг каждого пакета данных | не больше 32 для клиентов на kernel-модуле; увеличивает размер всех пакетов |
| `H1–H4` | обе, совпадают | диапазоны значений заголовка для четырёх типов сообщений | диапазоны `min-max` не пересекаются; значение выбирается случайно для каждого пакета |
| `HeaderProtectionKey` | обе, совпадают | шифрование заголовков пакетов (ChaCha20) | при заданном ключе все `S1–S4` не меньше 12; генерируется `awg genkey` |
| `Jc/Jmin/Jmax` | клиент | мусорные пакеты перед рукопожатием | `Jmax` меньше MTU системы, иначе фрагментация |
| `I1–I5` | клиент | пакеты-обманки, имитирующие другой протокол | итоговый пакет меньше MTU |
| `ContentPaddingAddition` | клиент, желательно обе | дополнительный паддинг внутри зашифрованных данных | не задан |
| `RekeyAfterTime` и другие тайминги | клиент | рандомизация таймингов WireGuard | не заданы |

`HeaderProtectionKey` — секрет того же уровня, что и ключи WireGuard. `awg show` выводит его открытым текстом, поэтому вывод интерфейса для логов и обсуждений фильтруется: `awg show awg0 | grep -v key`.

`ContentPaddingAddition` и тайминги — резерв усиления обфускации, если на какой-то сети туннель начнут глушить посреди сессии.

## 3. Установка и обновление

Обновление выполняется только из домашней сети: ошибка на этом LXC обрывает удалённый доступ, а восстановить его извне нечем. Софт и параметры обфускации обновляются раздельными этапами: сначала бинарники при неизменном конфиге (проверка совместимости со всеми клиентами), затем конфиг.

**Снимок.** На хосте Proxmox перед любыми изменениями:

```
pct snapshot 103 pre-<change> --description "before <change>"
```

**amneziawg-go.** Собирается на Dev VM, в LXC компилятор не ставится. Требуемая версия Go указана в `go.mod` нужного тега. Go берётся архивом во временный каталог, не пакетом. Сборка обязательно статическая (`CGO_ENABLED=0`), иначе бинарник зависит от версии glibc сборочной машины.

```
git clone --depth 1 --branch <tag> https://github.com/amnezia-vpn/amneziawg-go /tmp/amneziawg-go
cd /tmp/amneziawg-go
CGO_ENABLED=0 make
ldd amneziawg-go                  # expected: not a dynamic executable
scp amneziawg-go root@192.168.20.11:/usr/local/bin/amneziawg-go.new
```

**amneziawg-tools.** Собираются в LXC (`make` и `gcc` там есть) и сначала устанавливаются во временный каталог, чтобы сравнить с текущими:

```
git clone --depth 1 --branch <tag> https://github.com/amnezia-vpn/amneziawg-tools /root/amneziawg-tools
cd /root/amneziawg-tools/src
make
make install DESTDIR=/root/awg-stage WITH_WGQUICK=yes WITH_SYSTEMDUNITS=yes
diff /usr/bin/awg-quick /root/awg-stage/usr/bin/awg-quick
diff /lib/systemd/system/awg-quick@.service /root/awg-stage/usr/lib/systemd/system/awg-quick@.service
```

**Замена.** Старые бинарники сохраняются рядом. Работающий `amneziawg-go` заменяется через `mv`: перезапись запущенного файла завершается ошибкой `Text file busy`.

```
cp -a /usr/local/bin/amneziawg-go /usr/local/bin/amneziawg-go.old
cp -a /usr/bin/awg /usr/bin/awg.old
mv /usr/local/bin/amneziawg-go.new /usr/local/bin/amneziawg-go
install -m 755 /root/awg-stage/usr/bin/awg /usr/bin/awg
systemctl restart awg-quick@awg0
ls -l /proc/$(pgrep -f 'amneziawg-go awg0')/exe    # must not end with (deleted)
```

После замены подключение проверяется с клиента через мобильный интернет, не из дома: подключение к собственному белому IP из LAN зависит от NAT reflection на OPNsense и не проверяет AWG.

**Откат** — возврат `.old`-бинарников и перезапуск юнита, при серьёзной поломке — откат LXC на снимок.

**Проверка конфига без риска.** Новый конфиг проверяется на временном интерфейсе рядом с рабочим. `ListenPort` убирается, чтобы не конфликтовать с `awg0`:

```
amneziawg-go awgtest
awg-quick strip /root/awgtest.conf | sed '/^ListenPort/d' | awg setconf awgtest /dev/stdin
awg show awgtest | grep -v key
ip link del awgtest
```

При ручном запуске `amneziawg-go` может вывести предупреждение о поддержке AmneziaWG ядром — на работу в LXC это не влияет.

## 4. Клиенты

Приложение на устройстве — именно **AmneziaWG** с поддержкой 3.x (Linux: `amneziawg-tools` 3.x и kernel-модуль; Android: AmneziaWG для Android 3.x). Обычный WireGuard-клиент не понимает параметры обфускации.

Все клиенты работают в split-режиме: через туннель идут только домашние сети, интернет — через локальный канал клиента. Неработающий туннель не должен лишать клиента интернета.

### 4.1. Ноутбук

Конфиг — `/etc/amnezia/amneziawg/awg-home.conf`, поднимается `awg-quick up awg-home`.

```
[Interface]
PrivateKey = <client private key>
Address = 10.8.0.2/32
PostUp = resolvectl dns %i 192.168.20.1
PostUp = resolvectl domain %i '~kvasok.xyz' '~admin.be-free.online'
PostUp = resolvectl default-route %i false
# Client-side obfuscation
Jc = 4
Jmin = 40
Jmax = 70
# Must match server
S1 = <...>
S2 = <...>
S3 = <...>
S4 = <...>
H1 = <min-max>
H2 = <min-max>
H3 = <min-max>
H4 = <min-max>
HeaderProtectionKey = <header protection key>
# Decoy packet before handshake (client-side)
I1 = <b 0xc000000001><r 64><t>

[Peer]
PublicKey = <server public key>
PresharedKey = <preshared key>
Endpoint = <home public IP>:51820
AllowedIPs = 192.168.0.0/16, 10.8.0.0/24
PersistentKeepalive = 25
```

**Split-DNS.** Строки `DNS =` в конфиге нет намеренно: `awg-quick` регистрирует такой DNS в systemd-resolved как эксклюзивный, и при поднятом, но неработающем туннеле перестаёт резолвиться всё, включая интернет. Вместо этого `PostUp` направляет в туннель только запросы по `kvasok.xyz` и `admin.be-free.online` (Unbound на OPNsense, split-horizon), остальное резолвится DNS локальной сети. Маршрутизация в systemd-resolved идёт по суффиксу: `~kvasok.xyz` включает и сам домен, и все поддомены. `PostDown` не нужен: настройки исчезают вместе с интерфейсом.

**MTU.** Используется стандартный для `awg-quick` `1420`. С текущим `S4` полноразмерные пакеты проходят без фрагментации, проверено загрузкой страницы через туннель с мобильного интернета. При зависании больших передач на какой-то сети в `[Interface]` задаётся `MTU = 1280`.

### 4.2. Телефон

Конфиг хранится на сервере — `/etc/amnezia/amneziawg/client2.conf`, адрес `10.8.0.3`. Отличия от ноутбука: `DNS = 192.168.10.1` (на Android split-DNS средствами конфига недоступен), нет `PostUp` и `I1`. Передаётся на телефон QR-кодом:

```
qrencode -t ansiutf8 < /etc/amnezia/amneziawg/client2.conf
```

QR содержит приватный ключ и ключ защиты заголовков: снимки экрана с ним не сохраняются и никуда не пересылаются.

### 4.3. Добавление клиента

Серверные параметры обфускации хранятся в шаблоне `/root/awg-v3-params` (права `600`): `S1–S4`, `H1–H4`, `HeaderProtectionKey`. Клиентский конфиг собирается из него, а не копированием с экрана.

Генерация ключей на сервере: `awg genkey | tee client.key | awg pubkey > client.pub`, PSK — `awg genpsk`. Новый `[Peer]` добавляется в `awg0.conf` с уникальным `AllowedIPs = 10.8.0.<N>/32` и применяется на живом интерфейсе без разрыва существующих клиентов: `awg syncconf awg0 <(awg-quick strip awg0)`. Клиентский конфиг: ключи, адрес, клиентские параметры обфускации и содержимое `/root/awg-v3-params`. Политика доступа для нового пира задаётся правилами интерфейса INFRA на OPNsense — nftables сервисов править не требуется, они оперируют VPN-подсетью целиком.

### 4.4. Смена параметров обфускации

Параметры, которые должны совпадать, меняются одновременно на сервере и у всех клиентов: после перезапуска `awg0` клиенты со старыми параметрами не подключаются, пока не получат новые. Порядок: новый `/root/awg-v3-params`, проверка конфига на временном интерфейсе (раздел 3), замена `awg0.conf` с сохранением копии, перезапуск, затем клиенты. На клиентах строки `S1–S4`, `H1–H4`, `HeaderProtectionKey` удаляются и вставляются из шаблона, без вывода ключа на экран:

```
ssh root@192.168.20.11 cat /root/awg-v3-params > /tmp/awg-v3-params
sudo sed -i -E '/^(S1|S2|S3|S4|H1|H2|H3|H4|HeaderProtectionKey) =/d' /etc/amnezia/amneziawg/awg-home.conf
sudo sed -i '/^Jmax/r /tmp/awg-v3-params' /etc/amnezia/amneziawg/awg-home.conf
rm /tmp/awg-v3-params
```

## 5. Маршрутизация и модель адресации

VPN-подсеть **маршрутизируется**, а не маскарадится: выходя из туннеля, пакеты сохраняют оригинальный source-адрес `10.8.0.<N>`. Обратный путь обеспечен статическим маршрутом на OPNsense (`10.8.0.0/24` через `AMNEZIA_GW`), поэтому подмена адреса не нужна.

Это даёт различимость пиров на уровне firewall. Адрес в туннеле — криптографически подтверждённая идентичность: `AllowedIPs` в секции пира означает, что пакет с этим source-адресом принимается только от владельца соответствующего приватного ключа, подделка невозможна. Внешний IP клиента при этом меняется произвольно и в правилах не участвует.

Текущие пиры:

| Адрес | Устройство | Доступ |
| --- | --- | --- |
| `10.8.0.2` | Ноутбук | Полный — все сегменты, включая This Firewall (удалённое администрирование в объёме MGMT) |
| `10.8.0.3` | Телефон | Только Traefik `192.168.40.11:443` |

Разделение осмысленно: ноутбук — рабочий инструмент администрирования, телефон нужен лишь для доступа к опубликованным сервисам. Компрометация телефона (устройство на чужих сетях, больше стороннего софта) не даёт доступа к инфраструктуре.

**Следствие для allow-листов сервисов.** VPN-трафик приходит к сервисам с адресами `10.8.0.<N>`, поэтому в nftables LXC и VM разрешается VPN-подсеть целиком (`10.8.0.0/24`), наравне с MGMT. То же в middleware `trusted` на Traefik, в whitelist CrowdSec и в `hosts allow` Samba. Адресная политика по пирам живёт только на OPNsense: nftables отвечает на вопрос «из каких зон меня можно достичь», OPNsense — «кто куда имеет право ходить». Такое разделение сохраняет nftables-шаблон единым для всех хостов и локализует изменения при добавлении пира.

**Хосты в INFRA.** Хост в одном сегменте с AmneziaWG-LXC получает пакеты VPN-клиентов напрямую по L2, минуя OPNsense, а отвечает через свой шлюз по умолчанию. OPNsense не видел начала соединения и отбрасывает ответ: TCP зависает на установке. Поэтому на таких хостах задаётся собственный маршрут `10.8.0.0/24 via 192.168.20.11`. Сейчас это Xray VM `192.168.20.12`: маршрут задан netplan drop-in `/etc/netplan/60-vpn-route.yaml`, отдельно от сгенерированного cloud-init `50-cloud-init.yaml`.

## 6. nftables

Полный `/etc/nftables.conf`. Whitelist с `policy drop` на input, транзит VPN — в forward. NAT-таблицы нет: VPN-подсеть маршрутизируется. Автозагрузка через `nftables.service` (enabled).

```
#!/usr/sbin/nft -f

flush ruleset

define MGMT_NET = 192.168.10.0/24
define VPN_NET  = 10.8.0.0/24

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;

        # Loopback
        iifname "lo" accept

        # Conntrack
        ct state established,related accept
        ct state invalid drop

        # ICMPv4 - for ping and path MTU discovery
        ip protocol icmp icmp type {
            destination-unreachable,
            time-exceeded,
            parameter-problem,
            echo-request
        } accept

        # ICMPv6 - for NDP
        ip6 nexthdr icmpv6 icmpv6 type {
            destination-unreachable,
            packet-too-big,
            time-exceeded,
            parameter-problem,
            echo-request,
            echo-reply,
            nd-router-advert,
            nd-router-solicit,
            nd-neighbor-advert,
            nd-neighbor-solicit
        } accept

        # SSH - only from MGMT, VPN
        tcp dport 22 ip saddr { $MGMT_NET, $VPN_NET } accept

        # AmneziaWG - inbound VPN connections from the internet (any source)
        udp dport 51820 accept

        # Everything else falls into policy drop
    }

    chain forward {
        type filter hook forward priority filter; policy drop;

        # VPN traffic in both directions
        iifname "awg0" accept
        oifname "awg0" accept
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

`udp dport 51820 accept` без ограничения источника — VPN-подключения приходят с меняющихся внешних адресов. SSH — из MGMT и VPN по общему канону (см. `02-conventions.md`). Forward пропускает транзит клиентов в обе стороны: `iifname "awg0"` — из туннеля в сеть, `oifname "awg0"` — обратно; при маршрутизируемой подсети ответный трафик приходит на LXC как транзитный и без второго правила был бы съеден `policy drop`. IP forwarding включён (`net.ipv4.ip_forward=1`), без него LXC не маршрутизирует трафик клиентов.

## 7. OPNsense

**Порт-форвард.** Firewall → NAT → Destination NAT (Port Forward). Interface WAN, protocol UDP, destination — WAN address port `51820`, redirect target `192.168.20.11:51820`, associated filter rule. Это направляет входящий VPN-трафик с белого IP на AmneziaWG-LXC в INFRA. VPS в этом пути не участвует — доступ прямой по IP дома.

**Шлюз и маршрут.** System → Gateways → Configuration: `AMNEZIA_GW`, interface INFRA, IPv4, адрес `192.168.20.11`, мониторинг отключён (шлюз — контейнер, лишняя зависимость от ICMP-проверки не нужна: при неответе OPNsense пометил бы шлюз как down и снял маршрут). System → Routes → Configuration: сеть `10.8.0.0/24`, шлюз `AMNEZIA_GW`. Без маршрута обратный трафик к клиентам не находит пути.

**Правила интерфейса INFRA.** Политика по пирам задаётся здесь. Anti-lockout в INFRA отсутствует осознанно: сегмент одной ногой торчит в интернет (открытый 51820, исходящие соединения Xray), доступ к веб-морде фаервола хостам INFRA не нужен. Удалённое администрирование OPNsense обеспечивается правилом для ноутбука, где `any` включает This Firewall.

| # | Правило | Source | Destination | Порт |
| --- | --- | --- | --- | --- |
| 1 | Allow VPN notebook full access | `vpn_notebook` (`10.8.0.2`) | any | any |
| 2 | Allow VPN phone to Traefik | `vpn_phone` (`10.8.0.3`) | `traefik_lxc` (`192.168.40.11`) | 443 |
| 3 | Allow INFRA to NTP on firewall | INFRA net | This Firewall | 123/udp |
| 4 | Block access to private networks | INFRA net | `private_networks` | any |
| 5 | Allow INFRA to internet | INFRA net | any | any |

Правила 1–2 адресуют VPN-пиров, которые не входят в `INFRA net` и потому не попадают под блок. Правила 3–5 описывают собственный исходящий трафик хостов INFRA (`amneziawg`, `xray`): им нужен интернет и время, в локальную сеть они не ходят. DNS-запросы клиентов к фаерволу пропускает плавающее правило DNS к This Firewall. Входящие подключения к Xray на `10809` рулятся правилами интерфейсов-источников (SERVICES, DMZ, IOT), а не INFRA.

ICMP к хостам и шлюзам фильтруется, поэтому неотвечающий ping из туннеля не признак неисправности. Работоспособность туннеля проверяется по `latest handshake` в `awg show` и обращением к реальному сервису.

## 8. Автозапуск

Интерфейс `awg0` поднимается при старте LXC через шаблонный юнит `awg-quick@awg0.service` с drop-in, задающим userspace-бэкенд:

```
# /etc/systemd/system/awg-quick@awg0.service.d/override.conf
[Service]
Environment=AWG_QUICK_USERSPACE_IMPLEMENTATION=amneziawg-go
```

Юнит включён в автозапуск (`systemctl enable awg-quick@awg0`). nftables (`nftables.service`) и sysctl (`ip_forward`) тоже персистентны, поэтому после перезагрузки LXC весь VPN-стек поднимается сам, без ручных действий.

## 9. SSH

SSH ужесточён общим drop-in `/etc/ssh/sshd_config.d/10-hardening.conf` (см. `02-conventions.md`). Доступ к порту 22 ограничен на nftables источниками MGMT и VPN. Аутентификация — по ключам.

## 10. Зависимости

- **OPNsense** — порт-форвард UDP 51820 на LXC (вход снаружи), шлюз и статический маршрут `10.8.0.0/24` (обратный путь), правила интерфейса INFRA (политика доступа по пирам), маршрутизация VPN-трафика между сегментами, Unbound как DNS для клиентов.
- **Белый IP дома** — endpoint для клиентов; VPN завязан на прямую доступность WAN-адреса дома.
- **Хосты INFRA** — собственный маршрут `10.8.0.0/24 via 192.168.20.11` на каждом хосте сегмента, кроме самого AmneziaWG-LXC.
- **Allow-листы сервисов** — VPN-подсеть `10.8.0.0/24` в nftables хостов, middleware `trusted` Traefik, whitelist CrowdSec, `hosts allow` Samba.
- **Dev VM** — сборка `amneziawg-go` при обновлении.

## 11. Ключевые особенности

- **Прямой вход, минуя VPS.** AmneziaWG доступен по белому IP дома через порт-форвард на OPNsense. VPS используется только для публикации сервисов (HTTP через WG-туннель к Traefik), к VPN-доступу отношения не имеет.
- **AmneziaWG 3.1.** Случайные заголовки из диапазонов, паддинг всех типов пакетов и шифрование заголовков меняют профиль всего соединения, а не только рукопожатия. Совпадать на обеих сторонах должны `S1–S4`, `H1–H4` и `HeaderProtectionKey`; `Jc/Jmin/Jmax` и `I1–I5` — только на клиенте.
- **Установка из исходников.** Пакетов для Debian 12 нет: `amneziawg-go` собирается статически на Dev VM, `amneziawg-tools` — в LXC. Обновление — только из дома, со снимком LXC и сохранением старых бинарников.
- **Userspace-реализация в LXC.** Kernel-модуль в контейнер не загрузить — интерфейс работает через `amneziawg-go`. Требует userspace-переменной при подъёме (учтено в systemd drop-in).
- **Split-режим у всех клиентов.** Через туннель — только домашние сети, а на ноутбуке и DNS только для домашних доменов. Неработающий туннель не отключает клиенту интернет.
- **Маршрутизация вместо маскарада.** Клиенты сохраняют адрес `10.8.0.<N>`, что делает пиров различимыми на firewall и даёт Traefik реальные адреса вместо адреса LXC. Цена — статический маршрут на OPNsense, маршрут на хостах INFRA и VPN-подсеть в allow-листах сервисов.
- **Адрес в туннеле — криптографическая идентичность.** `AllowedIPs` привязывает source-адрес к приватному ключу пира; подделка невозможна, внешний IP клиента роли не играет. Это и позволяет строить политику доступа по адресам VPN-подсети.
- **Разделение слоёв фильтрации.** nftables сервисов разрешают VPN-подсеть целиком (зона), адресная политика по пирам — только на OPNsense (маршрутизация). Добавление пира не требует правки хостов.
- **Параметры обфускации не публикуются.** Уникальный набор параметров — часть защиты от DPI; в документации только структура, значения — в `awg0.conf` и `/root/awg-v3-params`.
