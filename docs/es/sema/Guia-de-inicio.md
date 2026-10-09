---
tags:
  - sema
  - guia
---

# Guía de inicio

> **Tipo:** Guía | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Documentación para **entender y usar SEMA sin conocer el proyecto ni el código de
antemano**. Los apartados son progresivos: se puede leer de principio a fin.

## 1. ¿Qué es SEMA?

**SEMA** (Sistema de Estación Meteorológica Autónoma) es una plataforma **modular**
para construir estaciones meteorológicas y ambientales basadas en el microcontrolador
**ESP32**.

La idea central es muy simple:

> **El hardware define las capacidades. La configuración web define cómo se utilizan.**

Conectás los sensores al ESP32 (hardware) y después, **desde una página web**, decidís
qué hace cada uno (qué mide, con qué nombre, cada cuánto, a dónde se envía) — **sin
recompilar ni tocar código**.

> **Simple por fuera, modular por dentro.** Por fuera: una estación que se configura
> desde el navegador. Por dentro: módulos independientes (sensores, alarmas,
> publicadores, almacenamiento, red, web) que el Core orquesta.

### 1.1 ¿Qué puede hacer hoy (v1.103.0)?

- Leer **17 modelos de sensor** por I²C, 1-Wire, ADC, PCNT y UART: `ADC`, `ADS1115`,
  `AHT20`, `AS3935`, `BH1750`, `BME280`, `BMP280`, `CO`, `DS18B20`, `PCNT`,
  `PMS5003`, `SCD30`, `SGP30`, `SHT31`, `SHT40`, `SOLAR` y `VEML6075`.
- **Detectar** automáticamente los dispositivos conectados al bus I²C (y los DS18B20
  presentes en el bus 1-Wire).
- Guardar **eventos y alarmas** en la flash interna (LittleFS): sobreviven a un
  reinicio. El **histórico de mediciones**, en cambio, se guarda en **microSD**
  (`storage.sd_enabled`), que es opcional.
- Publicar las mediciones por **webhook HTTP** y **MQTT**.
- Exponer una **API REST** (`/api/v1`) y un **dashboard web** con actualización en
  tiempo real por **WebSocket** (puerto 81).
- **Actualizarse por OTA** (sin cable), con particiones A/B y verificación SHA-256
  opcional.
- Calcular **magnitudes derivadas**: punto de rocío, índice de calor, sensación
  térmica, presión de vapor, humedad absoluta, VPD, QNH, altitud barométrica, AQI,
  tasa/acumulado de lluvia y dirección de viento.

!!! warning "Parcial o pendiente"
    - **Deep sleep**: `PowerManager::sleep()` y el wake-up por lluvia existen, pero
      hoy **ningún componente los invoca**; la estación no se duerme sola.
    - **Store & Forward**: no hay cola de reenvío para publicadores caídos.
    - **Servidor Central**: fuera del alcance (proyecto separado).
    - **HTTPS, RBAC por roles y OTA firmado**: pendientes.

## 2. Principios de diseño

Cinco reglas resumen la filosofía (detalle en [Arquitectura](Arquitectura.md)):

```text
SENSOR ≠ FUNCIÓN      → un mismo sensor puede medir distintas cosas
GPIO   ≠ SENSOR       → un pin no es un sensor; es un recurso asignable
BUS    ≠ SENSOR       → el bus (I²C/1-Wire) es transporte, no identidad
MODELO ≠ MAGNITUD     → dos modelos distintos pueden medir la misma magnitud
HARDWARE ≠ CONFIG     → el hardware da capacidades; la config decide el uso
```

Consecuencias prácticas:

- **El Core nunca depende de módulos opcionales**: si un sensor falla, el resto sigue
  midiendo.
- **Los sensores son intercambiables**: BME280, SHT40, SHT31, AHT20 y BMP280 miden lo
  mismo (temperatura/humedad/presión) detrás de la misma interfaz `Sensor`.
- **El catálogo de sensores sale de la configuración**, no del código
  (`SensorFactory::create()` → `src/core/sensors/SensorFactory.cpp`).

## 3. Hardware

### 3.1 Placa

- Entorno por defecto: **`esp32doit-devkit-v1`** (ESP32-WROOM, 4 MB de flash).
- Particiones **OTA redundantes**: hay dos copias del firmware (`app0`/`app1`) más
  `otadata`; si una actualización falla, se conserva la anterior
  (`partitions_4mb.csv`).
- Otros entornos: `esp32-s3-devkitc-1` (8 MB, Ethernet W5500 por SPI),
  `esp32-wroom-32u` (16 MB, PCB futura, pines **fijos**) y `demo`
  (`-D SEMA_DEMO=1`, valores ficticios para probar la web sin sensores).

### 3.2 Pines por defecto

| Función | Pin | Nota |
|---------|-----|------|
| I²C SDA | GPIO 21 | Bus I²C compartido (`i2c_sda`, `SEMA_PIN_I2C_SDA`) |
| I²C SCL | GPIO 22 | Bus I²C compartido (`i2c_scl`, `SEMA_PIN_I2C_SCL`) |
| 1-Wire | GPIO 4 | DS18B20, con pull-up de 4,7 kΩ (`SEMA_PIN_ONEWIRE`) |
| ADC batería | GPIO 34 | Divisor resistivo 11:1 para 12 V (`SEMA_PIN_BATTERY_ADC`) |
| SPI (WROOM) | SCK 14 · MISO 12 · MOSI 15 | Bus de LoRa/microSD (`HwProfile.hpp`) |
| microSD CS | GPIO 4 | `storage.sd_cs`; **comparte el número con 1-Wire**, no se usan juntos |
| CAN (TX/RX) | GPIO 5 / 4 | Default de `can.tx`/`can.rx` si no se fija por archivo |
| Modbus RTU (RX/TX) | GPIO 16 / 17 | Default de `modbus.rx`/`modbus.tx` |
| LoRa (CS/RST/DIO1/BUSY) | GPIO 10 / 32 / 26 / 27 | `SEMA_CS_LORA` y pines fijos |
| Zigbee (RX/TX) | GPIO 18 / 19 | Default de `zigbee.rx`/`zigbee.tx` |
| Ethernet RMII (MDC/MDIO) | GPIO 23 / 18 | Solo con Ethernet nativo habilitado |

!!! info "Dos mecanismos de pines fijos, no confundir"
    - **`SEMA_PINS_FROM_FILE`** (`include/hw/HwProfile.hpp`, build flag del entorno):
      fija los pines de **buses y módulos** (CAN, Modbus, Zigbee, LoRa, Ethernet) e
      impide configurarlos por la web. Vale `0` en WROOM y S3, y `1` en WROOM-32U.
    - **`SEMA_FIXED_HARDWARE`** (`include/core/BoardProfile.hpp`, default `0`): al
      ponerlo en `1` se **ignora el catálogo `sensors[]`** y se usa el catálogo fijo de
      sensores definido en ese archivo (`SemaCore::applySensors()`).

    El detalle completo, con restricciones y conflictos, está en
    [Guía de pines](Guia-de-pines.md).

### 3.3 Catálogo por defecto (sin `sensors[]` configurado)

Si `sensors[]` está vacío (o `SEMA_FIXED_HARDWARE = 1`), `SemaCore` arma este catálogo
(`src/core/SemaCore.cpp` §282-298):

| ID | Modelo | Magnitudes | Pin / dirección |
|----|--------|------------|-----------------|
| `EXT` | BME280 | temperatura, humedad, presión | I²C 0x76 u 0x77 |
| `INT` | SHT40 | temperatura, humedad | I²C 0x44 |
| `SOIL` | DS18B20 | temperatura | 1-Wire GPIO 4 |
| `LUX` | BH1750 | luminosidad | I²C 0x23 |
| `AUX` | AHT20 | temperatura, humedad | I²C 0x38 |
| `BATT` | ADC | voltaje de batería | ADC GPIO 34, `scale = 3,3 × 11 / 4095 ≈ 0,008864` V/count |

### 3.4 Sensores soportados (catálogo configurable)

| `model` | Interfaz | Magnitudes (`measurement`) | Dirección / pin |
|---------|----------|----------------------------|-----------------|
| `BME280` | I²C | `temperature`, `humidity`, `pressure` | 0x76 y si falla 0x77 (el driver prueba las dos) |
| `BMP280` | I²C | `temperature`, `pressure` | 0x76 (fijo en el driver) |
| `SHT40` | I²C | `temperature`, `humidity` | 0x44 (default de la librería) |
| `SHT31` | I²C | `temperature`, `humidity` | 0x44 (fijo en el driver) |
| `AHT20` | I²C | `temperature`, `humidity` | 0x38 |
| `BH1750` | I²C | `light` (lux) | 0x23 |
| `VEML6075` | I²C | `uva`, `uvb`, `uvi` | 0x10 |
| `SCD30` | I²C | `co2`, `temperature`, `humidity` | 0x61 |
| `SGP30` | I²C | `eco2`, `tvoc` | 0x58 |
| `AS3935` | I²C | `lightning_distance` | 0x03 (default de la librería) |
| `ADS1115` | I²C | la que definas en `channel` | 0x48 |
| `DS18B20` | 1-Wire | `temperature` | `pin` (GPIO 4 por defecto); `rom` opcional |
| `ADC` | ADC | la que definas en `channel` + `unit` | `pin`; `scale`/`offset` |
| `CO` | ADC | `co` (ppm) | `pin`; `scale`/`offset` |
| `SOLAR` | ADC | `solar_radiation` (W/m²) | `pin`; `scale`/`offset` |
| `PCNT` | GPIO (PCNT) | la que definas en `channel` (p. ej. `rain`, `wind_speed`) | `pin`; `scale` |
| `PMS5003` | UART | `pm1`, `pm25`, `pm10` | `rx`/`tx` (Serial2, 9600 8N1) |

!!! warning "El campo `address` de `sensors[]` no se aplica"
    `SensorFactory::create()` no pasa `spec.address` a los drivers, así que cada sensor
    usa la dirección por defecto de su librería (tabla de arriba). Para cambiar la
    dirección de un dispositivo I²C hay que editar el driver.

## 4. Compilar y flashear

- **Requisitos**: [PlatformIO](https://platformio.org/) (CLI o IDE) y un cable USB.
- **Compilar**: `pio run -e esp32doit-devkit-v1` → `.pio/build/esp32doit-devkit-v1/firmware.bin`.
- **Flashear por USB**: `pio run -e esp32doit-devkit-v1 -t upload`.
- **Monitor serie**: `pio device monitor -b 115200`.
- **Actualizar por OTA**: `POST /api/v1/ota` con `X-API-Key` y el binario en multipart
  (opcionalmente `X-SHA256` con el hash de 64 hex).

Los pasos completos, la salida esperada y las dependencias están en
[Compilación y flasheo](Compilacion-y-flasheo.md).

## 5. Primer arranque

1. Al encender, el ESP32 monta NVS, LittleFS y el histórico, incrementa el contador
   `boots` y carga la configuración guardada; si no hay —o es inválida— usa los
   defaults de fábrica (`station.id = "SEMA-001"`, `station.name = "Estación Norte"`,
   `network.hostname = "sema-001"`, `mdns = true`, `storage.retention_days = 30`).
2. Escanea el bus I²C y registra un modelo sugerido por dirección.
3. Se conecta a la **WiFi** en modo `STA`; si no hay SSID o el modo no es `STA`, abre
   un **punto de acceso** con el hostname como SSID.
4. **mDNS** publica el nombre saneado (`sema-001` → `sema-001.local`).
5. Abrí el dashboard:

```text
http://sema-001.local/
```

Verás el estado, las mediciones en vivo (el panel se refresca cada **5 s** y los
gráficos cada **60 s**) y los controles de edición si estás autenticado.

!!! tip "Sin `sema-001.local`"
    Usá la IP que imprime el puerto serie o `GET /api/v1/network`. En modo AP, el
    endpoint informa la IP del punto de acceso.

### 5.1 Por consola (Serial 115200)

Al arrancar imprime, entre otras cosas:

```text
SEMA v1.103.0 (hw rev0, schema 1, protocol 1)
Estación: Estación Norte (SEMA-001)
Config válida: sí
Wake reason: 0
Capacidades: ADC=1 PCNT=1 DualCore=1 CAN=1
I²C 0x76 → BME280/BMP280
Sensores: 6 registrados, 4 activos
Web local: http://192.168.1.10/
mDNS: http://sema-001.local/
Módulos registrados: 0
```

## 6. Configuración

La configuración es un **JSON versionado** (`schema_version: 1`) que se guarda en NVS
y se edita desde la web (`/config/network`, `/config/security`, `/config/system`,
`/config/wind`, `/config/sensors`) o por la API. Ejemplo mínimo:

```json
{
  "schema_version": 1,
  "station": { "id": "SEMA-001", "name": "Estación Norte" },
  "network": { "mode": "STA", "ssid": "MiRed", "password": "clave", "hostname": "sema-001", "mdns": true },
  "system": { "timezone": "America/Argentina/Buenos_Aires", "log_level": "INFO" },
  "storage": { "backend": "littlefs", "retention_days": 30, "sd_enabled": false, "sd_cs": 4 },
  "security": { "api_key": "clave-web-local", "server_key": "clave-servidor", "username": "admin", "password": "clave-web" },
  "publishers": { "webhook_url": "", "mqtt_host": "", "mqtt_port": 1883, "mqtt_topic": "sema/measurement" },
  "sensors": [
    { "id": "EXT",  "model": "BME280",  "enabled": true, "sda": 21, "scl": 22 },
    { "id": "SOIL", "model": "DS18B20", "enabled": true, "pin": 4 },
    { "id": "BATT", "model": "ADC", "enabled": true, "pin": 34, "channel": "voltage", "unit": "V", "scale": 0.008864, "offset": 0.0 }
  ],
  "rules": [ { "name": "high_temp", "sensor_id": "EXT", "channel_id": "temperature", "op": "gt", "value": 40.0 } ],
  "calibrations": [ { "sensor_id": "EXT", "channel_id": "temperature", "gain": 1.0, "offset": 0.0, "has_range": true, "min": -40.0, "max": 85.0 } ],
  "energy": { "rain_pin": 0 }
}
```

| Sección | Para qué |
|---------|----------|
| `station` | Identidad de la estación (`id`, `name`) |
| `network` | WiFi (modo, credenciales, hostname, mDNS, IP estática) |
| `system` | Zona horaria, NTP, nivel de log, unidades, idioma, altitud, veleta |
| `storage` | Backend declarado, retención y microSD |
| `security` | `api_key`, `server_key`, `extra_keys`, usuario y contraseña del login web |
| `publishers` | Webhook URL y MQTT (host, puerto, topic, usuario, clave) |
| `sensors` | **Catálogo de sensores** (vacío = catálogo por defecto) |
| `rules` | Reglas de alarma (nombre, sensor, canal, operador, umbral) |
| `calibrations` | Calibración por canal (gain/offset/rango) |
| `gpio` | Pines digitales standalone (entradas/salidas) |
| `mcp23s17_cs` / `mcp23s17_pins` | Expansor GPIO SPI de 16 pines |
| `shift_registers` | Registros de desplazamiento 74HC595/74HC165 |
| `spi_expanders` | Expansores SPI de bus (MAX14830 / SC18IS602B) |
| `modbus` · `can` · `lora` · `zigbee` · `ethernet` | Buses y módulos de comunicación |
| `i2c_sda` / `i2c_scl` | Bus I²C compartido |

!!! danger "Tres avisos importantes"
    1. **`storage.backend` no elige el backend real**: se valida, pero el histórico
       siempre va a la microSD y los eventos a LittleFS.
    2. **`storage.retention_days` solo aplica con microSD activa.**
    3. **`energy.rain_pin` no duerme la estación**: solo arma el wake-up por GPIO para
       cuando algo invoque el deep sleep (hoy, nadie).

La referencia exhaustiva clave por clave está en
[Referencia de configuración](Referencia-configuracion.md).

## 7. API REST

Base: `/api/v1`. La autenticación tiene dos mecanismos:

| Mecanismo | Cómo | Cuándo |
|-----------|------|--------|
| `X-API-Key` | Cabecera con `api_key`, `server_key` o una clave de `extra_keys` | Integraciones y operaciones de riesgo |
| Sesión web | Cookie `sema_auth` obtenida en `POST /login` | Navegador |

**Si no hay ninguna clave configurada, se permite todo** (primera configuración).
Resumen de rutas (la tabla completa, con payloads, está en [API REST](API-REST.md)):

| Método | Endpoint | Descripción | Auth |
|--------|----------|-------------|:----:|
| GET | `/status` | Nombre, firmware, uptime | — |
| GET | `/health` | Salud, heap y sensores online | — |
| GET | `/system` | Identidad, versiones, board y pines reservados | — |
| GET | `/sensors` | Catálogo + mediciones + derivadas + reloj | — |
| GET | `/history` | Histórico (`limit` 1…3000, default 50; `format=csv`) | — |
| GET | `/alarms` · `/events` | Eventos de alarma / todos los eventos | — |
| GET | `/capabilities` · `/network` · `/energy` · `/diagnostics` | Capacidades, red, energía y diagnóstico | — |
| GET | `/config` | Configuración completa (incluye claves) | ✔ |
| PUT | `/config` | Aplica configuración (transaccional) | ✔ |
| POST | `/config/network` · `/config/system` · `/config/sensors` · `/config/io` · `/config/buses` | Edición por sección | ✔ |
| POST | `/security/keys` | Alta/baja de claves adicionales | ✔ |
| GET | `/backup` · POST `/backup` | Respaldo autodescriptivo / restauración | ✔ |
| GET | `/wifi/scan` · `/update/check` | Escaneo WiFi / versión publicada | ✔ |
| GET/POST | `/gpio` · `/shift` | Leer/escribir GPIO y shift registers | POST ✔ |
| GET/POST | `/wind/north` · `/wind/resistors` | Calibración de la veleta | ✔ |
| GET/POST | `/dashboard/layout` | Layout del dashboard | ✔ |
| POST | `/restart` · `/ota` | Reinicio / actualización de firmware | ✔ |
| GET/POST | `/modbus` · `/can` · `/lora` · `/zigbee` | Buses y módulos (según build flags) | POST ✔ |

### 7.1 Ejemplos

```bash
# Leer el estado
curl http://sema-001.local/api/v1/status

# Leer mediciones (incluye derivadas)
curl http://sema-001.local/api/v1/sensors

# Exportar el histórico en CSV
curl "http://sema-001.local/api/v1/history?limit=3000&format=csv" -o sema_history.csv

# Cambiar configuración (requiere X-API-Key)
curl -X PUT http://sema-001.local/api/v1/config \
     -H "Content-Type: application/json" \
     -H "X-API-Key: clave-web-local" \
     -d '{"schema_version":1,"station":{"id":"SEMA-001","name":"Nueva"}}'
```

## 8. Dashboard y WebSocket

- **Dashboard**: página servida en `/`. Consume la API y muestra estado, mediciones,
  histórico y gráficos; el layout es configurable con Gridstack y se guarda en NVS
  (`/api/v1/dashboard/layout`). Si el login está configurado y no hay sesión, el menú
  de edición queda oculto hasta entrar por `/login`.
- **WebSocket**: `ws://<host>:81`. Cada ciclo de lectura difunde:

```json
{ "type": "measurements",
  "data": [ { "sensor_id": "EXT", "channel_id": "temperature", "measurement": "temperature",
              "value": 23.4, "unit": "degC", "quality": "VALID", "sequence": 123 } ] }
```

## 9. Cómo funciona (flujo de datos)

Cada ciclo de lectura (**cada 10 s**) ejecuta esta cadena
(`SemaCore::setup()` → tarea `sensors.read`):

```text
MEDIR → VALIDAR → PROCESAR → ALMACENAR → PUBLICAR
```

1. **MEDIR** — `SensorManager::readAll()` recorre los sensores registrados y pide hasta
   4 mediciones a cada uno; los sensores no `healthy()` se saltean.
2. **VALIDAR** — se aplica la calibración del canal (`gain`/`offset`/rango) según la
   clave `sensorId:channelId`.
3. **PROCESAR** — `DerivedEngine::compute()` agrega punto de rocío, índice de calor,
   presión de vapor y humedad absoluta al conjunto de mediciones.
4. **ALMACENAR** — cada medición va a `HistoryStore::append()` (microSD, JSONL) y los
   eventos/alarmas a `EventLog` (LittleFS, JSONL).
5. **PUBLICAR** — `PublisherManager::publishAll()` envía por webhook y MQTT; el
   `RuleEngine` evalúa las reglas y publica eventos `Alarm`; el WebSocket difunde las
   mediciones a los clientes conectados.

Además corren otras dos rutinas del `Scheduler`: `core.heartbeat` cada **5 s**
(alimenta el Health Monitor) y el `Watchdog` se refresca en cada vuelta del `loop()`
con un timeout de **10 s**. Cada hora se ejecuta la retención/agregación del histórico
si la microSD está activa.

## 10. Estructura del código

```text
SEMA/
├── include/
│   ├── core/              ← interfaces y cabeceras del Core
│   │   ├── SemaCore.hpp   ← orquestador (arranque y loop)
│   │   ├── ConfigManager.hpp  ← configuración + catálogo de sensores
│   │   ├── EventBus.hpp   ← bus de eventos tipado
│   │   ├── Measurement.hpp ← modelo canónico
│   │   ├── Capability.hpp ← capacidades de la plataforma
│   │   ├── PowerManager.hpp · Watchdog.hpp · Scheduler.hpp · HealthMonitor.hpp
│   │   ├── sensors/       ← Sensor, SensorManager, SensorFactory, drivers, I2cScanner
│   │   ├── alarms/        ← RuleEngine, Rule
│   │   ├── derived/       ← DerivedEngine, DerivedCalculator
│   │   ├── events/        ← EventLog
│   │   ├── publishers/    ← HttpPublisher, MqttPublisher, PublisherManager
│   │   ├── network/       ← WiFiManager ·  storage/ ← NvsStore, HistoryStore, Storage
│   │   └── web/           ← HttpServer (REST + dashboard + WebSocket)
│   └── hw/HwProfile.hpp   ← perfil de hardware por build flags
├── src/                   ← implementaciones (espejo de include/)
│   └── main.cpp           ← 18 líneas: setup() y loop()
├── partitions_4mb.csv · partitions_8mb.csv · partitions_16mb.csv
├── platformio.ini         ← entornos, features y lib_deps
└── docs/                  ← documentación de trabajo del repo
```

| Componente | Responsabilidad |
|------------|-----------------|
| `SemaCore` | Inicializa y conecta todos los servicios; conduce el `loop()` |
| `ConfigManager` | Carga, valida, aplica y persiste la configuración (NVS, transaccional) |
| `SensorManager` | Registra sensores, lee todos y aplica calibración |
| `SensorFactory` | Crea el driver según `model` (catálogo configurable) |
| `EventBus` / `EventLog` | Publica/suscribe eventos y los persiste en LittleFS |
| `RuleEngine` | Evalúa reglas de umbral y publica eventos `Alarm` |
| `DerivedEngine` / `DerivedCalculator` | Calcula magnitudes derivadas y conversión de unidades |
| `HistoryStore` | Persiste mediciones en microSD (JSONL + agregados) |
| `PublisherManager` | Envía mediciones (webhook HTTP, MQTT) |
| `HttpServer` | REST + dashboard + WebSocket + login |
| `WiFiManager` | WiFi STA/AP, mDNS y reconexión con backoff |
| `PowerManager` | Perfiles energéticos, deep sleep y wake por lluvia |
| `Watchdog` / `HealthMonitor` | Reinicio por bloqueo y estado de salud |

## 11. Cómo extender SEMA

Resumen: el paso a paso completo, con las rutas reales y las convenciones del repo,
está en [Guía de desarrollo](Guia-de-desarrollo.md).

- **Sensor nuevo**: `include/core/sensors/MiSensor.hpp` + `src/core/sensors/MiSensor.cpp`
  implementando `Sensor`, una rama en `SensorFactory::create()` y (si hace falta) la
  librería en `platformio.ini`.
- **Publicador nuevo**: implementar `Publisher` (`id()`, `enabled()`, `publish()`) y
  registrarlo con `publishers_.registerPublisher()` en `SemaCore::setup()`, más su
  configuración en `applyPublishers()`.
- **Regla nueva**: se define por configuración en `rules[]` (operadores `gt`, `lt`,
  `ge`, `le`) y la evalúa `RuleEngine::evaluate()`.
- **Derivada nueva**: agregar la fórmula y su `emit()` en
  `src/core/derived/DerivedCalculator.cpp` (o en `DerivedEngine::compute()` si debe
  viajar con cada medición).
- **Endpoint nuevo**: declarar el handler en `HttpServer.hpp` y registrar la ruta en
  `HttpServer::begin()` con `server_.on(...)`.

## 12. Flujo de desarrollo y releases

```text
código → pio run (SUCCESS) → bump SemVer en include/core/Version.hpp → CHANGELOG
       → firmware_manifest.json (SHA-256) → commit (Conventional Commits) → tag
       → push --tags → release en GitHub (firmware.bin) → actualizar Docs
```

- **Versionado SemVer**: `MAJOR.MINOR.PATCH`; el tag lleva `v` minúscula
  (`v1.103.0`) y la constante del firmware va sin `v` (`1.103.0`).
- Un cambio solo de documentación **no** genera release.
- Las convenciones completas están en [Guía de desarrollo](Guia-de-desarrollo.md) §5 y
  en el [Versionado](../inicio/Versionado.md) transversal.

## 13. Glosario resumido

| Término | Significado |
|---------|-------------|
| **Core** | Núcleo que orquesta los módulos; no depende de opcionales |
| **HAL** | Capa de abstracción de hardware (perfiles de placa/chip) |
| **Magnitud** | Lo que se mide (temperatura, humedad, presión, luz…) |
| **Canal** | Una magnitud concreta de un sensor (p. ej. `EXT:temperature`) |
| **Quality flag** | Estado de la medición (`VALID`, `STALE`, …) |
| **Derivada** | Magnitud calculada a partir de otras (punto de rocío, VPD…) |
| **Schema version** | Versión del formato de configuración (hoy `1`) |
| **NVS / LittleFS / microSD** | Config, eventos e histórico, respectivamente |
| **mDNS** | Nombre local `.local` sin necesidad de IP |
| **OTA** | Actualización de firmware por red, con rollback |
| **Deep sleep** | Modo de bajo consumo; en SEMA solo hay primitivas, sin uso automático |

Los términos completos están en el [Glosario](Glosario.md).

---

## Ver también

- [Home](Home.md) · [Instalación y mantenimiento](Instalacion-y-mantenimiento.md) ·
  [Compilación y flasheo](Compilacion-y-flasheo.md)
- [Arquitectura](Arquitectura.md) · [Configuración](Configuracion.md) ·
  [Referencia de configuración](Referencia-configuracion.md) · [API REST](API-REST.md)
- [Guía de desarrollo](Guia-de-desarrollo.md) · [Solución de problemas](Solucion-de-problemas.md) ·
  [FAQ](FAQ.md) · [Glosario](Glosario.md)
