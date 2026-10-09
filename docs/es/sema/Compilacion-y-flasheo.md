---
tags:
  - sema
  - desarrollo
---

# Compilación y flasheo

> **Tipo:** Guía | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Cómo compilar, flashear, monitorear y actualizar SEMA, con los entornos y las
particiones reales definidos en `platformio.ini`.

## 1. Requisitos

| Elemento | Detalle |
|----------|---------|
| Framework de build | **PlatformIO** (CLI o extensión de VS Code) |
| Plataforma | `espressif32` + framework `arduino` |
| Sistema de archivos | LittleFS (`board_build.filesystem = littlefs`) |
| Hardware de programación | USB con driver CP2102/CH340 según la placa |
| Toolchain | La descarga PlatformIO automáticamente en la primera compilación |

## 2. Entornos disponibles

| Entorno | Board | Chip / flash | Particiones | Notas |
|---------|-------|--------------|-------------|-------|
| `esp32doit-devkit-v1` (**default**) | DOIT DevKit v1 | ESP32-WROOM, 4 MB | `partitions_4mb.csv` | `BOARD_ESP32_WROOM`; Ethernet nativa LAN8720A (RMII) |
| `esp32-s3-devkitc-1` | ESP32-S3-DevKitC-1 | ESP32-S3, 8 MB | `partitions_8mb.csv` | `BOARD_ESP32_S3`; Ethernet por W5500 (SPI) |
| `esp32-wroom-32u` | `esp32dev` | ESP32-WROOM-32U, 16 MB | `partitions_16mb.csv` | `BOARD_ESP32_WROOM32U`; PCB futura, **pines fijos** |
| `demo` | DOIT DevKit v1 | ESP32, 4 MB | `partitions_4mb.csv` | Hereda del default + `-D SEMA_DEMO=1` (valores ficticios) |

`build_flags` comunes a los tres primeros:

```text
-D BOARD_ESP32_*
-D SEMA_USE_ETHERNET=1
-D SEMA_USE_LORA=1
-D SEMA_USE_MODBUS=1
-D SEMA_MODBUS_ISOLATED=1
-D SEMA_USE_CAN=1
-D SEMA_USE_ZIGBEE=1
-D SEMA_PINS_FROM_FILE=0     ;  =1 en esp32-wroom-32u (pines fijos de PCB)
```

> `SEMA_USE_SHIFT` está en **0** en todos los entornos: el driver de 74HC595/74HC165
> existe pero no se habilita por defecto. Ver
> [Expansores de entrada/salida](Expansores-de-entrada-salida.md).

## 3. Dependencias (`lib_deps`)

| Librería | Uso |
|----------|-----|
| `bblanchon/ArduinoJson@^6.21.5` | JSON de config, API y publicadores |
| `adafruit/Adafruit BME280 Library@^2.2.2` | BME280 |
| `adafruit/Adafruit Unified Sensor@^1.1.14` | Base de los drivers Adafruit |
| `adafruit/Adafruit BusIO@^1.14.5` | Abstracción I²C/SPI de Adafruit |
| `adafruit/Adafruit MCP23017 Arduino Library@^2.3.0` | Expansor I²C MCP23017 |
| `adafruit/Adafruit SHT4x Library@^1.0.0` | SHT40 |
| `adafruit/Adafruit ADS1X15@^2.0.0` | ADS1115 |
| `paulstoffregen/OneWire@^2.3.7` | Bus 1-Wire |
| `milesburton/DallasTemperature@^3.9.1` | DS18B20 |
| `knolleary/PubSubClient@^2.8` | MQTT |
| `links2004/WebSockets@^2.4.1` | WebSocket (puerto 81) |
| `adafruit/Adafruit AHTx0@^2.0.0` | AHT20 |
| `adafruit/Adafruit SHT31 Library@^2.2.0` | SHT31 |
| `adafruit/Adafruit BMP280 Library@^2.6.7` | BMP280 |
| `adafruit/Adafruit VEML6075 Library@^2.0.0` | UV (VEML6075) |
| `adafruit/Adafruit SCD30@^1.0.6` | CO₂ NDIR (SCD30) |
| `adafruit/Adafruit SGP30 Sensor@^2.0.0` | eCO₂/TVOC (SGP30) |
| `sparkfun/SparkFun AS3935 Lightning Detector Arduino Library@^1.4.9` | Detección de rayos |
| `4-20ma/ModbusMaster@^2.0.1` | Modbus RTU maestro |
| `jgromes/RadioLib@^6.0.0` | LoRa (SX1262 y compatibles) |
| `adafruit/Adafruit PM25 AQI Sensor@^1.0.6` | PMS5003 |

No se usa ninguna librería para el dashboard: el HTML/CSS/JS va embebido en el
firmware (`PROGMEM`) y Gridstack se sirve comprimido desde flash.

## 4. Compilar

```bash
# entorno por defecto
pio run -e esp32doit-devkit-v1

# otro entorno
pio run -e esp32-s3-devkitc-1
```

Salida esperada (termina en `SUCCESS`) y binario en
`.pio/build/<entorno>/firmware.bin`. Uso de flash/RAM de referencia: ver
[Rendimiento y memoria](Rendimiento-y-memoria.md) y
[Estadísticas y métricas](Estadisticas-y-metricas.md).

## 5. Flashear por USB

```bash
# compilar y subir
pio run -e esp32doit-devkit-v1 -t upload

# puerto explícito
pio run -e esp32doit-devkit-v1 -t upload --upload-port COM3

# borrar toda la flash (¡pierde configuración NVS y LittleFS!)
pio run -e esp32doit-devkit-v1 -t erase
```

## 6. Monitor serial

```bash
pio device monitor -b 115200
```

Muestra, en orden: versión y `hw`, schema de configuración, carga de config (o
defaults), montaje de LittleFS (y microSD si está habilitada), inicialización de
buses/sensores, conexión de red y arranque del servidor web.

## 7. Actualizar por OTA

```bash
curl -X POST http://<ip>/api/v1/ota \
  -H "X-API-Key: <security.api_key>" \
  -H "X-SHA256: <sha256-del-binario>" \
  -F "firmware=@.pio/build/esp32doit-devkit-v1/firmware.bin"
```

- La cabecera `X-SHA256` es **opcional**: si se envía, el firmware verifica la
  integridad de la partición escrita (`otaPartitionSha256()`).
- El binario debe corresponder a la **placa y flash** del entorno compilado.
- Ante fallo, las particiones A/B permiten volver a la versión anterior.

Ver [OTA y actualización](OTA-y-Actualizacion.md).

## 8. Tablas de particiones

| CSV | Flash | `nvs` | `otadata` | `app0` | `app1` | `spiffs` (LittleFS) |
|-----|:-----:|------:|----------:|-------:|-------:|--------------------:|
| `partitions_4mb.csv` | 4 MB | 0x5000 | 0x2000 | 0x1C0000 | 0x1C0000 | 0x70000 |
| `partitions_8mb.csv` | 8 MB | 0x5000 | 0x2000 | 0x300000 | 0x300000 | 0x1F0000 |
| `partitions_16mb.csv` | 16 MB | 0x5000 | 0x2000 | 0x300000 | 0x300000 | 0x9F0000 |

`app0`/`app1` habilitan **OTA con rollback**; `spiffs` aloja LittleFS (eventos; el
histórico va a microSD).

## 9. Verificación después de flashear

1. Monitor serial: arranque sin errores y versión correcta.
2. `GET /api/v1/system` → `firmware` y `config_schema` esperados.
3. `GET /api/v1/health` → `HEALTHY` y sensores online.
4. `GET /api/v1/diagnostics` → `reset_reason`, heap, histórico y dispositivos I²C.
5. Dashboard en `http://<ip>/` (o `http://sema-001.local/`).

## 10. Flujo obligatorio de release

Definido en `docs/REGLAS-DE-TRABAJO.md` del repo de código:

```text
1. pio run -e esp32doit-devkit-v1                 # SUCCESS
2. bump en include/core/Version.hpp (SEMA_FW_VERSION)
3. CHANGELOG.md (Added/Changed/Fixed/Removed)
4. docs/MEJORAS.md + docs/IMPLEMENTACION.md       # cabecera de versión
5. manifest: (Get-FileHash firmware.bin).Hash     # SHA-256 real
6. git add -A && git commit                        # Conventional Commits
7. git tag -a vX.Y.Z -m "..."                      # tag con "v" minúscula
8. git push origin main --tags
9. gh release create vX.Y.Z --notes-file <tmp> firmware.bin
10. Actualizar este repo Docs (SEMA) y compilar mkdocs
```

Los cambios **solo de documentación** no generan release (ver
[Versionado](../inicio/Versionado.md)).

## 11. Problemas comunes

| Síntoma | Causa | Solución |
|---------|-------|----------|
| `pio: command not found` | PlatformIO no instalado | Instalar PlatformIO Core o la extensión de VS Code |
| No aparece el puerto COM | Falta el driver USB-serie | Instalar CP2102 o CH340 |
| Error de compilación tras cambiar de placa | Entorno equivocado | Usar el entorno correspondiente a la placa/flash |
| El binario no arranca | Imagen de otra placa o partición | Recompilar para ese entorno; revisar el CSV de particiones |
| Cambios en la web no se ven | Caché de assets (1 h) | Recargar forzado (Ctrl+F5) |

---

## Ver también

- [Instalación y mantenimiento](Instalacion-y-mantenimiento.md) · [OTA y actualización](OTA-y-Actualizacion.md)
- [Compatibilidad](Compatibilidad.md) · [Guía de desarrollo](Guia-de-desarrollo.md) · [Rendimiento y memoria](Rendimiento-y-memoria.md)
