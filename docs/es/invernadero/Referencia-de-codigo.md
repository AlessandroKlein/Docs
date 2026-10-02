# Referencia de código (v3.29.0)

> **Tipo:** Referencia técnica | **Estado:** Estable | **Fecha:** 2026-10-02

Mapa completo del código del firmware **v3.29.0**: módulos, archivos, clases,
flujo de arranque y puntos de extensión. Corresponde al tag
[`v3.29.0`](https://github.com/AlessandroKlein/Invernadero/releases/tag/v3.29.0).

## 1. Metadatos de versión

`include/core/Version.hpp`:

```cpp
#define GH_FW_VERSION            "3.29.0"   // versión del firmware
#define GH_HW_VERSION            "rev0"     // revisión de hardware
#define GH_HW_PROFILE            "ESP32-GH-V1" // perfil de PCB
#define GH_CONFIG_SCHEMA_VERSION 2          // esquema de configuración
#define GH_PROTOCOL_VERSION      1          // versión del protocolo
#define GH_PINS_LOCKED           0          // 0=público, 1=PCB fija
```

`GH_PINS_LOCKED` es la única línea que cambia entre la compilación **pública**
(0, los usuarios configuran sus pines) y la de **PCB fabricada** (1, pines
bloqueados pero sensores/actuadores configurables).

## 2. Estructura del repositorio

```text
include/           58 headers (.hpp)  — interfaces de cada clase
src/               48 fuentes  (.cpp)  — implementación
├── core/           Tipos, Version, PinMap, PinConfig, PlatformTypes,
│                   CapabilityRegistry, ModuleRegistry, EventBus, Scheduler
├── config/         ConfigManager, Defaults
├── storage/        History, StorageManager
├── hardware/       BusManager, HardwareManager, SpiManager, AdcManager,
│                   ShiftRegister595/165, Mcp23017, Mcp23s17, ModbusRtu, CanManager
├── sensors/        SensorManager, SensorRegistry, ModbusProfileRegistry,
│                   ModbusGateway + drivers (SHT31/AHT20, DS18B20, ADS1115,
│                   BH1750, SCD4x, pulsos, ultrasónico, pH, EC)
├── actuators/      ActuatorManager, ActuatorRegistry
├── control/        Climate/Irrigation/Lighting/Roof/Safety, RuleEngine,
│                   CalculatedVariables
├── network/        NetworkManager (WiFi/Ethernet), MqttManager, WeatherStation
├── api/            RestApi, WebSocketServer
├── system/         Device, OTA, Watchdog, HealthMonitor, BootCounters,
│                   Logger, Diagnostics
└── web/            WebAssets (HTML embebido)

platformio.ini      targets y librerías
partitions/         tablas de particiones (default_8MB.csv por defecto)
firmware_manifest.json  manifest OTA (versión + SHA-256)
docs/              documentación fuente (espejo de esta wiki)
```

**Tamaño:** ~5.238 líneas de `.cpp` + ~2.793 líneas de `.hpp` ≈ **8.000 líneas**.

## 3. Módulos y responsabilidades

| Módulo | Archivos clave | Responsabilidad | Líneas |
|--------|---------------|-----------------|-------:|
| `core` | `Types.hpp`, `PinConfig.hpp`, `PlatformTypes.hpp`, `EventBus`, `Scheduler` | Tipos, configuración de pines (NVS), contratos, bus de eventos | 405 |
| `config` | `ConfigManager.cpp` | Config NVS+JSON, capas, migraciones, token de API | 599 |
| `storage` | `History`, `StorageManager` | Historial circular, LittleFS/SPIFFS/SD | 165 |
| `hardware` | `BusManager`, `HardwareManager`, `SpiManager`, `AdcManager`, `ShiftRegister595/165`, `Mcp23017`, `Mcp23s17`, `ModbusRtu` | Buses, expansores, pools, RS485 | 968 |
| `sensors` | `SensorManager`, `SensorRegistry`, `ModbusGateway`, drivers | Adquisición, catálogo, gateway Modbus | 1083 |
| `actuators` | `ActuatorManager`, `ActuatorRegistry` | Salidas, jerarquía, asignación de canales | 332 |
| `control` | `Climate/Irrigation/Lighting/Roof/Safety`, `RuleEngine` | Lógica de automatización | 305 |
| `network` | `NetworkManager`, `MqttManager`, `WeatherStation` | WiFi/Ethernet, MQTT, clima externo | 314 |
| `api` | `RestApi`, `WebSocketServer` | REST + WebSocket | 681 |
| `system` | `Device`, `OtaManager`, `Watchdog`, `HealthMonitor`, `BootCounters`, `Logger` | Ciclo de vida, OTA, salud | 713 |
| `main.cpp` | — | Arranque, tareas, wiring | 350 |

## 4. Flujo de arranque (`src/main.cpp`)

```text
setup():
  1) ConfigManager.begin()         → NVS + JSON + migraciones (schema 2)
  2) StorageManager.begin()        → LittleFS/SPIFFS + History
  3) PinConfigManager.begin()      → pines desde NVS (ghpins)
     HardwareManager.begin(pins)   → buses + catálogo de nodos (ghhw)
     HardwareManager.load()        → aplica ediciones del catálogo
     [pools] mcpPool/input/spiPool/adcPool  → instanciados desde el catálogo
     ActuatorManager.begin(cfg, &shift, mcpPool, 4)
  4) SensorManager.begin(cfg, pins, &sensorRegistry) → drivers + direcciones del catálogo
     SensorRegistry.load()         → aplica ediciones del catálogo (ghsensors)
  5) Controladores (clima, riego, luz, techo, seguridad, reglas)
  6) NetworkManager + MQTT + RestApi + WebSocket + OTA + Watchdog
  7) Tareas núcleo 0: sensorTask (prio 3) + controlTask (prio 2)
loop():                            // núcleo 1
  ArduinoOTA.handle() · api.handleClient() · ws.loop() · mqtt.loop()
  scheduler.tick() · modbusGateway.tick() · EventBus drain · watchdog feed
```

### Tareas FreeRTOS

```text
xTaskCreatePinnedToCore(sensorTask,  "sensors", 8192, ..., 3, ..., 0);  // núcleo 0, prio 3
xTaskCreatePinnedToCore(controlTask, "control", 8192, ..., 2, ..., 0);  // núcleo 0, prio 2
```

`sensorTask` adquiere y publica; `controlTask` aplica seguridad + controladores;
`loop()` (núcleo 1) atiende red/API/OTA. La sincronización es por semáforo
`xSemaphoreCreateMutex()` en `SensorManager` (accesores `valueAt`).

## 5. Modularidad runtime (v3.18 → v3.29)

Todo el hardware se configura por NVS **sin recompilar**:

| Elemento | Endpoint | Persistencia | Consumidor |
|----------|----------|--------------|-----------|
| Pines + direcciones I²C | `GET/PUT /api/v1/pins` | NVS `ghpins` | `PinConfig` → buses/drivers |
| Catálogo de sensores | `GET/PUT /api/v1/sensors/catalog` | NVS `ghsensors` | `SensorRegistry` → `SensorManager` |
| Catálogo de expansores | `GET/PUT /api/v1/hardware` | NVS `ghhw` | `HardwareManager` → pools |
| Perfiles Modbus | `GET /api/v1/modbus/profiles` | runtime | `ModbusGateway` |

### Pools de expansores (v3.26 → v3.29)

```text
mcpPool[4]    Mcp23017   I²C   → canales 32..95 (device 0..3, pin 0..15) en ActuatorManager
spiPool[4]    Mcp23s17   SPI   → CS desde el nodo del catálogo
adcPool[4]    AdcManager SPI   → MCP3208, 8 canales
input         ShiftRegister165 → pines + nº chips desde PinConfig (hc165_*)
shift         ShiftRegister595 → pines desde PinConfig (hc595_*)
```

La asignación de canales vive en `ActuatorManager::writeChannel()`: canal `0..31`
→ 74HC595 (soft-PWM); `32..95` → pool MCP23017.

## 6. API REST (48 endpoints, protección por `requireAuth`)

| Grupo | Endpoints |
|-------|-----------|
| Sistema | `GET /api/v1/status`, `/device`, `/diagnostics`, `/health`, `/boot`, `/logs`, `/events`, `/capabilities`, `/modules` |
| Config | `GET/PUT /api/v1/config`, `GET /config/export`, `/config/schema`, `POST /config/import`, `/config/rollback`, `/factory-reset`, `/reset` |
| Red | `GET /network`, `POST /network/scan`, `GET /weather` |
| Sensores | `GET /sensors`, `GET/PUT /sensors/catalog`, `GET /modbus`, `/modbus/profiles`, `/modbus/gateway`, `GET/POST /rs485`, `/rs485/scan` |
| Actuadores | `GET/POST /actuators`, `GET /actuators/catalog`, `GET /zones` |
| Hardware | `GET/PUT /hardware`, `GET /buses`, `GET/PUT /pins`, `GET /detect`, `GET /storage` |
| Automatización | `GET/POST/DELETE /automation` |
| Seguridad | `POST /auth/login`, `GET /token/status`, `POST /token/rotate`, `/token/revoke` |
| OTA | `GET /ota`, `GET /firmware` |
| Web | `GET /`, `GET /pins` (formularios HTML embebidos) |

## 7. Puntos de extensión

- **Agregar un driver de sensor:** crear la clase en `sensors/`, instanciarla en
  `SensorManager` (patrón `on(id, flag)` + `addr(id, def)`) y registrarla en
  `SensorRegistry::buildFromConfig`.
- **Agregar un expansor:** sumar un pool en `main.cpp` + rama en el loop del
  catálogo + mapeo en `ActuatorManager::writeChannel`.
- **Agregar un endpoint:** registrar en `RestApi::begin` + `handleXxx()`.
- **Nuevo tipo de datos:** extender `PlatformTypes.hpp` + serialización JSON.

## 8. Compilación y release

```bash
pio run                    # build (SUCCESS esperado, ~27 s)
pio run -t upload          # flasheo
# release: bump Version.hpp → CHANGELOG → SHA-256 → firmware_manifest.json
#          → git tag vX.Y.Z → gh release create
```

Ver [Compilación y flasheo](Compilacion.md) y [OTA](OTA-y-Actualizacion.md).
