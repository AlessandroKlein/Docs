---
tags:
  - sema
  - referencia
  - tipos
---

# Enumeraciones y tipos

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Catálogo completo de enumeraciones, structs canónicos y "enums de cadena" de SEMA.
Todo se verificó contra `include/core/**` y `include/hw/HwProfile.hpp`.

Los `enum class` de SEMA **no declaran valores explícitos**: el código numérico es
el implícito de C++ (0, 1, 2, …) en el orden de declaración, con `uint8_t` como tipo
subyacente. Donde la API expone el nombre, la tabla lo indica con la cadena exacta
que devuelve la función `*Name()` correspondiente.

## 1. `EventType` — `include/core/EventBus.hpp`

| Código | Valor | Nombre en API | Significado |
|:------:|-------|---------------|-------------|
| 0 | `Sensor` | `sensor` | Lectura de un sensor |
| 1 | `Rain` | `rain` | Evento de lluvia |
| 2 | `Lightning` | `lightning` | Rayo detectado |
| 3 | `Battery` | `battery` | Estado de batería |
| 4 | `Network` | `network` | Cambio de red |
| 5 | `Alarm` | `alarm` | Alarma disparada por una regla |
| 6 | `System` | `system` | Evento del sistema (arranque, …) |
| 7 | `Wake` | `wake` | Despertar de deep sleep |
| 8 | `Sleep` | `sleep` | Entrada en deep sleep |

`parseEventType(name)`: nombres en **minúscula**; `nullptr` o desconocido →
`EventType::System` (código 6). El enum **no** tiene centinela `Count`: `EventLog`
itera `0..Sleep` con un `TODO` en el código reconociéndolo.

## 2. `Severity` — `include/core/EventBus.hpp`

| Código | Valor | Nombre en API | Significado |
|:------:|-------|---------------|-------------|
| 0 | `Debug` | `DEBUG` | Depuración |
| 1 | `Info` | `INFO` | Informativo |
| 2 | `Notice` | `NOTICE` | Aviso |
| 3 | `Warning` | `WARNING` | Advertencia (la usa `RuleEngine` para las alarmas) |
| 4 | `Error` | `ERROR` | Error |
| 5 | `Critical` | `CRITICAL` | Crítico |

`parseSeverity(name)`: nombres en **MAYÚSCULA**; desconocido → `Severity::Info`.

## 3. `Quality` — `include/core/Measurement.hpp`

| Código | Valor | Nombre en API | Significado |
|:------:|-------|---------------|-------------|
| 0 | `Valid` | `VALID` | Lectura válida |
| 1 | `Invalid` | `INVALID` | Lectura inválida |
| 2 | `Stale` | `STALE` | Dato viejo |
| 3 | `Timeout` | `TIMEOUT` | Timeout de lectura |
| 4 | `OutOfRange` | `OUT_OF_RANGE` | Fuera del rango de calibración |
| 5 | `CalibrationError` | `CALIBRATION_ERROR` | Error de calibración |
| 6 | `CommunicationError` | `COMMUNICATION_ERROR` | Error de comunicación |
| 7 | `SensorDisconnected` | `SENSOR_DISCONNECTED` | Sensor desconectado |

`parseQuality(name)`: mayúsculas con guion bajo; desconocido → `Quality::Valid`.
Único valor que el firmware asigna por su cuenta: `OutOfRange`, desde
`applyCalibration()`.

## 4. `RuleOp` — `include/core/alarms/Rule.hpp`

| Código | Valor | Cadena en config | Significado |
|:------:|-------|------------------|-------------|
| 0 | `Gt` | `gt` | `>` |
| 1 | `Lt` | `lt` | `<` |
| 2 | `Ge` | `ge` | `>=` |
| 3 | `Le` | `le` | `<=` |

`parseRuleOp(name)`: `"lt"`, `"ge"`, `"le"`; **cualquier otro valor cae en `Gt`**
(incluido `"gt"` y cualquier operador futuro o mal escrito).

## 5. `Capability` — `include/core/Capability.hpp`

| Código | Valor | Nombre en API | Significado |
|:------:|-------|---------------|-------------|
| 0 | `WiFi` | `wifi` | Wi-Fi |
| 1 | `Bluetooth` | `bluetooth` | Bluetooth |
| 2 | `Ethernet` | `ethernet` | Ethernet |
| 3 | `Adc` | `adc` | ADC |
| 4 | `Dac` | `dac` | DAC |
| 5 | `Pcnt` | `pcnt` | Contador de pulsos |
| 6 | `LedcPwm` | `ledc_pwm` | PWM por LEDC |
| 7 | `I2c` | `i2c` | I²C |
| 8 | `Spi` | `spi` | SPI |
| 9 | `Uart` | `uart` | UART |
| 10 | `Can` | `can` | CAN/TWAI |
| 11 | `Psram` | `psram` | PSRAM |
| 12 | `RtcGpio` | `rtc_gpio` | GPIO con dominio RTC |
| 13 | `DeepSleep` | `deep_sleep` | Deep sleep |
| 14 | `DualCore` | `dual_core` | Doble núcleo |
| 15 | `Ieee802154` | `ieee802154` | 802.15.4 |
| 16 | `Count` | `unknown` | Centinela: cantidad de capacidades (no es una capacidad) |

`CapabilityManager::has()` indexa el bit `1u << código` en un `uint32_t`; el
centinela `Count` (16) entra en el rango de 32 bits.

Declaradas en runtime por `SemaCore::setup()` (13 de 16): Wi-Fi, Bluetooth, ADC,
DAC, PCNT, LEDC PWM, I²C, SPI, UART, CAN, RTC GPIO, DeepSleep y DualCore.
**No** se declaran `Ethernet`, `Psram` ni `Ieee802154`. `/api/v1/capabilities`
enumera solo las que están prendidas.

## 6. `ModuleState` — `include/core/Module.hpp`

| Código | Valor | Significado |
|:------:|-------|-------------|
| 0 | `Available` | Disponible (estado inicial de `state_`) |
| 1 | `Installed` | Instalado |
| 2 | `Configured` | Configurado |
| 3 | `Enabled` | Habilitado |
| 4 | `Running` | En ejecución |
| 5 | `Disabled` | Deshabilitado |
| 6 | `Uninstalled` | Desinstalado |
| 7 | `InstallError` | Error de instalación |
| 8 | `ConfigError` | Error de configuración |
| 9 | `RuntimeError` | Error en ejecución |
| 10 | `UpdateError` | Error de actualización |

Ciclo de vida que implementa `ModuleRegistry::enableAll()`:
`Available → install() → configure() → enable() → start()`.
`loopAll()` invoca `loop()` solo en `Enabled` (3) y `Running` (4). Los valores van en
PascalCase a propósito para no chocar con macros del core Arduino (`DISABLED`,
`ENABLED`).

## 7. `EnergyProfile` — `include/core/PowerManager.hpp`

| Código | Valor | Nombre en API | Significado |
|:------:|-------|---------------|-------------|
| 0 | `Performance` | `performance` | Máximo rendimiento |
| 1 | `Normal` | `normal` | Perfil por defecto (`profile_ = EnergyProfile::Normal`) |
| 2 | `LowPower` | `low_power` | Bajo consumo |
| 3 | `UltraLowPower` | `ultra_low_power` | Consumo ultra bajo |

⚠️ El perfil es informativo: `setProfile()` solo asigna la variable y ningún driver
lo consulta (`/api/v1/energy` lo expone como `profile`).

## 8. Motivos de despertar (`PowerManager::wakeReason()`)

`wakeReason()` **no** devuelve un enum de SEMA: devuelve el valor crudo de
`esp_sleep_get_wakeup_cause()`, cuyo tipo es `esp_sleep_source_t` de ESP-IDF
(alias `esp_sleep_wakeup_cause_t`). Valores del SDK usado por el proyecto
(`framework-arduinoespressif32`, `esp_hw_support/include/esp_sleep.h`):

| Código | Constante ESP-IDF | Significado |
|:------:|-------------------|-------------|
| 0 | `ESP_SLEEP_WAKEUP_UNDEFINED` | Reinicio que no vino de deep sleep |
| 1 | `ESP_SLEEP_WAKEUP_ALL` | No es un motivo: se usa para desactivar todas las fuentes |
| 2 | `ESP_SLEEP_WAKEUP_EXT0` | Señal externa por RTC_IO (el caso de `enableRainWakeup`) |
| 3 | `ESP_SLEEP_WAKEUP_EXT1` | Señal externa por RTC_CNTL |
| 4 | `ESP_SLEEP_WAKEUP_TIMER` | Timer (el caso de `sleep()`) |
| 5 | `ESP_SLEEP_WAKEUP_TOUCHPAD` | Touchpad |
| 6 | `ESP_SLEEP_WAKEUP_ULP` | Programa ULP |
| 7 | `ESP_SLEEP_WAKEUP_GPIO` | GPIO (solo light sleep en ESP32/S2/S3) |
| 8 | `ESP_SLEEP_WAKEUP_UART` | UART (solo light sleep) |
| 9 | `ESP_SLEEP_WAKEUP_WIFI` | Wi-Fi (solo light sleep) |
| 10 | `ESP_SLEEP_WAKEUP_COCPU` | Interrupción del COCPU |
| 11 | `ESP_SLEEP_WAKEUP_COCPU_TRAP_TRIG` | Crash del COCPU |
| 12 | `ESP_SLEEP_WAKEUP_BT` | Bluetooth (solo light sleep) |

`Serial` imprime el número sin traducir: en el arranque se ve `Wake reason: N`.
`/api/v1/energy` devuelve ese mismo número en `wake_reason`.

## 9. Structs canónicos

### 9.1 `Measurement` — `include/core/Measurement.hpp`

| Campo | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `stationId` | `String` | `""` | Identidad de la estación |
| `sensorId` | `String` | `""` | Id lógico del sensor (p. ej. `"EXT"`) |
| `channelId` | `String` | `""` | Canal lógico (un sensor puede aportar varias magnitudes) |
| `measurement` | `String` | `""` | Magnitud (`"temperature"`, `"humidity"`, …) |
| `value` | `float` | `0.0f` | Valor numérico |
| `unit` | `String` | `""` | Unidad canónica (`"degC"`, `"percent"`, `"hPa"`, …) |
| `quality` | `Quality` | `Quality::Valid` | Calidad de la medición |
| `sequence` | `uint32_t` | `0` | Secuencia monotónica por sensor |
| `timestamp` | `uint32_t` | `0` | Epoch (segundos). `Time.hpp` documenta "UTC", pero `nowEpoch()` suma el **offset local** |

### 9.2 `Event` — `include/core/EventBus.hpp`

| Campo | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `id` | `uint32_t` | `0` | `event_id` (secuencia; hoy siempre 0) |
| `timestampMs` | `uint32_t` | `0` | `millis()` monotónico |
| `source` | `String` | `""` | Id del sensor o módulo |
| `type` | `EventType` | `EventType::System` | Tipo |
| `severity` | `Severity` | `Severity::Info` | Severidad |
| `value` | `int32_t` | `0` | Payload numérico simple |
| `correlationId` | `String` | `""` | Correlación (id de regla, `"boot"`, …) |
| `target` | `String` | `""` | Destino opcional (sin uso actual) |

### 9.3 Otros structs de dominio

| Struct | Header | Campos |
|--------|--------|--------|
| `SensorInfo` | `core/sensors/SensorManager.hpp` | `String id`, `String model`, `String interface`, `bool healthy` |
| `DetectedDevice` | `core/sensors/I2cScanner.hpp` | `uint8_t address = 0`, `String model` (`""` si es desconocido) |
| `Calibration` | `core/Calibration.hpp` | `float offset = 0.0f`, `float gain = 1.0f`, `float min = 0.0f`, `float max = 0.0f`, `bool hasRange = false`, `bool enabled = false` |
| `Rule` | `core/alarms/Rule.hpp` | `String id`, `String sensorId` (`""` = cualquiera), `String channelId`, `RuleOp op`, `float threshold` |
| `Scheduler::Task` | `core/Scheduler.hpp` | `const char* name`, `uint32_t intervalMs`, `uint32_t lastRun`, `std::function<void()> fn` |
| `HealthMonitor::TaskWatch` (privado) | `core/HealthMonitor.hpp` | `const char* name`, `uint32_t timeoutMs`, `uint32_t lastMs` |

## 10. Structs de configuración (specs)

Todos en `include/core/ConfigManager.hpp`.

| Struct | Campos (tipo · default) |
|--------|-------------------------|
| `StationConfig` | `String id` · `String name` |
| `NetworkConfig` | `String mode` (comentario: `"STA"`\|`"AP"`) · `String ssid` · `String password` · `String hostname` · `bool mdns` · `String ip` (vacío = DHCP) · `String gateway` · `String subnet` · `String dns` |
| `SystemConfig` | `String timezone` · `String ntpServer` = `"pool.ntp.org"` · `String logLevel` · `String units` = `"metric"` · `String lang` = `"es"` · `float altitude` = `0.0f` m · `float windNorthOffset` = `0.0f` ° · `uint8_t windDirectionPin` = `0` (0 = sin veleta) · `float windRpull` = `10000.0f` Ω · `float windResistors[8]` = `{33000, 8200, 1000, 2200, 3900, 16000, 120000, 64900}` (orden N, NE, E, SE, S, SO, O, NO) · `String dashboardLayout` |
| `StorageConfig` | `String backend` · `uint32_t retentionDays` · `bool sdEnabled` = `false` · `uint8_t sdCsPin` = `4` |
| `SecurityConfig` | `String apiKey` · `String serverKey` · `String username` (vacío = `"admin"`) · `String password` (vacío = sin login) · `String extraKeys` (JSON `{"nombre":"clave"}`) |
| `EnergyConfig` | `uint8_t rainPin` = `0` (0 = deshabilitado) |
| `PublishersConfig` | `String webhookUrl` · `String mqttHost` · `uint16_t mqttPort` = `1883` · `String mqttTopic` = `"sema/measurement"` · `String mqttUser` · `String mqttPass` |
| `RuleSpec` | `String name` · `String sensorId` (`""` = cualquiera) · `String channelId` · `String op` (`"gt"`\|`"lt"`\|`"ge"`\|`"le"`) · `float value` = `0.0f` |
| `CalibrationSpec` | `String sensorId` · `String channelId` · `float gain` = `1.0f` · `float offset` = `0.0f` · `bool hasRange` = `false` · `float min` = `0.0f` · `float max` = `0.0f` |
| `SensorSpec` | `String id` · `String model` · `bool enabled` = `false` · `uint8_t address` = `0` · `String rom` · `uint8_t sda` = `21` · `uint8_t scl` = `22` · `uint8_t bus` = `0` (≠0 = CS de SC18IS602B) · `uint8_t uart` = `0` (≠0 = CS de MAX14830) · `uint8_t uartPort` = `0` (0..3) · `uint8_t pin` = `0` · `uint8_t rxPin` = `0` · `uint8_t txPin` = `0` · `String channel` · `String unit` · `float scale` = `1.0f` · `float offset` = `0.0f` |
| `GpioSpec` | `String id` · `uint8_t pin` = `0` · `String mode` · `uint8_t initial` = `0` · `uint8_t expanderAddr` = `0` (≠0 = MCP23017 en esa dirección) |
| `ShiftRegisterConfig` | `String type` = `"74HC595"` · `uint8_t latchPin` = `0` (RCLK/SH-LD) · `uint8_t pinModes[8]` = `{0}` (0 = no usado, 1 = usado) |
| `Mcp23s17Config` | `uint8_t csPin` = `0` (0 = no usar) · `uint8_t pinModes[16]` = `{0}` (0 = no usado, 1 = salida, 2 = entrada) |
| `SpiExpanderConfig` | `String type` = `"MAX14830"` (`"MAX14830"` UART \| `"SC18IS602B"` I²C) · `uint8_t csPin` = `0` |
| `ModbusConfig` | `bool enabled` = `false` · `uint8_t rxPin` = `16` · `uint8_t txPin` = `17` · `uint8_t deRePin` = `0` · `uint8_t uart` = `0` · `uint8_t uartPort` = `0` · `uint32_t baud` = `9600` · `uint8_t slaveId` = `1` · `uint16_t registerAddr` = `0` · `uint16_t registerCount` = `4` |
| `CanConfig` | `bool enabled` = `false` · `uint8_t txPin` = `5` · `uint8_t rxPin` = `4` · `uint32_t speed` = `500000` (125000 \| 250000 \| 500000 \| 1000000) |
| `LoraConfig` | `bool enabled` = `false` · `uint8_t csPin` = `5` · `uint8_t rstPin` = `14` · `uint8_t dio1Pin` = `26` · `uint8_t busyPin` = `27` · `float frequency` = `915.0f` MHz · `float bandwidth` = `125.0f` kHz · `uint8_t spreading` = `7` (7..12) · `uint8_t codingRate` = `5` (5..8) · `int8_t txPower` = `14` dBm |
| `ZigbeeConfig` | `bool enabled` = `false` · `uint8_t rxPin` = `16` · `uint8_t txPin` = `17` · `uint8_t uart` = `0` · `uint8_t uartPort` = `0` · `uint32_t baud` = `115200` |
| `EthernetConfig` | `bool enabled` = `false` · `uint8_t mdcPin` = `23` · `uint8_t mdioPin` = `18` · `uint8_t phyAddr` = `1` · `int powerPin` = `-1` · `uint8_t csPin` = `5` · `int rstPin` = `-1` · `int irqPin` = `4` · `uint8_t sckPin` = `18` · `uint8_t misoPin` = `19` · `uint8_t mosiPin` = `21` |
| `Config` | `uint32_t schemaVersion` = `1` + `station`, `network`, `system`, `storage`, `security`, `energy`, `publishers`, `std::vector<SensorSpec> sensors`, `std::vector<RuleSpec> rules`, `std::vector<CalibrationSpec> calibrations`, `std::vector<GpioSpec> gpio`, `Mcp23s17Config mcp23s17`, `std::vector<ShiftRegisterConfig> shiftRegisters`, `std::vector<SpiExpanderConfig> spiExpanders`, `ModbusConfig modbus`, `CanConfig can`, `LoraConfig lora`, `ZigbeeConfig zigbee`, `EthernetConfig ethernet`, `uint8_t i2cSda` = `21`, `uint8_t i2cScl` = `22` |

## 11. Enums de cadena (validados como texto, no como enum)

Estos "enums" existen solo como cadenas en la config o en el código. Las tablas
listan los valores que el firmware realmente reconoce.

### 11.1 `network.mode`

| Valor | Efecto real (`WiFiManager::begin`) |
|-------|------------------------------------|
| `"STA"` | Con `ssid` no vacío → cliente; con `ssid` vacío cae a AP |
| `"AP"` | Access point con el `hostname` como SSID |

Cualquier otro valor **invalida la config** (`ConfigManager::validate`) y además,
si llegara al runtime, haría caer a `startAp()`.

### 11.2 `gpio[].mode`

| Valor | Constante Arduino |
|-------|-------------------|
| `"output"` | `OUTPUT` (+ `initial`) |
| `"input"` | `INPUT` (también el default implícito: cualquier cadena desconocida) |
| `"input_pullup"` | `INPUT_PULLUP` |
| `"input_pulldown"` | `INPUT_PULLDOWN` |

### 11.3 `storage.backend`

| Valor | Estado real |
|-------|-------------|
| `"littlefs"` | Aceptado por `validate()`; **ningún** código elige backend con este campo |
| `"flash"` | Aceptado por `validate()`; sin implementación asociada |
| `"sd"` | Aceptado por `validate()`; el histórico en SD se activa con `storage.sdEnabled`, no con este campo |

### 11.4 `shift_registers[].type`

| Valor | Rol |
|-------|-----|
| `"74HC595"` | Salida (default); cualquier valor distinto de `"74HC165"` se trata como salida |
| `"74HC165"` | Entrada (comparación exacta) |

Compilado solo con `SEMA_USE_SHIFT=1`, que **por defecto está en 0**.

### 11.5 `spi_expanders[].type`

| Valor | Rol |
|-------|-----|
| `"MAX14830"` | Expansor UART por SPI (4 puertos U0–U3 según el CHANGELOG) |
| `"SC18IS602B"` | Expansor I²C por SPI |

Ambos son **opciones de configuración**: no hay driver propio en el repo (solo se
declaran y se muestran en la web).

### 11.6 `sensors[].model` — 17 modelos de `SensorFactory::create()`

| Modelo | Driver | `model()` real del driver | Interfaz | `measure()` llena |
|--------|--------|---------------------------|----------|-------------------|
| `ADC` | `AdcSensor` | `ESP32-ADC` | `ADC` | 1 (canal/unidad de config) |
| `ADS1115` | `Ads1115Sensor` | `ADS1115` | `I2C` | 1 (canal `pin` 0..3) |
| `AHT20` | `Aht20Sensor` | `AHT20` | `I2C` | 2 |
| `AS3935` | `As3935Sensor` | `AS3935` | `I2C` | 1 |
| `BH1750` | `Bh1750Sensor` | `BH1750` | `I2C` | 1 |
| `BME280` | `Bme280Sensor` | `BME280` | `I2C` | 3 |
| `BMP280` | `Bmp280Sensor` | `BMP280` | `I2C` | 2 |
| `CO` | `CoSensor` | `CO` | `ADC` | 1 |
| `DS18B20` | `Ds18b20Sensor` | `DS18B20` | `1-Wire` | 1 por dispositivo |
| `PCNT` | `PcntSensor` | `PCNT` | `GPIO` | 1 |
| `PMS5003` | `Pms5003Sensor` | `PMS5003` | `UART` | 3 |
| `SCD30` | `Scd30Sensor` | `SCD30` | `I2C` | 3 |
| `SGP30` | `Sgp30Sensor` | `SGP30` | `I2C` | 2 |
| `SHT31` | `Sht31Sensor` | `SHT31` | `I2C` | 2 |
| `SHT40` | `Sht40Sensor` | `SHT40` | `I2C` | 2 |
| `SOLAR` | `SolarSensor` | `SOLAR` | `ADC` | 1 |
| `VEML6075` | `Veml6075Sensor` | `VEML6075` | `I2C` | 3 |

⚠️ El valor de config y el `model()` del driver **no siempre coinciden**: `ADC`
(config) → `ESP32-ADC` (driver). Cualquier otro `model` deja `create()` en
`nullptr` y `SemaCore::applySensors()` imprime `Sensor desconocido: <id> (modelo <model>)`.

### 11.7 `Sensor::interface()` — valores posibles

`"I2C"` · `"1-Wire"` (con guion) · `"ADC"` · `"GPIO"` (el driver PCNT) · `"UART"`.

### 11.8 Boards y features (compile-time, `include/hw/HwProfile.hpp`)

| Board (`build_flags`) | `SEMA_BOARD_ID` | `SEMA_FLASH_MB` | `SEMA_NATIVE_ETH` |
|-----------------------|-----------------|----------------:|:-----------------:|
| `BOARD_ESP32_WROOM` | `esp32-wroom-4mb` | 4 | 1 |
| `BOARD_ESP32_WROOM32U` | `esp32-wroom32u-16mb` | 16 | 1 |
| `BOARD_ESP32_S3` | `esp32-s3-8mb` | 8 | 0 |

Transceiver RS485: `SEMA_MODBUS_ISOLATED=1` → `"TD501D485H"` (aislado);
`=0` → `"SN65HVD75DR"`.

---

## Ver también

- [Referencia de API interna](Referencia-API-interna.md) · [Referencia de código](Referencia-de-codigo.md) · [Referencia de configuración](Referencia-configuracion.md) · [Arquitectura](Arquitectura.md) · [Módulos y ciclo de vida](Modulos-y-ciclo-de-vida.md) · [Energía y consumo](Energia-y-consumo.md) · [Alarmas y reglas](Alarmas-y-reglas.md) · [Compatibilidad de versiones](Compatibilidad-de-versiones.md)
