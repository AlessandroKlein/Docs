---
tags:
  - sema
  - red
  - conectividad
---

# Conectividad y red

> **Tipo:** Concepto | **Estado:** Estable
> **Fecha:** 2026-10-08
> **Firmware:** v1.103.0

Cómo SEMA se conecta: **WiFi** (STA/AP), **mDNS**, **Ethernet** (LAN8720A por RMII
o W5500 por SPI) y **hora/NTP**. Toda la lógica está en
`src/core/network/WiFiManager.cpp`, `src/core/EthernetManager.cpp`,
`include/core/Time.hpp` y `include/hw/HwProfile.hpp`; la configuración, en
`include/core/ConfigManager.hpp`.

## 1. Modos de WiFi

`WiFiManager::begin(mode, ssid, password, hostname, ip, gateway, subnet, dns, mdns)`
se llama **una sola vez** en `SemaCore::setup()` con los valores de
`config.network`. La decisión es literal (`WiFiManager.cpp:12–30`):

```text
if (mode == "STA" && ssid.length() > 0)  → WiFi.mode(WIFI_STA) + WiFi.begin(ssid[,password])
else                                     → startAp(hostname)   (punto de acceso de emergencia)
```

| Situación | Modo resultante | IP local |
|-----------|-----------------|----------|
| `mode = "STA"` y `ssid` no vacío | **STA** (`WIFI_STA`) | `WiFi.localIP()` (DHCP o estática) |
| `mode = "AP"` | **AP** (`WIFI_AP`) | `WiFi.softAPIP()` — SEMA no fija la IP del AP: es la del core de Arduino |
| `mode = "STA"` con `ssid` vacío | **AP** (cae al AP de emergencia) | `WiFi.softAPIP()` |
| Cualquier otro `mode` | Rechazado por `ConfigManager::validate()` (solo `STA`/`AP`) | — |

Detalles verificados del AP de emergencia (`startAp`):

- SSID del AP = `network.hostname` o `"SEMA"` si el hostname está vacío.
- `WiFi.softAP(ap.c_str())` **sin contraseña → AP abierto**. No hay clave de AP
  configurable.
- No hay portal cautivo ni DNS propio: hay que entrar a mano a la IP que reporta
  `GET /api/v1/network` (la del AP del core de Arduino).

❌ **No existe un modo "WiFi off".** El enum de modos es `STA` | `AP` y
`validate()` rechaza cualquier otro valor; tampoco hay forma de apagar el radio
desde la API o la web. Deshabilitar el WiFi requeriría un tercer valor de
`network.mode` más una rama en `WiFiManager::begin`.

### 1.1 IP estática (solo en STA)

```cpp
if (ip.length() > 0 && gateway.length() > 0 && subnet.length() > 0) {
  if (aip.fromString(ip) && agw.fromString(gateway) &&
      asub.fromString(subnet) && adns.fromString(dns)) {
    WiFi.config(aip, agw, asub, adns);
  }
}
```

⚠️ Se exige que **los cuatro** valores parseen: si `dns` está vacío o mal formado,
`fromString(dns)` falla y **no se aplica ninguna IP estática** — se sigue usando
DHCP sin aviso. No hay validación en la web ni error HTTP por esto.

## 2. Reconexión con backoff

`WiFiManager::loop()` corre en cada iteración del `SemaCore::loop()`
(y por lo tanto con el watchdog alimentado). Su lógica (`WiFiManager.cpp:70–86`):

| Estado | Acción |
|--------|--------|
| STA y `WiFi.status() != WL_CONNECTED` | Si pasaron `reconnectInterval_` ms: `WiFi.reconnect()`, incrementa el contador y **duplica** el intervalo |
| STA y conectado (con intentos previos) | Reinicia `reconnectAttempts_ = 0` y `reconnectInterval_ = 2000` |
| AP | No hace nada: el AP no "se reconecta" |

Progresión real del backoff (tope **60.000 ms**):

```text
2 s → 4 s → 8 s → 16 s → 32 s → 60 s → 60 s → 60 s …
```

`connected()` en modo AP devuelve **siempre `true`** (el AP siempre está arriba) y
`rssi()` devuelve **0**: en AP no hay señal que medir.

## 3. mDNS (`hostname.local`)

Se activa solo si `mdns == true` **y** `hostname` no está vacío
(`WiFiManager.cpp:34–48`):

1. El nombre se **sanea**: cada carácter que no sea alfanumérico o `-` se reemplaza
   por `-`; si el resultado queda vacío, se usa `"sema"`.
2. `MDNS.begin(safe)` — si falla, `mdnsStarted_` queda en `false`.
3. Si arrancó: `MDNS.addService("http", "tcp", 80)`.

Así, con `hostname = "sema-001"` la estación responde en `http://sema-001.local/`.

| Dato | Valor |
|------|-------|
| Default de `network.hostname` | `"sema-001"` (`ConfigManager.cpp:60`) |
| Default de `network.mdns` | `true` |
| Servicio anunciado | `_http._tcp` en el puerto **80** (el único) |
| Servicio propio (`_sema._tcp`) | ❌ No existe |
| Visibilidad | Solo en la **LAN**: mDNS no cruza routers ni funciona sin red local |
| Estado consultable | `wifi_mdns` y `wifi_host` en `GET /api/v1/system` |

⚠️ **El hostname no se publica por DHCP**: en todo el repositorio no hay ninguna
llamada a `WiFi.setHostname()` ni a `esp_netif_set_hostname()`. El router verá el
nombre por defecto del core (`espressif` / `ESP-XXXXXX`), no `sema-001`. El
`hostname` se usa **solo** para mDNS y como SSID del AP de emergencia.

## 4. Ethernet

Hay **dos caminos** elegidos en tiempo de compilación con `SEMA_NATIVE_ETH`
(`include/hw/HwProfile.hpp:32–45`); ambos terminan integrados a **lwIP**, la misma
pila que usa el WiFi, por lo que el `WebServer` sirve sobre Ethernet sin cambios
(`src/core/EthernetManager.cpp`).

| Board (`build_flags`) | `SEMA_BOARD_ID` | Flash | Chip Ethernet | Capa física | Driver |
|-----------------------|-----------------|-------|---------------|-------------|--------|
| `BOARD_ESP32_WROOM` | `esp32-wroom-4mb` | 4 MB | MAC **nativa** EMAC | **LAN8720A** RMII | `ETH.h` (Arduino core) |
| `BOARD_ESP32_WROOM32U` | `esp32-wroom32u-16mb` | 16 MB | MAC nativa EMAC | LAN8720A RMII | `ETH.h` |
| `BOARD_ESP32_S3` | `esp32-s3-8mb` | 8 MB | sin MAC nativa | **W5500** SPI | `esp_eth` (ESP-IDF) + lwIP |

### 4.1 LAN8720A (RMII, MAC nativa)

```cpp
ETH.begin(cfg_.phyAddr, cfg_.powerPin, cfg_.mdcPin, cfg_.mdioPin,
          ETH_PHY_LAN8720, ETH_CLOCK_GPIO0_IN);
```

| Pin | GPIO | Configurable |
|-----|------|--------------|
| `MDC` | 23 | Sí (`ethernet.mdc`) |
| `MDIO` | 18 | Sí (`ethernet.mdio`) |
| `PHY_ADDR` | 1 | Sí (`ethernet.phy_addr`, solo por `PUT /api/v1/config`) |
| Alimentación de PHY | `-1` (sin control) | Sí (`ethernet.power`) |
| `TXD0` | 19 | ❌ Fijo del EMAC |
| `TXD1` | 22 | ❌ Fijo |
| `TX_EN` | 21 | ❌ Fijo |
| `RXD0` | 25 | ❌ Fijo |
| `RXD1` | 26 | ❌ Fijo |
| `CRS_DV` | 27 | ❌ Fijo |
| `RX_ER` | 13 | ❌ Fijo |
| `REF_CLK` | 0 | ❌ Fijo (reloj de 50 MHz, `ETH_CLOCK_GPIO0_IN`) |

Los 10 pines RMII se reservan y se informan en `reserved_pins` de
`GET /api/v1/system` **solo** si `SEMA_NATIVE_ETH && SEMA_USE_ETHERNET` y
`ethernet.enabled` es verdadero. Recuperar el reloj en GPIO0 significa que el
ESP32 no puede arrancar en modo flash/boot con la PHY alimentada (detalle de
diseño de la placa).

### 4.2 W5500 (SPI, sin MAC nativa)

Secuencia real del driver (`EthernetManager.cpp:44–107`):

| Paso | Detalle |
|------|---------|
| 1. Bus SPI | `spi_bus_initialize(SEMA_ETH_SPI_HOST, …, SPI_DMA_CH_AUTO)`, `max_transfer_sz = 4000` |
| 2. Dispositivo | `command_bits = 16` (dirección), `address_bits = 8` (control), `mode = 0`, **20 MHz**, `queue_size = 20`, `spics = ethernet.cs` |
| 3. MAC | `esp_eth_mac_new_w5500()`, `smi_mdc/mdio = -1`, `int_gpio_num = ethernet.irq` |
| 4. PHY | `esp_eth_phy_new_w5500()`, `phy_addr`, `reset_gpio_num = ethernet.rst` |
| 5. Driver | `esp_eth_driver_install()` |
| 6. lwIP | `esp_netif_new(ESP_NETIF_DEFAULT_ETH)` + `esp_eth_set_default_handlers` + `esp_netif_attach` |
| 7. Arranque | `esp_eth_start()` → **DHCP** |

`SEMA_ETH_SPI_HOST` es **2 (`SPI3_HOST`)** en el S3 — periférico dedicado, separado
del SPI de Arduino (SPI2/FSPI) que usa LoRa/RadioLib. Por eso Ethernet y LoRa
conviven: cada uno tiene su CS.

| Pin | Default | Configurable por `POST /config/buses` |
|-----|---------|----------------------------------------|
| `ethernet.cs` | 5 | Sí |
| `ethernet.irq` | 4 | ❌ (solo `PUT /config`) |
| `ethernet.rst` | -1 | ❌ |
| `ethernet.sck` | 18 | ❌ |
| `ethernet.miso` | 19 | ❌ |
| `ethernet.mosi` | 21 | ❌ |

### 4.3 Prioridad WiFi / Ethernet

No hay ninguna lógica de prioridad, ruteo o *failover* en el código:

- `WiFiManager::begin()` y `EthernetManager::apply()` corren **independientemente**
  en `SemaCore::setup()`; Ethernet solo si `ethernet.enabled`.
- El `WebServer` escucha en todas las interfaces de lwIP: la web responde igual por
  la IP del WiFi y por la IP de Ethernet (si ambas están arriba).
- `GET /api/v1/network` reporta los dos por separado (`ip`/`rssi` de WiFi y el
  sub-objeto `ethernet` con `enabled`, `connected`, `ip`). El firmware **no declara**
  una interfaz como primaria ni expone la ruta elegida para salir a Internet.
- mDNS se registra únicamente sobre la interfaz WiFi (`MDNS.begin` en
  `WiFiManager::begin`): **`hostname.local` no resuelve si solo hay Ethernet**.
- Ethernet **no admite IP estática**: `EthernetConfig` no tiene campos de IP y la
  configuración solo llega a lwIP por DHCP.

⚠️ **No verificable:** cuál de las dos interfaces usa lwIP como salida por defecto
cuando ambas tienen ruta (depende del orden de registro de `netif` en lwIP, que el
firmware no fija explícitamente).

### 4.4 `SEMA_USE_ETHERNET` y estado

- `SEMA_USE_ETHERNET = 0` saca todo el `EthernetManager` del build: el `#if` envuelve
  el `.cpp` completo y el miembro `ethernet_` + el accesor `ethernet()` de
  `SemaCore.hpp`. ⚠️ **Pero `HttpServer::onNetwork()` (~línea 2009) usa
  `core_->ethernet()` sin condicionar por el flag**: con `SEMA_USE_ETHERNET=0` ese
  símbolo no existe y la compilación fallaría. En la práctica no se nota porque las
  cuatro placas de `platformio.ini` compilan con `SEMA_USE_ETHERNET=1`; queda como
  deuda si algún día se quiere un build sin Ethernet.
- `connected()`: nativo → `ready_ && ETH.linkUp()`; W5500 → IP de `esp_netif`
  distinta de 0.
- `loop()` de Ethernet está vacío: **lwIP gestiona el DHCP solo**.

## 5. Hora y NTP

La sincronización se hace **una vez**, en `SemaCore::setup()` (`SemaCore.cpp:161–162`):

```cpp
configTzTime(posixTz(config_.get().system.timezone),
             config_.get().system.ntpServer.c_str(), "time.nist.gov");
```

- **Dos servidores**: el configurado (`system.ntp_server`, default `pool.ntp.org`) y
  `time.nist.gov` como respaldo fijo.
- La zona horaria IANA del config se traduce a una cadena **POSIX TZ** con
  `posixTz()` (`SemaCore.cpp:21–31`). Las que el firmware entiende:

| `system.timezone` | POSIX TZ | Offset |
|-------------------|----------|--------|
| `America/Argentina/Buenos_Aires` | `<-03>3` | -3 fijo |
| `America/Sao_Paulo` | `<-03>3` | -3 fijo |
| `America/Santiago` | `<-04>4<-03>,M9.1.0/24,M4.1.0/24` | -4 / -3 (DST) |
| `America/Bogota` | `<-05>5` | -5 fijo |
| `America/Mexico_City` | `<-06>6` | -6 fijo |
| `America/New_York` | `EST5EDT,M3.2.0,M11.1.0` | -5 / -4 |
| `America/Los_Angeles` | `PST8PDT,M3.2.0,M11.1.0` | -8 / -7 |
| `Europe/Madrid` · `Europe/Berlin` | `CET-1CEST,M3.5.0,M10.5.0/3` | +1 / +2 |
| `Europe/London` | `GMT0BST,M3.5.0/1,M10.5.0` | 0 / +1 |
| Cualquier otra (p. ej. `UTC`, `Asia/Tokyo`, `Australia/Sydney`) | `UTC0` | **0** |

⚠️ `Asia/Tokyo` y `Australia/Sydney` aparecen en el `<select>` de
`/config/system` (etiquetados +9 y +10/+11) pero **`posixTz()` no los mapea**: caen
al `return "UTC0"` y la hora queda en UTC. Son opciones de UI sin soporte real.

### 5.1 `nowEpoch()`: época **local**, no UTC

`include/core/Time.hpp` define el reloj que usa todo el firmware:

| Función | Comportamiento |
|---------|----------------|
| `nowEpoch()` | Si `time(nullptr) <= 1000000000` (antes de 2001-09-09 = sin NTP) → devuelve **`millis()/1000`** (uptime). Si hay reloj → devuelve `t + timezoneOffsetSeconds()` |
| `timezoneOffsetSeconds()` | `t - mktime(gmtime(t))`; devuelve `0` si todavía no hay NTP |

Es decir, `nowEpoch()` devuelve la **hora local con el offset ya sumado**, no un
timestamp UTC. ⚠️ El comentario de `Measurement.hpp:37` dice
`uint32_t timestamp = 0; // epoch seconds UTC`, pero el valor real es
`UTC + offset`. Con zona -3, un `timestamp` de 1760000000 corresponde a las
22:13 UTC del día anterior en términos UTC. Cualquier integración que asuma UTC
debe restar el offset.

Este mismo reloj alimenta: el histórico (`ts` en `/api/v1/history`), el `timestamp`
del payload MQTT/webhook, la entrada `clock` de `/api/v1/sensors` y la retención por
tiempo (`retentionDays * 86400`).

⚠️ Los **eventos y alarmas** son la excepción: `ts` en `/api/v1/events` y
`/api/v1/alarms` es `millis()` (uptime en ms), no época. Ver [API-REST](API-REST.md) §6.

## 6. Qué pasa sin Internet

SEMA está diseñado para funcionar **sin nube**: todo el núcleo es local.

| Función | Sin Internet (pero con red local) | Sin red local (modo AP) |
|---------|-----------------------------------|--------------------------|
| Web local (`/`, `/api/v1/*`) | ✅ Funciona | ✅ Funciona en `192.168.4.1` |
| mDNS `hostname.local` | ✅ Funciona (es LAN, no Internet) | ❌ No aplica (AP sin mDNS) |
| NTP | ❌ Falla → `nowEpoch()` = uptime (`millis()/1000`) y offset 0 | ❌ Igual |
| Histórico / almacenamiento | ✅ Funciona (microSD/LittleFS) | ✅ Funciona |
| `/api/v1/update/check` | ❌ Se cuelga hasta **8 s** y responde `latest: ""`, `update: false` | ❌ Igual |
| Publicador MQTT | ❌ `connect()` falla en cada medición (cada 10 s) | ❌ Igual |
| Publicador webhook | ❌ `POST` falla (timeout **2 s** por medición) | ❌ Igual |
| OTA (subida local) | ✅ Funciona: el `.bin` viaja por la LAN, no por Internet | ✅ Igual |
| Estación de referencia (nube) | ❌ Inalcanzable | ❌ Igual |

Consecuencia observable del NTP caído: los `ts` del histórico arrancan en valores
chicos (segundos de uptime) y crecen hasta el próximo reinicio; con NTP, saltan a
la época local. La web no marca ese cambio de escala.

## 7. Servicios y puertos expuestos

| Puerto | Protocolo | Servicio |
|--------|-----------|----------|
| 80/tcp | HTTP | `WebServer` (dashboard, páginas y API `/api/v1`) |
| 81/tcp | WebSocket | `WebSocketsServer` (`ws://<ip>:81/`) |
| 5353/udp | mDNS | `ESPmDNS` (solo con WiFi y `mdns = true`) |
| 1883/tcp | MQTT | **Saliente** hacia el broker (cliente, no servidor) |

No hay servidor MQTT, ni FTP, ni Telnet, ni SNMP, ni HTTPS.

## 8. Configuración relacionada

| Clave (JSON) | Default | Efecto |
|--------------|---------|--------|
| `network.mode` | `"STA"` | `STA` o `AP` (validado) |
| `network.ssid` / `password` | `""` / `""` | Credenciales STA; sin SSID cae a AP |
| `network.hostname` | `"sema-001"` | mDNS y SSID del AP |
| `network.mdns` | `true` | Habilita mDNS |
| `network.ip` / `gateway` / `subnet` / `dns` | `""` | IP estática STA (requiere las cuatro) |
| `system.timezone` | `"America/Argentina/Buenos_Aires"` | Mapeada a POSIX TZ |
| `system.ntp_server` | `"pool.ntp.org"` | Primer servidor NTP |
| `ethernet.enabled` | `false` | Habilita el Ethernet del build |
| `ethernet.mdc` / `mdio` / `cs` | 23 / 18 / 5 | Pines de gestión (LAN8720A) o CS (W5500) |

Los endpoints para cambiarlas son `POST /api/v1/config/network`,
`POST /api/v1/config/system`, `POST /api/v1/config/buses` y `PUT /api/v1/config`
(ver [API-REST](API-REST.md) §7). Recordá que
**`POST /api/v1/config/network` reinicia el equipo** y que
**`POST /api/v1/config/buses` no aplica los buses en caliente**.

## 9. Pendientes y limitaciones

| Tema | Estado |
|------|--------|
| Modo WiFi apagado (radio off) | ❌ No implementado |
| Contraseña / seguridad del AP de emergencia | ❌ AP abierto |
| Portal cautivo en modo AP | ❌ No implementado |
| Hostname por DHCP (`WiFi.setHostname`) | ❌ No implementado |
| IP estática en Ethernet | ❌ No implementado (solo DHCP) |
| Failover / prioridad WiFi ↔ Ethernet | ❌ No implementado |
| mDNS sobre Ethernet | ❌ No implementado (solo WiFi) |
| Servicio mDNS propio (`_sema._tcp`) | ❌ No implementado |
| HTTPS local / certificado | ❌ No implementado |
| IPv6 | ❌ No configurado |
| Zonas horarias fuera de la tabla de `posixTz()` | ⚠️ Caen a `UTC0` (incluye opciones del `<select>`) |

---

## Ver también

- [API-REST](API-REST.md) · [Comunicaciones-remotas](Comunicaciones-remotas.md) · [MQTT-y-WebSocket](MQTT-y-WebSocket.md)
- [Configuracion](Configuracion.md) · [Referencia-configuracion](Referencia-configuracion.md) · [Variables-modificables](Variables-modificables.md)
- [Hardware-y-Conexiones](Hardware-y-Conexiones.md) · [Guia-de-pines](Guia-de-pines.md) · [Buses-y-perifericos](Buses-y-perifericos.md)
- [Seguridad](Seguridad.md) · [Solucion-de-problemas](Solucion-de-problemas.md) · [Identidad-y-estados](Identidad-y-estados.md)
