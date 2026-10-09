---
tags:
  - sema
  - lora
  - zigbee
  - modbus
  - can
---

# Comunicaciones remotas

> **Tipo:** Referencia | **Estado:** En desarrollo
> **Fecha:** 2026-10-08
> **Firmware:** v1.103.0

Buses y radios de SEMA más allá del WiFi: **Modbus RTU** (RS485), **CAN/TWAI**,
**LoRa** (SX1262) y **Zigbee** (CC2652P2), más los **publicadores** salientes
(webhook HTTP y MQTT, detallados en [MQTT-y-WebSocket](MQTT-y-WebSocket.md)) y la
relación con el **Servidor Central**.

Cada bus tiene su manager en `src/core/` con un `apply(config)` que se llama en el
arranque (`SemaCore::setup`) y —solo algunos— en caliente desde `PUT /api/v1/config`.

## 1. Buses y flags de compilación

| Módulo | Flag | Default | Clase | Ruta HTTP | Reconfigura en caliente |
|--------|------|---------|-------|-----------|--------------------------|
| Modbus RTU | `SEMA_USE_MODBUS` | `1` | `ModbusManager` | `GET /api/v1/modbus` | `PUT /api/v1/config` |
| CAN 2.0 | `SEMA_USE_CAN` | `1` | `CanManager` | `GET`/`POST /api/v1/can` | `PUT /api/v1/config` |
| LoRa | `SEMA_USE_LORA` | `1` | `LoraManager` | `GET`/`POST /api/v1/lora` | `PUT /api/v1/config` |
| Zigbee | `SEMA_USE_ZIGBEE` | `1` | `ZigbeeManager` | `GET`/`POST /api/v1/zigbee` | `PUT /api/v1/config` |
| Ethernet | `SEMA_USE_ETHERNET` | `1` | `EthernetManager` | `GET /api/v1/network` | `PUT /api/v1/config` |
| Shift registers | `SEMA_USE_SHIFT` | **`0`** | `ShiftRegisterManager` | `GET`/`POST /api/v1/shift` | — |

Los flags viven en `include/hw/HwProfile.hpp` (default `SEMA_ON = 1`) y las cuatro
placas de `platformio.ini` los compilan todos en `1`. Con un flag en `0` el módulo
**no se compila** (y, en el caso de RadioLib, la librería no se linkea) y su ruta
HTTP desaparece → `404 {"error":"not found"}`.

⚠️ **Todos los buses son síncronos y bajo demanda**: no hay una tarea del
`scheduler` que haga polling periódico. El único camino para provocar tráfico es
llamar a la ruta HTTP correspondiente (o, en Modbus/CAN/LoRa/Zigbee, usar la API).
Eso significa que **nada se registra en el histórico ni se publica por MQTT** a
partir de estos buses: son canales de E/S crudos, no fuentes de mediciones.

## 2. Modbus RTU (maestro, RS485)

`ModbusManager` (`include/core/ModbusManager.hpp`, `src/core/ModbusManager.cpp`)
sobre la librería **ModbusMaster 2.0.1** (`4-20ma/ModbusMaster@^2.0.1`).

| Parámetro (`config.modbus`) | Default | Uso real |
|-----------------------------|---------|----------|
| `enabled` | `false` | Sin esto, `apply()` no hace nada y `ready()` queda en `false` |
| `rx` / `tx` | `16` / `17` | `Serial2.begin(baud, SERIAL_8N1, rx, tx)` |
| `de_re` | `0` | GPIO de DE/RE del transceiver RS485. `0` = **sin control** (se asume auto-dirección) |
| `baud` | `9600` | Velocidad del `Serial2` |
| `slave_id` | `1` | Esclavo Modbus |
| `register` | `0` | Dirección del primer registro holding |
| `count` | `4` | Cantidad de registros a leer |
| `uart` / `uart_port` | `0` / `0` | ⚠️ **Ignorados**: el código siempre usa `Serial2` (no hay driver MAX14830) |

- **UART fijo `Serial2`** e interfaz `SERIAL_8N1`. No hay paridad ni stop bits
  configurables (8N1, como manda Modbus RTU).
- **Transceiver**: `SEMA_MODBUS_ISOLATED=1` (default) → `TD501D485H` (aislado
  galvánicamente); `=0` → `SN65HVD75DR` (sin aislar). El control DE/RE es idéntico
  en ambos; el firmware solo cambia la macro informativa
  `SEMA_MODBUS_TRANSCEIVER` (`HwProfile.hpp:76–80`).
- **Solo lectura de holding registers**: `readHoldingRegisters(register, count)`. No
  hay escritura de coils/registros, ni lectura de discretos/entradas, ni
  diagnóstico (`readCoils`, `writeSingleRegister`, etc. no se usan).
- **Un solo esclavo**: no hay barrido de IDs ni multiplexación.
- Los valores se devuelven **crudos (uint16)**, sin escala ni interpretación de
  signo/punto flotante. La conversión la hace el consumidor de la API.

Llamada real desde la API (`GET /api/v1/modbus`):

```cpp
const uint8_t result = core_->modbus().read();
// ready, result, values[]
```

| `result` | Significado |
|----------|-------------|
| `0` (`ku8MBSuccess`) | Lectura OK; `values[]` tiene `count` elementos |
| `0xFF` | El bus no está listo (`enabled: false` o fallo de `apply()`) — no se toca el puerto |
| Otros códigos | Errores de ModbusMaster (timeout, CRC, excepción del esclavo, …) |

⚠️ Cuando la lectura falla **no** se limpian `values_`: la lista conserva los
últimos valores exitosos, así que un `result != 0` puede venir acompañado de datos
viejos. Contrastá siempre `result` antes de usar `values`.

`POST /api/v1/config/buses` **no** aplica Modbus en caliente: guarda y hay que
reiniciar (`POST /api/v1/restart`). Los campos `baud`, `slave_id`, `register` y
`count` **solo** se pueden cambiar con `PUT /api/v1/config`.

## 3. CAN / TWAI

`CanManager` sobre el driver **TWAI** de ESP-IDF (`driver/twai.h`), modo **CAN 2.0**
(normal, no FD).

| Parámetro (`config.can`) | Default | Uso real |
|--------------------------|---------|----------|
| `enabled` | `false` | Habilita el driver |
| `tx` / `rx` | `5` / `4` | `TWAI_GENERAL_CONFIG_DEFAULT(tx, rx, TWAI_MODE_NORMAL)` |
| `speed` | `500000` | Bits por segundo |

Mapeo de velocidad real (`timingFor()`):

| `speed` configurado | Configuración TWAI |
|---------------------|--------------------|
| `125000` | `TWAI_TIMING_CONFIG_125KBITS()` |
| `250000` | `TWAI_TIMING_CONFIG_250KBITS()` |
| `500000` | `TWAI_TIMING_CONFIG_500KBITS()` |
| `1000000` | `TWAI_TIMING_CONFIG_1MBITS()` |
| **Cualquier otro valor** | `TWAI_TIMING_CONFIG_500KBITS()` (default silencioso) |

- Filtros: `TWAI_FILTER_CONFIG_ACCEPT_ALL()` → **acepta todas las tramas**. ❌ No hay
  filtros, ni máscaras, ni modo listen-only, ni `TWAI_MODE_NO_ACK`.
- Instalación: `twai_driver_install() + twai_start()`; `apply()` **no desinstala** el
  driver anterior ni reinicia el chip, así que reaplicar la config con distinta
  velocidad en la misma sesión no reconfigura el bus de forma limpia (requiere
  reinicio).
- `send()`: acepta `dlc <= 8`, identifica estándar o extendida (`extd`), timeout de
  **1000 ms**. Devuelve `false` si no está listo o si `dlc > 8`.
- `receive()`: **no bloqueante** (`pdMS_TO_TICKS(0)`), devuelve **una** trama por
  llamada.
- No se exponen contadores de error, estado del bus (`TWAI_STATE_*`), ni
  `twai_initiate_recovery()`: si el bus entra en *bus-off*, no hay recuperación ni
  diagnóstico por API.

## 4. LoRa (SX1262)

`LoraManager` sobre **RadioLib 6.x** (`jgromes/RadioLib@^6.0.0`), módulo
`Silicontra SX1262PATR8-GC`.

| Parámetro (`config.lora`) | Default | Uso real |
|---------------------------|---------|----------|
| `enabled` | `false` | Habilita el radio |
| `cs` | `5` (default de `LoraConfig`) | CS del `Module(cs, dio1, rst, busy)` |
| `rst` | `14` | Reset del SX1262 |
| `dio1` | `26` | Interrupción DIO1 |
| `busy` | `27` | Pin BUSY |
| `frequency` | `915.0` MHz | Frecuencia de `begin()` |
| `bandwidth` | `125.0` kHz | Ancho de banda |
| `spreading` | `7` | Factor de ensanchamiento SF7…SF12 |
| `coding_rate` | `5` | Coding rate 5…8 (4/5…4/8) |
| `tx_power` | `14` dBm | Potencia de transmisión |

⚠️ Ojo con los defaults: `LoraConfig` trae `csPin = 5`, pero `HwProfile.hpp` define
`SEMA_CS_LORA 10` y el handler `POST /api/v1/config/buses` usa **`cs = 10`** como
default. Si el bus se habilita sin fijar `lora.cs` en el JSON de config, el CS
efectivo es **5** (el de `LoraConfig`), no 10.

Cadena de inicialización real (`LoraManager.cpp:26–34`):

```cpp
g_loraModule = new Module(cs, dio1, rst, busy);
g_loraRadio  = new SX1262(g_loraModule);
state = g_loraRadio->begin(frequency, bandwidth, spreading, codingRate,
                           RADIOLIB_SX126X_SYNC_WORD_PRIVATE, txPower);
if (state == RADIOLIB_ERR_NONE) { g_loraRadio->startReceive(); ready_ = true; }
```

| Dato | Valor |
|------|-------|
| Sync word | `RADIOLIB_SX126X_SYNC_WORD_PRIVATE` (0x1424), fijo |
| Modo de recepción | `startReceive()` de **un solo paquete**: tras cada `send()` se vuelve a llamar (el radio no queda en RX continuo) |
| `send(data, len)` | Rechaza `len == 0` o `len` inválido; tras transmitir con éxito vuelve a `startReceive()` |
| `receive(data, maxLen)` | **No bloqueante**: devuelve `0` si `available() <= 0`; recorta a `maxLen` |
| Tamaño máximo por paquete en la API | **64 bytes** (buffer del handler) |
| CRC / preamble / address filtering | ❌ No configurables (defaults de RadioLib) |
| RSSI / SNR / estado del radio | ❌ No expuestos por API |
| Reintentos / ACK | ❌ No implementados |

`apply()` **borra y recrea** el objeto de radio en cada llamada (libera el `Module`
y el `SX1262` previos), así que reaplicar la config reinicia el radio de verdad.

## 5. Zigbee (co-procesador CC2652P2 por UART)

`ZigbeeManager` habla con el módulo **RF-BM-2652P2 / CC2652P2** como *Zigbee Network
Processor* (ZNP) usando el protocolo **MT (Monitor & Test)** de TI sobre UART.

| Parámetro (`config.zigbee`) | Default | Uso real |
|-----------------------------|---------|----------|
| `enabled` | `false` | Habilita el co-procesador |
| `rx` / `tx` | `16` / `17` (default de `ZigbeeConfig`) | `Serial1.begin(baud, SERIAL_8N1, rx, tx)` |
| `baud` | `115200` | Velocidad del `Serial1` |
| `uart` / `uart_port` | `0` / `0` | ⚠️ **Ignorados**: el código siempre usa `Serial1` (sin driver MAX14830) |

⚠️ Otro choque de defaults: `POST /api/v1/config/buses` usa `rx = 18`, `tx = 19`
para Zigbee, mientras que `ZigbeeConfig` (JSON de config) trae `16`/`17`. El valor
efectivo depende de qué endpoint se usó para guardar.

### 5.1 Formato de trama (MT/ZNP)

```text
SOF(0xFE) | LEN(1) | CMD0(1) | CMD1(1) | payload(LEN-2) | FCS(1)
FCS = LEN ^ CMD0 ^ CMD1 ^ payload[0] ^ … ^ payload[n]      (XOR, sin SOF)
```

### 5.2 Tramas que usa el firmware

| Trama | Dirección | Payload real |
|-------|-----------|--------------|
| `SYS_RESET` | ESP32 → CC2652P2 | `0x41 0x00`, payload `{0x01}` (soft reset), enviada en cada `apply()` |
| `AF_DATA_REQUEST` | ESP32 → CC2652P2 | `0x24 0x01`; ver campos abajo |
| `AF_INCOMING_MSG` | CC2652P2 → ESP32 | `0x44 0x81` (AREQ), parseada en `handleFrame()` |

Payload de `AF_DATA_REQUEST` (`ZigbeeManager.cpp:56–71`): `destination` (uint16 LE),
endpoint destino **0x01**, endpoint fuente **0x01**, cluster **0x0001** (uint16 LE),
`transId` (contador que incrementa por trama), `options = 0x00`, `radius = 10`,
`len`, `data[len]`. El largo de datos está acotado a **110 bytes** por `send()`.

Parseo de `AF_INCOMING_MSG` (offsets exactos del payload, que arranca después de
`CMD0`/`CMD1`):

| Offset | Campo |
|--------|-------|
| 0–1 | `groupId` |
| 2–3 | `clusterId` |
| 4–5 | `srcAddr` (uint16 LE) → `lastSrc()` |
| 6 | `srcEndpoint` |
| 7 | `dstEndpoint` |
| 8 | `wasBroadcast` |
| 9 | `linkQuality` |
| 10 | `securityUse` |
| 11–14 | `timestamp` |
| 15 | `transSeq` |
| 16 | `len` |
| 17… | `data` |

Solo se guarda **el último mensaje** (`hasMessage_`): si llegan dos antes de que la
API lea, el primero se pierde. Buffers: `rxBuf_[256]` para el stream, `lastMsg_[128]`
para el mensaje.

### 5.3 Modo standalone

El *standalone* que menciona la especificación (README §"Zigbee Network") **no está
implementado**: el firmware solo hace `SYS_RESET` y `AF_DATA_REQUEST`/`AF_INCOMING_MSG`.
No hay comandos de red (formación de red, *permit join*, *network steering*,
`ZDO_STARTUP_FROM_APP`, `ZB_WRITE_CONFIGURATION` de canal/PAN ID), ni coordinador
propio, ni enrolamiento de dispositivos. En la práctica el CC2652P2 debe estar ya
**flasheado y asociado a una red existente** antes de conectar el ESP32.

⚠️ Tampoco existe el **monitoreo IPC del coprocesador** que pide la especificación
(README §427: `IPC_TIMEOUT`, `COPROCESSOR_OFFLINE`, `COPROCESSOR_RESET`): no hay
heartbeat, ni ACK, ni detección de módulo ausente. `ready_` solo refleja que
`Serial1.begin()` se llamó.

## 6. Publicadores salientes

| Publicador | Id | Configuración | Destino |
|-----------|----|---------------|---------|
| Webhook HTTP | `"webhook"` | `publishers.webhook_url` | `POST` JSON a una URL |
| MQTT | `"mqtt"` | `publishers.mqtt_host/port/topic/user/pass` | Broker MQTT (QoS 0, retain false) |

Ambos consumen el **Modelo Canónico de Medición** cada **10 s** (tarea
`sensors.read`) y esta es la única vía **saliente** hacia servicios externos. Los
detalles de payload, timeout y limitaciones están en
[MQTT-y-WebSocket](MQTT-y-WebSocket.md).

❌ **No hay publicadores en la nube implementados** (ThingSpeak, Windy, etc.): la
interfaz `Publisher` (`include/core/publishers/Publisher.hpp`) está pensada para
agregarlos, pero en v1.103.0 solo se registran `webhook` y `mqtt`.

## 7. Relación con el "Servidor Central"

El Servidor Central **está fuera del alcance de este firmware**: no existe ningún
cliente HTTP/MQTT que se conecte a él, ni cola de mensajes, ni reintentos, ni
almacenamiento *store-and-forward*. Lo único que hay en el código es:

| Elemento | Dónde | Qué hace realmente |
|----------|-------|--------------------|
| `security.server_key` | `ConfigManager.hpp:60` | Clave que el Servidor Central usaría para autenticarse **contra** SEMA. En `authorized()` es una credencial válida como `X-API-Key` |
| `SEMA_PROTOCOL_VERSION = 1` | `include/core/Version.hpp` | Constante de versión del protocolo; se informa en `GET /api/v1/system` (`protocol`) y en el arranque por serial |
| `security.api_key` | idem | Clave de la web local |
| `security.extra_keys` | idem | JSON `{"nombre":"clave"}` de claves adicionales revocables |

⚠️ **`server_key` no está restringida**: como `webAuthed()` = `sessionAuthorized() ||
authorized()`, presentar `server_key` en `X-API-Key` habilita **todas** las rutas de
escritura (`PUT /api/v1/config`, `POST /api/v1/security/keys`, `POST
/api/v1/restart`, …). El comentario del código dice "config de riesgo, vía API":
hoy la clave del Servidor Central tiene los mismos permisos que la de la web.

❌ **No implementado:** registro/alta contra el Servidor Central, *provisioning*,
telemetría agregada, comandos remotos, actualización remota de config, ACK de
mensajes, `SEMA_PROTOCOL_VERSION` negociada en runtime (es una constante de
compilación) y cualquier forma de autenticación mutua. Es el punto de integración
futuro, no una funcionalidad actual.

## 8. Pendientes y limitaciones

| Tema | Estado |
|------|--------|
| Polling periódico de Modbus/CAN/LoRa/Zigbee | ❌ Solo bajo demanda por HTTP |
| Escritura Modbus (coils/registros) · varios esclavos | ❌ No implementado |
| Filtros y máscaras CAN · listen-only · recuperación de bus-off | ❌ No implementado (`ACCEPT_ALL`) |
| Contadores/estado de error CAN | ❌ No expuestos |
| LoRa: CRC, preamble, addressing, RSSI/SNR, ACK, RX continuo | ❌ No implementado |
| Zigbee: formación de red, *permit join*, coordinador standalone | ❌ No implementado |
| Monitoreo IPC del coprocesador (heartbeat/timeout) | ❌ No implementado |
| Driver MAX14830 / SC18IS602B | ❌ Solo se configuran el tipo y el CS: `uart`/`uart_port`/`bus` se guardan pero **ningún driver los usa** |
| Shift registers (74HC595/74HC165) | ⚠️ Beta: `SEMA_USE_SHIFT=0` en todas las placas |
| Cliente del Servidor Central | ❌ No implementado (solo `server_key` + `SEMA_PROTOCOL_VERSION`) |
| Aplicación en caliente de buses desde `/config/buses` | ⚠️ Requiere reinicio manual |

---

## Ver también

- [API-REST](API-REST.md) · [MQTT-y-WebSocket](MQTT-y-WebSocket.md) · [Conectividad-y-red](Conectividad-y-red.md)
- [Buses-y-perifericos](Buses-y-perifericos.md) · [Expansores-de-entrada-salida](Expansores-de-entrada-salida.md) · [Hardware-y-Conexiones](Hardware-y-Conexiones.md)
- [Configuracion](Configuracion.md) · [Referencia-configuracion](Referencia-configuracion.md) · [Seguridad](Seguridad.md)
- [Identidad-y-estados](Identidad-y-estados.md) · [Futuro](Futuro.md) · [Mejoras-y-roadmap](Mejoras-y-roadmap.md)
