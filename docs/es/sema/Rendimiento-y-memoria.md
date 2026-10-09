---
tags:
  - sema
  - rendimiento
  - memoria
---

# Rendimiento y memoria

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Todos los números de esta página salen de una compilación real del firmware y de la
lectura directa del código. Donde un dato no se pudo medir, se dice.

## 1. Uso real de Flash y RAM

Compilación ejecutada el **2026-10-08** con PlatformIO y el entorno por defecto:

```console
$ pio run -e esp32doit-devkit-v1
Building in release mode
Retrieving maximum program size .pio\build\esp32doit-devkit-v1\firmware.elf
Checking size .pio\build\esp32doit-devkit-v1\firmware.elf
RAM:   [==        ]  17.1% (used 56136 bytes from 327680 bytes)
Flash: [========  ]  79.1% (used 1450949 bytes from 1835008 bytes)
========================= [SUCCESS] Took 38.99 seconds =========================
```

| Métrica | Valor | Lectura |
|---------|-------|---------|
| RAM estática ocupada | **56 136 bytes** | Variables globales y buffers estáticos, calculado por el enlazador |
| RAM total considerada por la herramienta | **327 680 bytes** (320 KB) | SRAM del ESP32 clásico |
| Porcentaje de RAM reportado | **17,1 %** | El porcentaje que imprime PlatformIO no es exactamente `56136/327680` (17,13 %); puede diferir por el redondeo de la barra gráfica |
| Flash ocupada | **1 450 949 bytes** (~1,38 MB) | Tamaño del binario |
| Flash de la partición de aplicación | **1 835 008 bytes** (`0x1C0000` = 1,75 MB) | No son los 4 MB de la placa: `partitions_4mb.csv` reserva dos particiones OTA |
| Porcentaje de Flash reportado | **79,1 %** | Coincide con `1450949 / 1835008` |
| Flash libre en la partición de app | **≈ 384 059 bytes** (~375 KB) | `1835008 − 1450949` |
| Tiempo de compilación (enlace incluido, sin recompilar dependencias) | **38,99 s** | Build incremental con `.pio` ya poblado; una compilación limpia tarda más |

### 1.1 Por qué la partición de app es de 1,75 MB y no 4 MB

`partitions_4mb.csv` (entorno `esp32doit-devkit-v1`):

| Partición | Tipo | Offset | Tamaño | Bytes |
|-----------|------|--------|--------|-------|
| `nvs` | data/nvs | `0x9000` | `0x5000` | 20 480 |
| `otadata` | data/ota | `0xE000` | `0x2000` | 8 192 |
| `app0` | app/ota_0 | `0x10000` | `0x1C0000` | **1 835 008** |
| `app1` | app/ota_1 | `0x1D0000` | `0x1C0000` | 1 835 008 |
| `spiffs` | data/spiffs | `0x390000` | `0x70000` | 458 752 |

La actualización OTA redundante (A/B) es la razón de que queden ~375 KB de margen en
cada slot. Ese margen es el límite duro para agregar código en esta placa: si el
binario supera 1 835 008 bytes, no entra.

### 1.2 Otros entornos (no recompilados en esta verificación)

Los otros tres entornos de `platformio.ini` existen y se pueden compilar, pero **no
se compilaron en esta pasada** (restricción de una sola compilación). Sus
particiones son distintas:

| Entorno | Placa | Flash | Archivo de particiones |
|---------|-------|-------|------------------------|
| `esp32doit-devkit-v1` (default) | ESP32-WROOM-32 | 4 MB | `partitions_4mb.csv` |
| `esp32-s3-devkitc-1` | ESP32-S3 | 8 MB | `partitions_8mb.csv` |
| `esp32-wroom-32u` | ESP32-WROOM-32U | 16 MB | `partitions_16mb.csv` |
| `demo` | hereda de `esp32doit-devkit-v1` + `-D SEMA_DEMO=1` | 4 MB | `partitions_4mb.csv` |

⚠️ **No verificado**: uso de Flash/RAM de los entornos S3, 32U y demo. `docs/GUIA.md`
no publica cifras de tamaño, así que no hay un número documentado que citar como
alternativa.

## 2. Buffers y tamaños relevantes

### 2.1 Documentos JSON (ArduinoJson `DynamicJsonDocument`)

Se asignan **en el heap** en el momento de serializar y se liberan al salir de la
función. Los grandes son la fuente principal de picos de heap:

| Buffer | Bytes | Dónde | Para qué |
|--------|-------|-------|----------|
| Configuración (serializar / deserializar) | **16 384** | `ConfigManager.cpp:121` y `:283` | JSON completo de configuración (schema 1) |
| Histórico (respuesta web) | **16 384** | `HttpServer.cpp:2350` | Hasta `limit=3000` mediciones |
| Broadcast de mediciones (WebSocket) | **8 192** | `HttpServer.cpp:148` | Vector completo de mediciones del ciclo |
| Backup de configuración | **8 192** | `HttpServer.cpp:1387` | Config + metadatos |
| Sensores (respuesta `/api/v1/sensors`) | **8 192** | `HttpServer.cpp:2141` | Catálogo + mediciones + derivadas + reloj |
| Config PUT | **8 192** | `HttpServer.cpp:1539` | Body entrante de configuración |
| Config sensores | **8 192** | `HttpServer.cpp:1582` | Body de `/api/v1/config/sensors` |
| Config IO | **8 192** | `HttpServer.cpp:1652` | Body de `/api/v1/config/io` |
| Config buses | **8 192** | `HttpServer.cpp:1714` | Body de `/api/v1/config/buses` |
| Events / Alarms | **8 192** | `HttpServer.cpp:2369` y `:2386` | Log de eventos (máx. 100) |
| Update check | **8 192** | `HttpServer.cpp:1836` | Manifiesto remoto de firmware |
| Modo demo (`SEMA_DEMO`) | **8 192** | `HttpServer.cpp:2061` | Catálogo y mediciones ficticias |
| System (`/api/v1/system`) | **2 048** | `HttpServer.cpp:1402` | Identidad, pines reservados, reset reason |
| Dashboard layout GET | **2 048** | `HttpServer.cpp:2278` | JSON del layout Gridstack |
| Capabilities | **1 024** | `HttpServer.cpp:1989` | Lista de capacidades |
| Diagnostics | **1 024** | `HttpServer.cpp:2027` | Salud, heap, I²C, contadores |
| GPIO | **1 024** | `HttpServer.cpp:2405` | Especificaciones y valores |
| Modbus | **1 024** | `HttpServer.cpp:2470` | Registros leídos |
| Health | **768** | `HttpServer.cpp:1356` | Estado + tareas vigiladas |
| Network | **384** | `HttpServer.cpp:2004` | WiFi + Ethernet |

Documentos chicos, repetidos en los caminos calientes:

| Buffer | Bytes | Dónde |
|--------|-------|-------|
| Evento individual (EventLog) | 256 | `EventLog.cpp:80` (escribir) y `:104` (parsear) |
| Medición individual (HistoryStore) | 256 | `HistoryStore.cpp:12` (parsear), `:107` (escribir), `:258` (agregado) |
| Medición publicada (MQTT) | 256 | `MqttPublisher.cpp:46` |
| Medición publicada (webhook) | 256 | `HttpPublisher.cpp:19` |
| `/api/v1/status` | 256 | `HttpServer.cpp:1345` |
| Backup | 256 | `HttpServer.cpp:1781` |
| GpioWrite / CanWrite / LoraWrite / ZigbeeWrite | 256 | `HttpServer.cpp:2429`, `2491`, `2541`, `2614` |
| Shift GET/POST | 256 | `HttpServer.cpp:2441` y `:2457` |
| Sensors config GET | 512 | `HttpServer.cpp:1505` |
| Update check (respuesta) | 512 | `HttpServer.cpp:1825` |
| Energy | 256 | `HttpServer.cpp:2018` |

Nota metodológica: ArduinoJson 6.21.6 asigna el **pool completo** pedido en
`DynamicJsonDocument`, aunque el documento use menos. El pico de heap se produce en
las rutas que combinan buffers grandes: `GET /api/v1/history` con
`limit=3000` (16 KB) y `PUT /api/v1/config` (16 KB de deserialización + 16 KB de
serialización para guardar en NVS).

### 2.2 Buffers de pila y de driver

| Buffer | Tamaño | Dónde | Nota |
|--------|--------|-------|------|
| Buffer de mediciones por sensor | **4 `Measurement`** | `SensorManager.cpp:24` (`Measurement buffer[4]`) | En la pila de `loopTask`. Un driver puede devolver como máximo 4 mediciones por lectura |
| Payload de trama Zigbee | **128 bytes** | `ZigbeeManager.cpp:56` | `send()` rechaza `len > 110` |
| Buffer de recepción Zigbee | `rxBuf_` | `include/core/ZigbeeManager.hpp` | Se reinicia si se llena |
| Payload de escritura Zigbee (API) | **110 bytes** | `HttpServer.cpp:2620` | `uint8_t buf[110]` |
| Buffer de respuesta CAN | **8 bytes** | `CanManager` (campo `data` de `twai_message_t`) | CAN 2.0 clásico; `dlc > 8` se rechaza |
| Payload de datos LoRa | variable | `LoraManager::send(data, len)` | Acotado por `maxLen` del llamador |
| Registros Modbus | `registerCount` | `ModbusConfig` (default **4**) | Configurable; es el tamaño del vector `values_` |
| Tokens de sesión web | 24 chars | `HttpServer.cpp:126` | `%08lx%08lx` de dos `esp_random()` |
| Claves de encabezado colectadas | 2 | `HttpServer.cpp:131-132` | `X-API-Key` y `Cookie` |

### 2.3 Assets web embebidos en flash

`HttpServer` sirve dos assets comprimidos con gzip embebidos en el binario
(`include/core/web/GridstackAssets.h`, arrays `gridstack_css_gz` y
`gridstack_js_gz`), expuestos como `/gridstack.min.css` y `/gridstack-all.min.js`
con `Content-Encoding: gzip` y `Cache-Control: max-age=3600`. No se cargan en RAM
salvo al servirlos. Su tamaño no se midió en esta verificación (⚠️ no verificado).

## 3. Límites del sistema

| Límite | Valor | Fuente |
|--------|-------|--------|
| Sensores máximos | **Sin límite explícito** | `ConfigManager::validate()` no acota `sensors[]`; `SensorManager` usa un `std::vector` |
| Mediciones por sensor y por lectura | **4** | Buffer `Measurement buffer[4]` en `SensorManager::readAll()` |
| Mediciones en RAM | Todas las del último ciclo, **sin tope** | `SensorManager::measurements_` se limpia y repuebla cada 10 s |
| Entradas del histórico | **10 000** (`maxEntries_ = 10000`) | `include/core/storage/HistoryStore.hpp:46` |
| Retención del histórico | **`retentionDays` × 86 400 s**, default **30 días** | `ConfigManager.cpp:65` y `SemaCore.cpp:81` |
| Bucket de agregación | **3 600 s** (1 hora) | `SemaCore.cpp:386` |
| Rotación del histórico | Conserva la **mitad más reciente** | `HistoryStore::rotate()` |
| Eventos en RAM | **100** (`maxEntries = 100`) | `include/core/EventLog.hpp:17` |
| Rotación del log de eventos | Al llegar a **200** líneas (`maxEntries_ × 2`) conserva las últimas 100 | `EventLog.cpp:91` |
| Tareas vigiladas por `HealthMonitor` | **10** (`kMaxTasks`) | `include/core/HealthMonitor.hpp:16` |
| Tareas del `Scheduler` | Hoy **2** | `SemaCore.cpp:183` y `:188` |
| Timeout de `HttpPublisher` | **2 000 ms** | `HttpPublisher.cpp:35` |
| Timeout de socket de `MqttPublisher` | **15 000 ms** (default de PubSubClient) | `PubSubClient.h:34-36` |
| Timeout del TWDT | **10 s** con panic | `SemaCore.cpp:203` |
| Timeout de `POST /api/v1/update/check` | **8 000 ms** | `HttpServer.cpp:1833` |
| `limit` máximo de `/api/v1/history` | **3 000** | `HttpServer.cpp:2288` |
| `limit` por defecto de `/api/v1/history` | **50** | `HttpServer.cpp:2285` |
| Duración de la sesión web | **3 600 000 ms** (1 h deslizante) | `HttpServer.cpp:47` |
| Intentos de login fallidos antes del bloqueo | **5** | `HttpServer.cpp:1329` |
| Duración del bloqueo por login fallido | **60 000 ms** | `HttpServer.cpp:1331` |

## 4. Dónde se va el tiempo y la memoria

### 4.1 Costos por ciclo de adquisición (cada 10 s)

Por cada medición producida (típicamente 15-25 mediciones con el catálogo por
defecto más las 4 derivadas):

1. `history_.append(m)` → **un `SD.open(..., "a")` + `println` + `close()` por
   medición** (`HistoryStore.cpp:120-125`). No hay buffering ni escritura en lote.
   Es el costo de E/S dominante del ciclo.
2. `publishers_.publishAll(m)` → si hay webhook configurado, **un POST HTTP por
   medición** con el timeout de 2 s cada uno. Con 20 mediciones y el webhook caído,
   el ciclo puede tardar hasta 40 s: ⚠️ **excede el TWDT de 10 s** y puede provocar
   un reinicio. Es el riesgo de rendimiento más grave del diseño actual.
3. `rules_.evaluate(m)` → doble bucle reglas × mediciones, O(R×M). Con R y M
   chicos es despreciable.

`broadcastMeasurements()` se hace **una sola vez** por ciclo y solo si hay clientes
WebSocket conectados (`ws_.connectedClients() == 0` → retorno temprano,
`HttpServer.cpp:144-146`), lo que evita construir el documento de 8 KB sin
necesidad.

### 4.2 Costos de memoria

- El firmware reserva **56 136 bytes** de RAM estática. El resto de la SRAM queda
  para heap de FreeRTOS, pilas de tareas del sistema y las pilas de red.
- El **peor caso de heap** es atender una petición de `history` con 3 000 entradas
  (16 KB de documento + los objetos `Measurement` en un `std::deque`, cada uno con
  5 `String`) mientras hay un broadcast WebSocket en curso (8 KB). Con 20 mediciones
  simultáneas no es un problema; con 3 000 mediciones sí hay presión de heap y
  fragmentación.
- `HistoryStore` **no guarda nada en RAM**: `readRecent()` devuelve las mediciones
  en un `std::deque` que el llamador libera al terminar la petición.
- `EventLog` mantiene un `std::deque<Event>` de hasta 100 entradas en RAM de forma
  permanente; cada `Event` tiene 4 `String` (`source`, `correlationId`, `target`, más
  los metadatos), así que es una estructura relativamente costosa para 100
  entradas: ~100 × (4 × sizeof(String) + 12 bytes + heap de los strings).
- El **histórico no se guarda en flash interna**: `HistoryStore::append()` devuelve
  `false` inmediatamente si `sdEnabled_` es falso (`HistoryStore.cpp:98-100`). Sin
  microSD, las gráficas quedan vacías y **no se consume flash interna**, que es el
  comportamiento buscado por diseño.

### 4.3 Puntos de atención para futuras optimizaciones

| Observación | Impacto | Vía de solución |
|-------------|---------|-----------------|
| Un `open`/`close` de SD por medición | Latencia y desgaste de la tarjeta en cada ciclo | Agrupar las mediciones del ciclo y escribir en lote |
| Un POST HTTP por medición en el webhook | Hasta 2 s × N mediciones, supera el TWDT | Publicar por lote o mover a una tarea propia con cola |
| `MQTT_SOCKET_TIMEOUT` de 15 s sin ajustar | Puede superar el TWDT de 10 s y reiniciar | Llamar a `setSocketTimeout()` con un valor < 10 s |
| `HttpPublisher` con timeout de 2 s × N | Igual que arriba | Ídem |
| `history.aggregate()` recorre el archivo entero dentro del `loop()` | Bloqueo periódico (cada hora) proporcional al tamaño del archivo | Ejecutarlo en una tarea separada o por lotes |
| `GET /api/v1/history` lee el archivo completo y devuelve hasta 3 000 entradas | Pico de heap de 16 KB y bloqueo de E/S | Limitar por rango temporal y transmitir en streaming |
| Documento de configuración de 16 KB asignado en cada guardado | Pico de heap puntual | Aceptable hoy; tenerlo presente si se agregan claves |

## 5. Cómo medir en tu propia placa

```text
1. Uso estático (lo que reporta este documento):
   pio run -e esp32doit-devkit-v1        → líneas RAM: y Flash:

2. Heap libre en runtime:
   GET /api/v1/health      → campo "free_heap"     (ESP.getFreeHeap())
   GET /api/v1/diagnostics → campo "free_heap"

3. Ocupación del histórico:
   GET /api/v1/diagnostics → "history": { "entries", "max" }

4. Tareas y salud:
   GET /api/v1/health      → array "tasks" con "name" y "healthy"
   GET /api/v1/diagnostics → "tasks" (cantidad de tareas del Scheduler) y
                             "modules" (módulos registrados, hoy 0)

5. Estado de la microSD:
   GET /api/v1/system      → "sd_enabled" (config) y "history_available" (real)
```

⚠️ No hay instrumentación de tiempo de ejecución por etapa (no hay contadores de
duración ni trazas de performance). La estimación de costos de §4.1 surge del
análisis del código, no de una medición en placa.

---

## Ver también

- [Arquitectura](Arquitectura.md) · [Tareas-y-concurrencia](Tareas-y-concurrencia.md)
- [Diagnostico-y-salud](Diagnostico-y-salud.md) · [Almacenamiento-e-historico](Almacenamiento-e-historico.md)
- [Modulos-y-ciclo-de-vida](Modulos-y-ciclo-de-vida.md) · [Configuracion](Configuracion.md)
- [Energia-y-consumo](Energia-y-consumo.md) · [Solucion-de-problemas](Solucion-de-problemas.md)
