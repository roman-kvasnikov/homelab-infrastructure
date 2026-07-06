---
name: edge-vps
description: |
  Публичная точка входа: внешний VPS проксирует HTTPS-трафик на домашний Traefik через служебный WireGuard-туннель. VPS работает на L4 (nginx stream + PROXY protocol), TLS не терминирует и сертификаты не хранит. Документ описывает роль VPS, nginx stream, WG-туннель и сторону Traefik. Используй для вопросов по публичному доступу к сервисам извне, VPS, WG-туннелю.
---

# Публичный вход через VPS

Публичные сервисы (`*.kvasok.xyz`) доступны из интернета через внешний VPS. На домашнем роутере наружу не проброшено ни одного порта — входящий трафик терминируется на VPS, а домашний Traefik сам инициирует исходящий WireGuard-туннель к VPS. VPS работает исключительно на L4: проксирует TCP-поток 443 в туннель, не зная про SNI и не терминируя TLS.

Этот путь отвечает **только** за публикацию HTTP-сервисов. Удалённый доступ в саму сеть (VPN) организован отдельно через AmneziaWG в INFRA и к VPS отношения не имеет (см. `vpn-amneziawg.md`).

## 1. Поток трафика

```
[Клиент из интернета] --HTTPS 443--> [VPS: nginx stream]
      --PROXY protocol / WG-туннель--> [Traefik 192.168.40.11:443]
            --> терминация TLS, middleware, маршрутизация на бэкенд
```

Клиент подключается к публичному адресу VPS на 443. Nginx на VPS прозрачно перекидывает TCP-поток в WireGuard-туннель к Traefik, добавляя PROXY protocol (чтобы Traefik видел реальный IP клиента). TLS терминирует уже Traefik дома — VPS сертификаты не хранит.

## 2. VPS

Внешний Linux-VPS (Timeweb), единственная точка входа публичного трафика. Firewall — nftables. Сертификаты не хранит, TLS не терминирует. Держит служебный WireGuard-туннель к домашнему Traefik.

### 2.1. Открытые порты

| Порт | Назначение |
| :--- | :--- |
| SSH (нестандартный) | Администрирование, доступ только по ключу |
| 443/tcp | HTTPS, проксируется в WG-туннель на Traefik |
| 51820/udp | WireGuard — служебный туннель к Traefik |

Порт 80 не открыт: ACME-валидация идёт по DNS-01 на Traefik, HTTP-challenge не нужен.

### 2.2. nginx (L4 stream)

Nginx работает только в режиме `stream` (L4). Он не разбирает SNI и не терминирует TLS — прозрачно перекидывает 443 в WireGuard-туннель с включённым **PROXY protocol**, чтобы Traefik получал реальный IP клиента, а не адрес VPS.

```nginx
stream {
    log_format proxy '$remote_addr [$time_local] '
                     '$protocol $status $bytes_sent $bytes_received '
                     '$session_time -> $upstream_addr';

    access_log /var/log/nginx/stream.log proxy;

    server {
        listen 443 reuseport;
        proxy_pass 10.0.0.2:443;      # Traefik inside the WG tunnel
        proxy_protocol on;
        proxy_connect_timeout 3s;
        proxy_timeout 300s;
    }
}
```

`proxy_pass` указывает на адрес Traefik внутри туннеля (`10.0.0.2`), `proxy_protocol on` добавляет заголовок PROXY protocol с реальным IP клиента.

### 2.3. WireGuard-туннель

Служебный туннель между VPS (`10.0.0.1`) и Traefik (`10.0.0.2`). Инициирует соединение **домашний Traefik** (исходящее), поэтому на домашнем роутере не нужен проброс портов — туннель поднимается изнутри наружу. По туннелю идёт только публичный HTTPS-трафик (проксируемый nginx'ом на VPS в сторону Traefik). VPN-клиентов через этот туннель нет — VPN организован отдельно через AmneziaWG.

## 3. Сторона Traefik

Домашний Traefik (`192.168.40.11`, DMZ) — второй конец туннеля и точка терминации TLS. Детали Traefik (домены, сертификаты, middleware) — в `ingress-traefik.md`; здесь только то, что относится к стыку с VPS.

Traefik держит WireGuard-интерфейс к VPS (адрес `10.0.0.2` в туннеле). Entrypoint `websecure` (443) принимает PROXY protocol **только** от VPS внутри туннеля: `proxyProtocol.trustedIPs: ["10.0.0.1"]`. Это гарантирует, что подделать реальный IP клиента через заголовок PROXY protocol может только доверенный сосед (VPS), а не произвольный источник.

Аналогично Traefik доверяет `X-Forwarded-*` заголовкам только от адреса VPS в туннеле (`forwardedHeadersTrustedIPs: 10.0.0.1`). На nftables Traefik порт 443 на интерфейсе туннеля открыт для адреса VPS (`10.0.0.1`).

## 4. Домен и сертификаты

Основной домен — `kvasok.xyz`, wildcard `*.kvasok.xyz` указывает на публичный адрес VPS. Сертификаты Let's Encrypt получает **Traefik** через DNS-01 challenge (wildcard). VPS сертификаты не хранит и в выдаче не участвует — он работает на L4, не видя TLS. Детали ACME — в `ingress-traefik.md`.

Внутри сети те же имена резолвятся split-horizon (Unbound) в локальный адрес Traefik (`192.168.40.11`), минуя VPS — см. `home-network.md`.

## 5. Ключевые особенности

- **VPS — только L4-вход, ничего больше.** Nginx stream проксирует 443 в туннель с PROXY protocol. TLS не терминирует, сертификаты не хранит, VPN не обслуживает. Минимальная поверхность: по сути перекладчик пакетов.
- **Туннель инициирует дом.** Traefik сам поднимает исходящий WG к VPS — на домашнем роутере нет проброшенных портов, входящих соединений в дом извне напрямую нет.
- **PROXY protocol доверяется только VPS.** Traefik принимает PROXY protocol и `X-Forwarded-*` исключительно от адреса VPS в туннеле (`10.0.0.1`), что не даёт подделать IP клиента.
- **VPN отделён от публичного входа.** Удалённый доступ в сеть — через AmneziaWG (INFRA), не через VPS. VPS и VPN — независимые пути (см. `vpn-amneziawg.md`).
