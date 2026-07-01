# NaïveProxy — форк [Haonixao](https://github.com/Haonixao/naiveproxy)

> Форк [klzgrad/naiveproxy](https://github.com/klzgrad/naiveproxy) для работы в связке с [Naive Server](https://github.com/Haonixao/naive-server) в **Stealth Mode**. Оригинальное описание проекта — ниже, после разделителя.

## Отличия от upstream

Этот форк добавляет две группы изменений поверх официального клиента. Они **не** входят в `klzgrad/naiveproxy` и нужны для stealth-сценария с самоподписанными сертификатами и расширенным паддингом.

### 1. Отключена проверка TLS-сертификатов (TCP и QUIC)

Официальный NaïveProxy, как и Chromium, отклоняет соединения с недоверенными, самоподписанными или невалидными по сроку сертификатами. В stealth-режиме Naive Server отдаёт именно такой сертификат — поэтому стандартный клиент **несовместим** с Naive Server.

В форке проверка отключена на всех уровнях, необходимых для обоих транспортов:

| Транспорт           | Файл                                               | Что сделано                                                                                                        |
| :------------------ | :------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------- |
| **HTTPS (TCP/TLS)** | `src/net/socket/ssl_client_socket_impl.cc`         | Обход в `DoHandshake`, `DoHandshakeComplete`, `HandleVerifyResult`; `IsAllowedBadCert` всегда разрешает сертификат |
| **QUIC (UDP)**      | `src/net/third_party/quiche/.../tls_handshaker.cc` | `VerifyCert()` сразу возвращает `ssl_verify_ok`                                                                    |
| **QUIC (UDP)**      | `src/net/quic/crypto/proof_verifier_chromium.cc`   | `DoVerifyCert` / `DoVerifyCertComplete` пропускают верификацию цепочки                                             |

**Практический эффект:** клиент подключается к `naive_server -mode stealth` по `https://` и `quic://` без ограничений на работу с самоподписанными сертификатами.

**Совместимость:** с `naive_server -mode official` (Let's Encrypt) форк тоже работает. Для official-режима можно использовать и стандартный `klzgrad/naiveproxy`.

### 2. Padding Variant2 — паддинг на всём потоке

Официальный протокол паддинга (**Variant1**, wire-format `"1"`) набрасывает случайные байты только на **первые 8** операций чтения/записи в CONNECT-потоке (`kFirstPaddings = 8`). Дальше трафик идёт без паддинга — так устроены `klzgrad/naiveproxy` и Caddy forwardproxy.

Форк добавляет **Variant2** (`"2"`):

|                           | Variant1 (upstream)         | Variant2 (форк)       |
| :------------------------ | :-------------------------- | :-------------------- |
| Область действия          | первые 8 read/write         | **каждая** read/write |
| Оверхед                   | низкий                      | выше                  |
| Защита от length-analysis | на старте потока            | на всём потоке        |
| Сервер                    | `-padding 1` (по умолчанию) | `-padding 2`          |

Изменённые файлы: `naive_protocol.h/cc`, `naive_padding_socket.cc`, `naive_proxy_bin.cc`.

Клиент объявляет серверу поддержку `{Variant1, Variant2, None}`; тип согласуется через заголовки `padding-type-request` / `padding-type-reply`. Для Variant2 на сервере нужен `naive_server -padding 2` — со стандартным Caddy forwardproxy Variant2 **не работает**.

### Когда какой клиент использовать

| Сценарий                                | Клиент                        |
| :-------------------------------------- | :---------------------------- |
| Naive Server, stealth (`-mode stealth`) | **Этот форк**                 |
| Naive Server, stealth + `-padding 2`    | **Этот форк**                 |
| Naive Server, official (Let's Encrypt)  | Форк или `klzgrad/naiveproxy` |
| Caddy / HAProxy forwardproxy            | `klzgrad/naiveproxy`          |

### Сборка

Сборка как у upstream — см. [.github/workflows/build.yml](.github/workflows/build.yml). Готовое Docker-окружение для сборки клиента: [naive-server/naiveproxy_in_docker](https://github.com/Haonixao/naive-server/tree/main/naiveproxy_in_docker).

## ⚠️ Предупреждение о безопасности

Этот форк **намеренно менее безопасен**, чем официальный `klzgrad/naiveproxy`, в части **проверки личности сервера**: цепочка CA, срок действия, имя (SNI/CN) — всё отключено для TCP/TLS и QUIC. Клиент примет любой сертификат от того endpoint, с которым завершит handshake.

Это нужно для stealth-режима с самоподписанным сертификатом, но убирает криптографическую гарантию «на другом конце именно мой сервер, а не чужой».

**Какой риск реален:** подмена сервера **в момент подключения** — если DPI, провайдер или ошибка в `host-resolver-rules`/маршруте направит вас на **чужой эмулятор**, клиент подключится к нему без предупреждения. Это impersonation на этапе handshake, не «вклинивание» в уже идущий поток.

**Какой риск не создаётся этим патчем:** вставиться в середину TLS-сессии с **вашим реальным** `naive_server` и читать/менять трафик. Для этого атакующему всё равно нужно терминировать TLS у себя ещё на handshake — просто отключение проверки сертификатов само по себе не открывает канал для mid-stream splice.

**Что остаётся в любом случае:** после успешного handshake трафик шифруется TLS; пассивный прослушиватель линии без участия в handshake его не расшифрует — ни с проверкой сертификатов, ни без неё.

Используйте форк, если принимаете компромисс «доверяю тому, куда меня направили при подключении». Если нужна строгая аутентификация сервера — `naive_server -mode official` с валидным сертификатом и стандартный `klzgrad/naiveproxy`.

---

# NaïveProxy (upstream) ![build workflow](https://github.com/klzgrad/naiveproxy/actions/workflows/build.yml/badge.svg)

NaïveProxy uses Chromium's network stack to camouflage traffic with strong censorship resistence and low detectablility. Reusing Chrome's stack also ensures best practices in performance and security.

The following traffic attacks are mitigated by using Chromium's network stack:

* Website fingerprinting / traffic classification: mitigated by [traffic multiplexing in HTTP/2](https://arxiv.org/abs/1707.00641) and parroting preambles.
* [TLS parameter fingerprinting](https://arxiv.org/abs/1607.01639): defeated by reusing [Chrome's network stack](https://www.chromium.org/developers/design-documents/network-stack).
* [Active probing](https://ensa.fi/active-probing/): defeated by *application fronting*, i.e. hiding proxy servers behind a commonly used frontend server with application-layer routing.
* Length-based traffic analysis: mitigated by padding and fragmentation.

## Architecture

[Browser → Naïve client] ⟶ Censor ⟶ [Frontend → Naïve server] ⟶ Internet

NaïveProxy uses Chromium's network stack to parrot traffic between regular Chrome browsers and standard frontend servers.

The frontend server can be any well-known reverse proxy that is able to route HTTP/2 traffic based on HTTP authorization headers, preventing active probing of proxy existence. Known ones include Caddy with its forwardproxy plugin and HAProxy.

The Naïve server here works as a forward proxy and a packet length padding layer. Caddy forwardproxy is also a forward proxy but it lacks a padding layer. A [fork](https://github.com/klzgrad/forwardproxy) adds the NaïveProxy padding layer to forwardproxy, combining both in one.

## Download NaïveProxy

Download [here](https://github.com/klzgrad/naiveproxy/releases/latest). Supported platforms include: Windows, Android (with [Exclave](https://github.com/dyhkwong/Exclave), [husi](https://github.com/xchacha20-poly1305/husi), [NekoBox](https://github.com/MatsuriDayo/NekoBoxForAndroid)), Linux, Mac OS, and OpenWrt ([support status](https://github.com/klzgrad/naiveproxy/wiki/OpenWrt-Support)).

Users should always use the latest version to keep signatures identical to Chrome.

Build from source: Please see [.github/workflows/build.yml](https://github.com/klzgrad/naiveproxy/blob/master/.github/workflows/build.yml).

## Server setup

The following describes the naïve fork of Caddy forwardproxy setup.

Download [here](https://github.com/klzgrad/forwardproxy/releases/latest) or build from source:
```sh
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
~/go/bin/xcaddy build --with github.com/caddyserver/forwardproxy=github.com/klzgrad/forwardproxy@naive
```

Example Caddyfile (replace `user` and `pass` accordingly):
```
{
  order forward_proxy before file_server
  log {
    exclude http.log.error # Avoid logging user activity
  }
}
:443, example.com {
  tls me@example.com
  encode
  forward_proxy {
    basic_auth user pass
    hide_ip
    hide_via
    probe_resistance
  }
  file_server {
    root /var/www/html
  }
}
```
`:443` must appear first for this Caddyfile to work. See Caddyfile [docs](https://caddyserver.com/docs/caddyfile/directives/tls) for customizing TLS certificates. For more advanced usage consider using [JSON for Caddy 2's config](https://caddyserver.com/docs/json/).

Run with the Caddyfile:
```
sudo setcap cap_net_bind_service=+ep ./caddy
./caddy start
```

See also [Systemd unit example](https://github.com/klzgrad/naiveproxy/wiki/Run-Caddy-as-a-daemon) and [HAProxy setup](https://github.com/klzgrad/naiveproxy/wiki/HAProxy-Setup).

## Client setup

Run `./naive` with the following `config.json` to get a SOCKS5 proxy at local port 1080.
```json
{
  "listen": "socks://127.0.0.1:1080",
  "proxy": "https://user:pass@example.com"
}
```

Or `quic://user:pass@example.com`, if it works better. See also [parameter usage](https://github.com/klzgrad/naiveproxy/blob/master/USAGE.txt) and [performance tuning](https://github.com/klzgrad/naiveproxy/wiki/Performance-Tuning).

## Third-party integration

* [v2rayN](https://github.com/2dust/v2rayN), GUI client

## Notes for downstream

Do not use the master branch to track updates, as it rebases from a new root commit for every new Chrome release. Use stable releases and the associated tags to track new versions, where short release notes are also provided.

## Padding protocol, an informal specification

The design of this padding protocol opts for low overhead and easier implementation, in the belief that proliferation of expendable, improvised circumvention protocol designs is a better logistical impediment to censorship research than sophisicated designs.

### Proxy payload padding

NaïveProxy proxies bidirectional streams through HTTP/2 (or HTTP/3) CONNECT tunnels. The bidirectional streams operate in a sequence of reads and writes of data. The first `kFirstPaddings` (8) reads and writes in a bidirectional stream after the stream is established are padded in this format:
```c
struct PaddedData {
  uint8_t original_data_size_high;  // original_data_size / 256
  uint8_t original_data_size_low;  // original_data_size % 256
  uint8_t padding_size;
  uint8_t original_data[original_data_size];
  uint8_t zeros[padding_size];
};
```
`padding_size` is a random integer uniformally distributed in [0, `kMaxPaddingSize`] (`kMaxPaddingSize`: 255). `original_data_size` cannot be greater than 65535, or it has to be split into several reads or writes.

`kFirstPaddings` is chosen to be 8 to flatten the packet length distribution spikes formed from common initial handshakes:
- Common client initial sequence: 1. TLS ClientHello; 2. TLS ChangeCipherSpec, Finished; 3. H2 Magic, SETTINGS, WINDOW_UPDATE; 4. H2 HEADERS GET; 5. H2 SETTINGS ACK.
- Common server initial sequence: 1. TLS ServerHello, ChangeCipherSpec, ...; 2. TLS Certificate, ...; 3. H2 SETTINGS; 4. H2 WINDOW_UPDATE; 5. H2 SETTINGS ACK; 6. H2 HEADERS 200 OK.

Further reads and writes after `kFirstPaddings` are unpadded to avoid performance overhead. Also later packet lengths are usually considered less informative.

### H2 RST_STREAM frame padding

In experiments, NaïveProxy tends to send too many RST_STREAM frames per session, an uncommon behavior from regular browsers. To solve this, an END_STREAM DATA frame padded with total length distributed in [48, 72] is prepended to the RST_STREAM frame so it looks like a HEADERS frame. The server often replies to this with a WINDOW_UPDATE because padding is accounted in flow control. Whether this results in a new uncommon behavior is still unclear.

### H2 HEADERS frame padding

The CONNECT request and response frames are too short and too uncommon. To make its length similar to realistic HEADERS frames, a `padding` header is filled with a sequence of symbols that are not Huffman coded and are pseudo-random enough to avoid being indexed. The length of the padding sequence is randomly distributed in [16, 32] (request) or [30, 62] (response).

### Opt-in of padding protocol

NaïveProxy clients should interoperate with any regular HTTP/2 proxies unaware of this padding protocol. NaïveProxy servers (i.e. any proxy server capable of the this padding protocol) should interoperate with any regular HTTP/2 clients (e.g. regular browsers) unaware of this padding protocol.

NaïveProxy servers and clients determines whether the counterpart is capable of this padding protocol by the presence of the `padding` header in the CONNECT request and response respectively. The padding procotol is enabled only if the `padding` header exists.

The first CONNECT request to a server cannot use "Fast Open" to send payload before response, because the server's padding capability has not been determined from the first response and it's unknown whether to send padded or unpadded payload for Fast Open.

## Changes from Chromium upstream

- Minimize source code and build size (0.3% of the original)
- Disable exceptions and RTTI, except on Mac and Android.
- Support OpenWrt builds
- (Android, Linux) Use the builtin verifier instead of the system verifier (drop dependency of NSS on Linux) and read the system trust store from (following Go's behavior in crypto/x509/root_unix.go and crypto/x509/root_linux.go):
  - The file in environment variable SSL_CERT_FILE
  - The first available file of
    -  /etc/ssl/certs/ca-certificates.crt (Debian/Ubuntu/Gentoo etc.)
    -  /etc/pki/tls/certs/ca-bundle.crt (Fedora/RHEL 6)
    -  /etc/ssl/ca-bundle.pem (OpenSUSE)
    -  /etc/pki/tls/cacert.pem (OpenELEC)
    -  /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem (CentOS/RHEL 7)
    -  /etc/ssl/cert.pem (Alpine Linux)
  - Files in the directory of environment variable SSL_CERT_DIR
  - Files in the first available directory of
    -  /etc/ssl/certs (SLES10/SLES11, https://golang.org/issue/12139)
    -  /etc/pki/tls/certs (Fedora/RHEL)
    -  /system/etc/security/cacerts (Android)
- Handle AIA response in PKCS#7 format
- Allow higher socket limits for proxies
- Force tunneling for all sockets
- Support HTTP/2 and HTTP/3 CONNECT tunnel Fast Open using the `fastopen` header
- Pad RST_STREAM frames
