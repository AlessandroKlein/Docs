---
tags:
  - sema
  - soporte
  - limitaciones
---

# Limitaciones y bugs conocidos

> **Tipo:** Referencia / Soporte | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Consolidado de **limitaciones reales** de `v1.103.0`, verificadas leyendo el código
(archivo y línea indicados). No es una lista de deseos: son comportamientos que hoy
sorprenden o que directamente no funcionan como la interfaz sugiere.

> **Cómo se obtuvo**: auditoría cruzada de todo el firmware contra esta documentación
> el 2026-10-08. **No se probó en hardware**: lo marcado como "deducido del código"
> surge de la lectura, no de una ejecución.

## 1. Severidad

| Nivel | Significado |
|:-----:|-------------|
| 🔴 | Puede hacer perder datos, funciones o falsa sensación de seguridad |
| 🟠 | Función anunciada que no hace nada, o dato engañoso |
| 🟡 | Molestia / requiere conocer el detalle para no equivocarse |

---

## 2. Seguridad y autenticación

| # | Nivel | Limitación | Evidencia | Impacto / vía de solución |
|:-:|:-----:|------------|-----------|---------------------------|
| S1 | 🔴 | `webAuthed()` devuelve `true` si `security.password` está vacía, **aunque haya `api_key`** | `HttpServer.cpp:1280-1286` | Quien llegue a la IP puede leer y escribir configuración. Configurar siempre `security.password`; documentado en [Seguridad](Seguridad.md) |
| S2 | 🟠 | La UI de OTA **no envía `X-API-Key`**; con contraseña configurada la subida responde `401` aunque la pantalla diga "Flasheado. Reiniciando…" | `HttpServer.cpp` (`onOtaUpload`) | Usar `curl` con `X-API-Key`; corregir el formulario embebido |
| S3 | 🟠 | WebSocket en el puerto **81 sin autenticación** | `HttpServer.hpp` (`WebSocketsServer ws_{81}`) | Cualquiera en la red puede escuchar las mediciones. Aislar la red o no exponer el puerto |
| S4 | 🟠 | HTTP sin TLS y OTA sin firma | `HttpServer.cpp` (servidor ESP32) | Tráfico y claves en claro; usar red de confianza/VPN |
| S5 | 🟡 | `extra_keys` permite revocar claves, pero no hay roles ni rotación asistida | `ConfigManager.hpp:63` | RBAC pendiente (ver [Futuro](Futuro.md)) |
| S6 | 🟡 | `GET /api/v1/gpio` y `GET /api/v1/shift` no piden autenticación (los `POST` sí) | `HttpServer.cpp:102-105` | Solo lectura de estado de salidas; conviene proteger también |
| S7 | 🟠 | `server_key` (pensada para el Servidor Central) da **acceso de escritura a toda la API web**, porque `webAuthed()` también acepta `authorized()` | `HttpServer.cpp:1280-1286` | Tratarla como una clave más del equipo, no como una credencial de solo lectura |
| S8 | 🟡 | MQTT sin TLS y webhook HTTP sin firma (timeout 2 s) | `MqttPublisher.cpp`, `HttpPublisher.cpp` | Usar broker en red local o VPN |

## 3. Configuración que se guarda pero no se aplica

| # | Nivel | Clave / endpoint | Efecto real | Evidencia |
|:-:|:-----:|------------------|-------------|-----------|
| C1 | 🟠 | `i2c_sda` / `i2c_scl` | El bus se abre con los pines fijos del perfil (21/22): cambiarlos por web no afecta al I²C compartido | `SemaCore.cpp:112` usa `Wire.begin(SEMA_PIN_I2C_SDA, SEMA_PIN_I2C_SCL)` |
| C2 | 🟠 | `sensors[].address` (selector «ID») | Ningún driver lo lee: las direcciones están fijas en código (BMP280 0x76, SHT40/SHT31 0x44, VEML6075 0x10, SCD30 0x61, SGP30 0x58, AS3935 0x03, ADS1115 0x48) | `src/core/sensors/*.cpp` |
| C3 | 🟠 | `sensors[].bus` / `uart` / `uart_port` | Ningún driver los usa; **y `POST /api/v1/config/sensors` los descarta** (solo `PUT /api/v1/config` los conserva) | `HttpServer.cpp` (`onConfigSensors`) |
| C4 | 🟠 | `sensors[].dashboard_layout` en el backup | `serialize()` no emite `dashboard_layout` (vive en la clave NVS `layout`): restaurar un backup **no** restaura el dashboard | `ConfigManager.cpp` (`serialize`) |
| C5 | 🟠 | `POST /api/v1/config/io` | La UI envía `gpio: []` → **borra los GPIO standalone**; además usa `latch`/`expander` mientras el JSON completo usa `latch_pin`/`expander_addr` | `HttpServer.cpp` (`onConfigIo`) |
| C6 | 🟡 | `POST /api/v1/config/buses` | No aplica varios campos (baud/slave_id/register/count y parámetros de radio) que sí acepta `PUT /config` | `HttpServer.cpp` (`onConfigBuses`) |
| C7 | 🟡 | `storage.backend` | Solo se valida como cadena: **no selecciona backend** (el histórico siempre va a SD) | `ConfigManager.cpp`, `HistoryStore.cpp` |
| C8 | 🟡 | `system.log_level` | Se guarda y no se usa | `ConfigManager.hpp:37` |
| C9 | 🟠 | Migración de esquema | No existe `ConfigManager::migrate()`; `validate()` rechaza `schemaVersion != 1` y `load()` cae a defaults **sin aviso por API** (solo por serie) | `ConfigManager.cpp:48-54` |

## 4. Hardware y expansores

| # | Nivel | Limitación | Evidencia / impacto |
|:-:|:-----:|------------|---------------------|
| H1 | 🟠 | `SEMA_USE_SHIFT=0` en **todos** los entornos: `GET /api/v1/shift` responde 200 pero sin efecto (`shift_enabled=false`) y la web oculta el editor | `platformio.ini`; sin cascada: `writeByte()` manda el mismo byte a todos los 74HC595 y `readByte()` lee solo el primer 74HC165 |
| H2 | 🟠 | MCP23S17, MAX14830 y SC18IS602B son **solo configuración**: no hay driver que escriba/lea sus pines | `ConfigManager.hpp` (structs), sin `.cpp` de driver |
| H3 | 🟠 | `GpioManager::apply()` inicializa **una sola** dirección de MCP23017 y escribe todos los pines contra ese chip | `GpioManager.cpp` |
| H4 | 🟠 | `PCNT_UNIT_0`/`PCNT_CHANNEL_0` están fijos: anemómetro y pluviómetro **se pisan** | `PcntSensor.cpp` |
| H5 | 🟠 | `Serial2` se comparte entre Modbus y PMS5003 | `ModbusManager.cpp`, `Pms5003Sensor.cpp` |
| H6 | 🟡 | `I2cScanner::modelForAddress()` no reconoce AS3935 (0x03), ADS1115 (0x48-0x4B), SGP30 (0x58) ni VEML6075 (0x10); asigna 0x5C a AM2320 | `I2cScanner.cpp` |
| H7 | 🟡 | `AdcSensor::model()` devuelve `ESP32-ADC` y `PcntSensor::interface()` devuelve `GPIO`: los nombres de la API no coinciden con la configuración (`ADC`/`PCNT`) | Drivers |
| H8 | 🟡 | `PMS5003`: el canal `pm1` publica `pm10_env` y `pm10` publica `pm100_env` (nombres invertidos en el driver) | `Pms5003Sensor.cpp` |
| H9 | 🟡 | El selector de GPIO de la web ofrece 1..39 sin excluir flash (6-11; 26-32 en S3) ni consola (1/3); `reserved_pins` solo cubre SPI y RMII | `HttpServer.cpp` (páginas de config) |
| H10 | 🟡 | Dos pines fijos del perfil 32U coinciden: `SEMA_PIN_MODBUS_DERE = 0` = `SEMA_PIN_ETH_REF_CLK = 0` | `HwProfile.hpp` |
| H11 | 🟡 | `lora.rst` tiene tres defaults distintos: header 14, `ConfigManager.cpp:432` 32, web/config 14 (y macro fija 32 en el perfil 32U); en WROOM el 14 pisa `SEMA_SPI_SCK` | `HwProfile.hpp`, `ConfigManager.cpp` |
| H12 | 🟡 | Zigbee `rx/tx`: el header declara 16/17 y el parseo/web usan 18/19 | `ConfigManager.hpp` vs `ConfigManager.cpp` |
| H13 | 🟡 | No hay **PWM** implementado (`ledc`/`analogWrite` no existen en el firmware); solo la capacidad declarada | `SemaCore.cpp:71`, ver [Actuadores y salidas](Actuadores-y-Salidas.md) |
| H14 | 🟡 | `Capability::Ethernet`, `Psram` e `Ieee802154` nunca se declaran; `Dac` y `Bluetooth` se declaran `true` sin driver | `SemaCore.cpp` (tabla de capacidades) |
| H15 | 🟡 | LoRa `cs` con dos defaults: 5 (`LoraConfig`) y 10 (perfil de hardware/`buses`) | `ConfigManager.hpp`, `HwProfile.hpp` |
| H16 | 🟠 | `-D SEMA_USE_ETHERNET=0` **rompería la compilación**: `HttpServer::onNetwork()` usa `core_->ethernet()` sin guarda de flag | `HttpServer.cpp` |
| H17 | 🟡 | Zonas horarias del `<select>` que `posixTz()` no mapea (p. ej. Asia/Tokyo, Australia/Sydney) caen silenciosamente a `UTC0` | `HttpServer.cpp` / `Time.hpp` |
| H18 | 🟡 | Zigbee: el "modo standalone" y el monitoreo IPC del coprocesador (README §426/§427: `IPC_TIMEOUT`, `COPROCESSOR_OFFLINE`, reset) **no están implementados**; solo `SYS_RESET` + `AF_DATA_REQUEST`/`AF_INCOMING_MSG` | `ZigbeeManager.cpp` |
| H19 | 🟠 | Modbus: `values_` **no se limpia** al fallar una lectura (quedan datos viejos con `result != 0`) y no hay tarea de polling (solo lee cuando la API lo pide). CAN: `ACCEPT_ALL` sin filtros ni recuperación de bus-off; una velocidad no listada cae a 500 kbps en silencio | `ModbusManager.cpp`, `CanManager.cpp` |
| H20 | 🔴 | **Conflicto de pines por defecto**: `SEMA_PIN_ONEWIRE = 4` y el default de `storage.sd_cs_pin = 4`. Usar 1-Wire y microSD a la vez (configuración de fábrica) colisiona | `HwProfile.hpp`, `ConfigManager.hpp:55` |
| H21 | 🟠 | Varios drivers usan un objeto de librería `static`: **no se pueden usar dos sensores del mismo modelo** (BME280, SHT40, SHT31, BMP280, ADS1115, SCD30, SGP30, AS3935, PMS5003). `PcntSensor` fija `PCNT_UNIT_0`/`CHANNEL_0`: un solo contador de pulsos por estación (lluvia **o** viento) | `src/core/sensors/*.cpp` |
| H22 | 🟡 | De los 8 valores de `Quality` definidos, solo se emiten `VALID`, `COMMUNICATION_ERROR` (BMP280, SHT31, BH1750) y `SENSOR_DISCONNECTED` (BH1750, DS18B20): los otros 5 no se generan nunca | Drivers |
| H23 | 🟡 | Los ADR 0009 y 0011 describen **Safe Mode** y rollback automático del bootloader que no están implementados en el código | `ConfigManager.cpp`, sin `sdkconfig` |

## 5. Energía, tareas y módulos

| # | Nivel | Limitación | Evidencia / impacto |
|:-:|:-----:|------------|---------------------|
| E1 | 🟠 | `PowerManager::sleep()` y `EnergyProfile::setProfile()` **no tienen llamadores**: el equipo nunca duerme por firmware y el perfil queda en `normal` | `PowerManager.cpp`; el deep sleep solo ocurre si algo externo lo invoca |
| E2 | 🟡 | `HistoryStore::prune()` y `setMaxEntries()` nunca se llaman (la retención por tiempo sí se configura, pero la poda depende de la ruta que la invoque) | `SemaCore.cpp` |
| E3 | 🟠 | `ModuleRegistry::count()` es siempre **0**: ninguna clase deriva de `Module` (no hay módulos reales) | `ModuleRegistry.cpp`; coincide con [Mejoras y roadmap](Mejoras-y-roadmap.md) 🔄 |
| E4 | 🟡 | El watchdog es plano (TWDT), no jerárquico por tarea | `Watchdog.cpp` |
| E5 | 🟡 | Los perfiles energéticos no cambian el comportamiento y **no hay endpoint para cambiarlos** (`GET /api/v1/energy` es de solo lectura) | `PowerManager.cpp`, `HttpServer.cpp` |

## 6. Almacenamiento y datos

| # | Nivel | Limitación | Evidencia / impacto |
|:-:|:-----:|------------|---------------------|
| D1 | 🔴 | El histórico escribe **solo en microSD**; sin SD, `append()` devuelve `false` y no hay histórico ni gráficas | `HistoryStore.cpp` (`<SD.h>`); el comentario de `HistoryStore.hpp` dice "backend LittleFS" y **contradice** al `.cpp` |
| D2 | 🟡 | El `EventLog` se evalúa contra la cola RAM (100 entradas) y su `ts` es `millis()`, no época | `EventLog.cpp` |
| D3 | 🟡 | La página `/events` formatea `ts` como epoch → en un equipo real sin NTP muestra fechas de 1970 | `HttpServer.cpp` (JS embebido) |
| D4 | 🟡 | El backup no incluye el layout del dashboard y `backup.timestamp` es `millis()/1000` (uptime), no fecha | `HttpServer.cpp` (`onBackup`), `ConfigManager.cpp` |
| D5 | 🟠 | Las reglas **no accionan salidas**: el `RuleEngine` solo publica el evento; la automatización es externa | `RuleEngine.cpp` |
| D6 | 🟠 | `station_id` viaja **siempre vacío**: `Measurement::stationId` no se asigna en ningún punto del firmware (se publica así por MQTT/webhook) | `Measurement.hpp`, `SemaCore.cpp` |
| D7 | 🟡 | `nowEpoch()` devuelve hora **local** (UTC + offset), aunque el comentario de `Measurement.hpp` diga "epoch seconds UTC" | `Time.hpp` |
| D8 | 🟡 | El dashboard **no consume el WebSocket** (no hay `new WebSocket` en el firmware): refresca por polling cada 5 s; el WS del puerto 81 queda como interfaz para clientes externos | `HttpServer.cpp` |
| D9 | 🟠 | Las reglas se evalúan **por lote cada 10 s y sin detección de flanco**: mientras la condición siga siendo verdadera se emite una alarma nueva en cada ciclo (sin histéresis ni cooldown) | `RuleEngine.cpp`, `SemaCore.cpp` |

## 7. OTA y actualización

| # | Nivel | Limitación | Evidencia / impacto |
|:-:|:-----:|------------|---------------------|
| O1 | 🔴 | `X-SHA256` se compara contra el SHA-256 de la **partición completa rellenada**, no contra el `sha256` del `.bin` del manifiesto: usar el valor del manifiesto devuelve `400` | `HttpServer.cpp` (`otaPartitionSha256`, `onOtaUpload`) |
| O2 | 🔴 | Un hash incorrecto **no revierte** el flasheo ya aplicado (`Update.end(true)` marcó el slot) | `HttpServer.cpp` |
| O3 | 🟠 | El rollback automático del bootloader no está configurado de forma verificable (no hay `sdkconfig` en el repo ni llamada a `esp_ota_mark_app_valid_cancel_rollback()`) | Proyecto PlatformIO |
| O4 | 🟡 | `GET /api/v1/update/check` compara versiones con `!=` de cadenas: un *downgrade* figura como `update: true`; además usa `setInsecure()` | `HttpServer.cpp` (`onUpdateCheck`) |
| O5 | 🟡 | `firmware_manifest.json` no tiene entrada para la placa de 16 MB (`esp32-wroom32u-16mb`, el `board_id` que reporta `GET /api/v1/system`) | `firmware_manifest.json` |

## 8. Calidad y pruebas

| # | Nivel | Limitación | Evidencia / impacto |
|:-:|:-----:|------------|---------------------|
| Q1 | 🟠 | **No hay tests automatizados**: `test/` solo contiene un README; sin entorno `native` ni CI en el repo | `test/README`; ver [Pruebas y validación](Pruebas-y-validacion.md) |
| Q2 | 🟡 | La validación depende de `pio run` + revisión manual de logs y endpoints | Idem |

## 9. Documentación del repositorio de código desactualizada

Estos archivos de `SEMA/docs/` contradicen al código en `v1.103.0` (se listan para no
seguir sus datos a ciegas; este apartado del repo Docs es la versión corregida):

| Archivo | Contradicción |
|---------|---------------|
| `docs/PINES-POR-BOARD.md` | §3 da SD SPI 4/23/19/18 (el bus real es CS 4 + `SEMA_SPI_*`: 15/12/14 en WROOM) y LoRa RST 14; §4 afirma que los 16 pines del MCP23S17 sirven de CS para SD/LoRa (falso) |
| `docs/ETHERNET-Y-BUILDFLAGS.md` | §2.2 muestra `SEMA_SPI_SCK 18` en la rama no-S3 (el real es 14) |
| `docs/VERSIONADO.md` | Menciona `ConfigManager::migrate()` y schema 2 (no existen); usa nombres `GH_*` y describe `update_channel` inexistente |
| `docs/CONTINUACION.md` | El checklist §4 sigue pidiendo "52 releases" y `version: 0.52.0` (real: 159 releases, 1.103.0) |
| `docs/FUTURO.md` | Da por pendientes ítems ya implementados (rotación del EventLog, agregación, CSV, retención, tema, idiomas, zona horaria, SD) |
| `include/core/storage/HistoryStore.hpp` | El comentario de cabecera dice "backend LittleFS" y su `.cpp` usa microSD |

---

## 10. Cómo reportar

- Vulnerabilidades: seguir `SECURITY.md` del repositorio y **no** abrir un issue público
  con detalles explotables.
- Bugs funcionales: issue en <https://github.com/AlessandroKlein/SEMA/issues> con la
  salida de `GET /api/v1/diagnostics`, la versión (`GET /api/v1/system`) y el log serie.

---

## Ver también

- [Solución de problemas](Solucion-de-problemas.md) · [Pruebas y validación](Pruebas-y-validacion.md) · [Seguridad](Seguridad.md)
- [Mejoras y roadmap](Mejoras-y-roadmap.md) · [Futuro](Futuro.md) · [Referencia de configuración](Referencia-configuracion.md)
